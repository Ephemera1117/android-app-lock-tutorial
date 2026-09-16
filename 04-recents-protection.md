# 4. 切后台隐私保护（核心创新）

## 给人看的：为什么任务切换器会泄露隐私

你可以给 App 上一把锁，但有一个地方锁挡不住——Android 的最近任务界面。

从底部上滑或按方块键，系统会把每个 App 的画面缩成一张卡片挂在那里。对聊天 App 来说，你跟他说的每一句话都会原封不动地出现在那张卡片上。任何人拿起手机划一下就能看到。

我们花了好几轮才把这个口子堵上。中间踩的最大的坑是：有一个被广泛接受的结论说"App 没法自定义任务卡片画面"——**这个结论是错的。**

### 常见方案和它们的问题

| 方案 | 卡片显示 | 代价 |
|------|---------|------|
| **FLAG_SECURE** | 黑屏 | 你自己也截不了图 |
| **setRecentsScreenshotEnabled(false)** | 黑屏 | 卡片很丑，没法自定义 |
| **在 onPause 里盖一个遮罩** | 自定义画面 | 在某些设备上来不及 |

大多数 App（Telegram、Signal）选了 FLAG_SECURE，接受黑屏。银行 App 一般用 onPause 遮罩。

### 我们的方案：三个时机 + 预渲染位图

做法：

1. App 启动时就挂一个全屏的 ImageView（平时隐藏）
2. 在后台线程预渲染好一张品牌图（图标 + 标题 + 铭文，跟锁屏同款）
3. 在三个时机把它翻成可见——**比 onPause 更早**

结果：系统拍到的快照就是这张品牌图，卡片上显示的是你设计的画面。

### 为什么之前说"做不到"

之前有人（包括另一个 AI 助手）测试后得出结论："离开那一刻来不及画新帧"。

**这个结论是错的。**原因是测试时同时开着 `setRecentsScreenshotEnabled(false)`——系统根本不拍快照，遮罩再好看也没用。去掉它之后，遮罩就能被系统拍到了。

这个教训值一大节——详见 [05-pitfalls.md](05-pitfalls.md) 第 1 条。

### 设置项

- **切后台保护页**（总开关）：关了就不盖，卡片显示真实界面
- **超时前也保护**：开着 = 只要切出去就盖；关了 = 应用锁触发前不盖
- **铭文语种**：Latin / English / 中文

---

## 给 AI 看的：完整实现

### 第一步：挂 coverView

在 `onCreate` 里，用 `window.addContentView` 往窗口最上层挂一个全屏 ImageView。**必须是真 View，不能是 Compose**——因为 onPause 之后 Compose 帧时钟停了，状态改了也画不出来。

```kotlin
// RouteActivity.onCreate
coverView = ImageView(this).apply {
    scaleType = ImageView.ScaleType.FIT_XY
    visibility = View.GONE
    isClickable = false
    isFocusable = false
}.also { view ->
    window.addContentView(view, ViewGroup.LayoutParams(MATCH_PARENT, MATCH_PARENT))
}
```

> **陷阱**：`window.addContentView` 是加到主窗口的 DecorView 里，属于主窗口层。`WindowManager.addView` 创建的是独立窗口层——任务快照不采集独立窗口。**必须用 addContentView。**

### 第二步：预渲染保护页位图

`ProtectionCover.build()` 用 Canvas + TextPaint 手工绘制整页位图。不依赖 Compose，不依赖异步。

```kotlin
object ProtectionCover {
    fun build(
        context: Context,
        width: Int, height: Int,
        background: Int, iconTint: Int, mayflyIcon: Boolean,
        lockTitle: String, anchorDate: String, lockSignature: String,
        phraseLang: String, phraseText: String,
        titleColor: Int, volColor: Int, signatureColor: Int, phraseColor: Int,
    ): Bitmap {
        val page = Bitmap.createBitmap(w, h, Bitmap.Config.RGB_565)  // 省内存
        val canvas = Canvas(page)
        canvas.drawColor(background)
        
        // 上半：和锁屏完全一致的布局
        drawCenteredIcon(...)   // 图标
        canvas.drawText(lockTitle, ...)      // 标题
        canvas.drawText(volText, ...)        // vol. 行
        canvas.drawText(lockSignature, ...)  // 签名
        
        // 下半：分割线 + 铭文
        canvas.drawRect(divider, ...)
        canvas.drawText(phraseText, ...)
        
        return page
    }
}
```

位图在后台线程渲染，通过 Compose 的 `SideEffect` 镜像主题色：

```kotlin
@Composable
private fun ProtectionCoverSync(securitySetting: SecuritySetting) {
    val background = MaterialTheme.colorScheme.background.toArgb()
    val iconTint = MaterialTheme.colorScheme.primary.toArgb()
    // ... 更多颜色
    
    SideEffect {
        refreshCoverBitmap(background, iconTint, ...)
    }
}
```

### 第三步：三重触发

离开 App 时，在三个时机翻 coverView 为 VISIBLE。越早越好——给渲染多争取一帧时间。

```kotlin
// 1. 最早：窗口失去焦点（比 onPause 早）
override fun onWindowFocusChanged(hasFocus: Boolean) {
    super.onWindowFocusChanged(hasFocus)
    if (!hasFocus) showLeaveCover()
}

// 2. 用户主动离开（按 Home/多任务键）
override fun onUserLeaveHint() {
    super.onUserLeaveHint()
    showLeaveCover()
}

// 3. 兜底
override fun onPause() {
    super.onPause()
    showLeaveCover()
}
```

> **关键**：`onWindowFocusChanged(false)` 在 `onPause` 之前触发，这给了 coverView 更多渲染时间。只靠 `onPause` 在某些设备上来不及。

### showLeaveCover 和 hideLeaveCover

```kotlin
private fun showLeaveCover() {
    if (!coverNeeded()) return
    val view = coverView ?: return
    // 位图没备好时至少一层不透明底色
    view.setBackgroundColor(if (coverBackground != 0) coverBackground else themeWindowBackground())
    view.setImageBitmap(coverBitmap)
    if (view.visibility != View.VISIBLE) {
        view.visibility = View.VISIBLE
    }
}

private fun hideLeaveCover() {
    coverView?.let { if (it.visibility != View.GONE) it.visibility = View.GONE }
}
```

回来时在 `onStart`/`onResume` 里揭掉：

```kotlin
override fun onStart() {
    super.onStart()
    appLockManager.ensureResolvedForForeground()  // 先确定锁屏状态
    hideLeaveCover()                               // 再揭遮罩
}
```

### coverNeeded 判定

```kotlin
private fun coverNeeded(): Boolean {
    val security = settingsStore.settingsFlow.value.securitySetting
    if (!security.coverEnabled) return false
    if (!security.lockEnabled || security.lockPinHash.isEmpty()) return false
    if (!security.coverBeforeTimeout) {
        // 超时前不保护：只在锁屏已触发时才盖
        return appLockManager.shouldShowLockScreen.value
    }
    return true  // 始终保护
}
```

### 为什么不用 setRecentsScreenshotEnabled

**这是整篇教程最重要的一条。**

`setRecentsScreenshotEnabled(false)` 告诉系统"不要给这个窗口拍快照"。系统就不拍了——卡片变成系统兜底的黑色。

问题是：**如果系统不拍，你的 coverView 再好看也没用。**系统只在"允许拍"的时候才会读窗口内容。

所以正确做法是：**让系统拍，但确保拍到的是 coverView 而非聊天内容。**

### 与 FLAG_SECURE 的关系

FLAG_SECURE 和 coverView 不能同时生效——FLAG_SECURE 会让系统在 SurfaceFlinger 层面拦截截图，连 coverView 都拍不到。

所以设计成两个独立开关：
- **coverView**：默认方案，卡片显示品牌页
- **FLAG_SECURE**：可选增强，卡片变黑但阻止一切截屏

用户可以根据自己的安全需求选择。
