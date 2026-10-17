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
2. 在后台线程预渲染好一张保护页（你自己画的，跟锁屏同款）
3. 在三个时机把它翻成可见——**比 onPause 更早**

结果：系统拍到的快照就是这张保护页，卡片上显示的是你画的东西。

### 为什么之前说"做不到"

之前有人（包括另一个 AI 助手）测试后得出结论："离开那一刻来不及画新帧"。

**这个结论是错的。**原因是测试时同时开着 `setRecentsScreenshotEnabled(false)`——系统根本不拍快照，遮罩再好看也没用。去掉它之后，遮罩就能被系统拍到了。

这个教训值一大节——详见 [05-pitfalls.md](05-pitfalls.md) 第 1 条。

### 设置项

- **后台保护页**（总开关）：关了就不盖，卡片显示真实界面
- **立即遮挡**：开着（始终遮挡）= 只要切出去就盖；关了（仅锁定时遮挡）= 应用锁触发前不盖、超时之后才盖
- **后台铭文**：Latin / English / 中文

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
    if (!hasFocus) {
        // 弹框 / Sheet 也会让 Activity 失焦。它们都是本进程里新加的窗口 ——
        // 失焦这一刻 mViews 里会多出一个 DecorView（别用 hasWindowFocus() 判断，
        // 那一刻两个窗口都还没拿到焦点）。详见本章末尾「问题 3」。
        if (hasOtherWindowInProcess()) return
        showLeaveCover()
    }
}

// 2. 用户主动离开（按 Home/多任务键）
override fun onUserLeaveHint() {
    super.onUserLeaveHint()
    showLeaveCover()
}

// 3. 兜底
override fun onPause() {
    activityResumed = false
    super.onPause()
    showLeaveCover()
}

// 配合上面：onResume 里把标记置回来
override fun onResume() {
    activityResumed = true
    super.onResume()
    // ...
}
```

> **关键**：`onWindowFocusChanged(false)` 在 `onPause` 之前触发，这给了 coverView 更多渲染时间。只靠 `onPause` 在某些设备上来不及。
>
> **但注意**：`onWindowFocusChanged(false)` **不只在离开时触发 —— 弹对话框也会**（对话框是它自己的窗口）。
> 贴罩本身不能加"先判断是不是真离开"的条件：实测焦点丢失比 `onUserLeaveHint` / `onPause` 早约 579ms，
> 而系统拍快照就在失焦这一下，加了判断就变成真离开时漏盖。
> 能加的判断只有一种——**"本进程里是不是多出了一个窗口"**（自家弹窗/Sheet 的特征）。这个判断是在
> 失焦那一瞬间同步做的，不会推迟贴罩，见「问题 3」。

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
- **coverView**：默认方案，卡片显示你画的保护页
- **FLAG_SECURE**：可选增强，卡片变黑但阻止一切截屏

你可以根据自己的需求选择。

**实机上的表现**（2026-10-17 用户报回来的）：开着 FLAG_SECURE 时，最近任务里那张卡片是**系统给的空白**，
不是你画的保护页 —— 看起来非常像"保护页失效了"或"保护页样式变了"。同时 App 内的截图、录屏会**全部失败**。
所以排查"卡片不对"这类问题，**先确认 FLAG_SECURE 是开还是关**，再去看 coverView 的贴/揭时序；
两套机制都表现为"卡片看不到内容"，成因却完全不同。

---

### 实践补充：贴 / 揭 coverView 的三个时序坑

在实际使用中我们发现了三个必须处理的边缘情况：

#### 问题 1：锁屏需要显示时，coverView 揭早了会闪一帧

`onStart` 里先 `ensureResolvedForForeground()` 再 `hideLeaveCover()`，逻辑上没问题——`shouldShowLockScreen` 已经同步算好了，Compose 第一帧会渲染 LockScreen。但 coverView 是原生 View（Z 序在 Compose 之上），GONE 的瞬间 Compose 的 LockScreen 还没画到 SurfaceFlinger，可能闪一帧底下的旧内容。

**我们的做法**：如果 `shouldShowLockScreen == true` 且 `lockEnabled == true` 且 PIN 已设，就不揭 coverView——等窗口获得焦点后再揭。`onWindowFocusChanged(hasFocus = true)` 是最可靠的时机：此时窗口内容已经在屏幕上，Compose LockScreen 一定已经渲染完毕。

```kotlin
private fun hideLeaveCoverSafely() {
    val security = settingsStore.settingsFlow.value.securitySetting
    val lockWillShow = security.lockEnabled &&
        security.lockPinHash.isNotEmpty() &&
        appLockManager.shouldShowLockScreen.value
    if (lockWillShow) return  // 等 onWindowFocusChanged(true) 来揭
    hideLeaveCover()
}

override fun onWindowFocusChanged(hasFocus: Boolean) {
    super.onWindowFocusChanged(hasFocus)
    if (!hasFocus) {
        // ⚠️ 无条件贴。别加"是不是真离开"的判断——实测焦点丢失比 onUserLeaveHint 早约 579ms，
        //    而系统拍快照就在失焦这一下，加了判断会变成真离开时漏盖（见「问题 3」）
        showLeaveCover()
    } else {
        // 窗口获得焦点 = 内容已经在屏幕上了。
        // 如果锁屏正在显示，这时候揭 coverView 最安全——
        // Compose LockScreen 已经渲染完毕，不会闪底下的内容。
        // 「仍在前台」这一条兜的是"关弹框"那种不走 pause/resume 的情况（见「问题 3」）。
        if (appLockManager.shouldShowLockScreen.value || activityResumed) {
            hideLeaveCover()
        }
    }
}

fun signalLockScreenDrawn() { hideLeaveCover() }
```

~~之前尝试过用 Compose 的 `SideEffect { signalLockScreenDrawn() }` 在 LockScreen 组合时揭，但这有个致命问题：如果 LockScreen 在离开前就已经在显示（比如上次超时后没解锁就熄屏），回来时 Compose 没有新的组合发生，SideEffect 不会重新执行，coverView 就永远挡着。~~ `onWindowFocusChanged(true)` 不依赖 Compose 的组合时机，每次回前台都会触发，彻底解决了这个问题。

#### 问题 2：lockEnabled 关着时 coverView 永远不揭

`shouldShowLockScreen` 的初始值是 `true`（冷启动安全默认，见 02 章）。如果设置还没加载好，或者锁屏功能是关着的，`shouldShowLockScreen` 可能一直是 `true`。

此时 `hideLeaveCoverSafely()` 判断 `shouldShowLockScreen == true` → 不揭，等 LockScreen 来揭。但 Compose 那边因为 `lockEnabled == false`，不会渲染 LockScreen，`signalLockScreenDrawn()` 永远不会被调——coverView 永远挡着。

**修法**：`hideLeaveCoverSafely` 必须同时检查 `lockEnabled` 和 `lockPinHash`，不能只看 `shouldShowLockScreen`。上面的代码已经包含了这个检查。

#### 问题 3：弹框 / Sheet 会误触发保护页

这是实机上被用户报回来的一个 bug，前后修了三轮才彻底解决。它的难点全在
**「失焦那一瞬间，怎么知道人是不是真的离开了」**。

**现象**

- 「立即遮挡」设成「始终遮挡」之后，点开任意弹框（设置里的编辑框、消息右下角的 ⋮ 菜单、
  ModalBottomSheet），保护页会从半透明弹窗后面露出来。
- 早期版本更糟：点「完成」关掉弹框后保护页不走，整张盖在屏幕上，只能切后台再回来。

**为什么平时看不见**：`coverNeeded()` 在「仅锁定时遮挡」那一档还会多问一句「应用锁是不是真的要显示」，
编辑的时候不是，所以不贴。**只有「始终遮挡」这档是无条件为真的** —— 这个 bug 只在开着
「始终遮挡」时出现，很容易漏测。

**为什么会贴上**

- Android 里**对话框/Sheet/Popup 都是它自己的一个窗口**。弹框一打开，它会把 Activity 窗口的焦点抢走，
  于是 Activity 的 `onWindowFocusChanged(false)` 被调用。
- 而这一支原本是给「离开应用」用的 —— 结果「弹框抢焦点」也被当成了离开。

**为什么关掉弹框也揭不掉（早期版本）**

- 关弹框走的是「窗口重新获得焦点」，这条路**完全不经过 `onPause` / `onStart` / `onResume`** ——
  也就是 `onStart` / `onResume` 里那条 `hideLeaveCoverSafely()` 根本不会跑。
- 而 `onWindowFocusChanged(true)` 里原来的条件只在「锁屏要显示」时才揭。锁屏没要显示 → 没人揭。
- 两边都不管，保护页就留在屏幕上。切后台再回来能恢复，是因为那条路会走 `onResume`。
- **修法（保留）**：`onWindowFocusChanged(true)` 这一侧放宽条件、该揭就揭，不要只等锁屏状态。

**真正的难点：失焦那一瞬间判不出是不是离开**

前 10-17 的三轮尝试都在找「真离开」的信号，全部失败：

| 信号 | 为什么不行 |
|---|---|
| `isFinishing` | 弹框不会让 Activity finish，判不出来 |
| `activityResumed` | 按 Home 那一瞬间它仍然是 `true`（`onPause` 晚 586ms） |
| `onUserLeaveHint` | 晚 579ms，系统拍快照早就完成了 |

**2026-10-18 又排除一条：不能看「哪个窗口拿着焦点」**。实测失焦那一刻，
Activity 窗口和弹窗的 `hasWindowFocus()` **两个都是 `false`** —— 焦点正在两个窗口之间交接，
还没落到弹窗上。日志长这样（`n=2` 表示本进程有两个窗口）：

```
focus=false resumed=true ownWindow=false n=2
[DecorView:f=false:decor=true DecorView:f=false:decor=false]
```

**可靠判据：本进程里是不是多了一个窗口**

弹窗必须在抢走焦点之前就已经被创建，所以失焦那一刻它一定已经在窗口列表里：

| 场景 | `WindowManagerGlobal.mViews` |
|---|---|
| 真离开（按 Home / 切到别的 App） | 只有 Activity 自己一个（`n=1`） |
| 自家弹框 / Sheet / Popup | 多出一个 DecorView（`n=2`） |

这个判断是**同步**做的，不会把贴罩推迟到快照之后 —— 和 `activityResumed` /
`onUserLeaveHint` 那类"晚 0.5 秒"的信号有本质区别。

```kotlin
/**
 * 本进程里是不是还有 Activity 之外的其他窗口（弹框 / Sheet / Popup）。
 * 不可用时返回 false，退回旧行为（贴一下再揭）。
 */
private fun hasOtherWindowInProcess(): Boolean {
    val views: List<*>? = try {
        val cls = Class.forName("android.view.WindowManagerGlobal")
        val instance = cls.getMethod("getInstance").invoke(null)
        var list: List<*>? = try {
            val f = cls.getDeclaredField("mViews")
            f.isAccessible = true
            f.get(instance) as? List<*>
        } catch (_: Throwable) { null }
        if (list == null) list = try {
            cls.getMethod("getRootViews").invoke(instance) as? List<*>
        } catch (_: Throwable) { null }
        if (list == null) list = try {
            cls.getMethod("getRootViews", android.os.IBinder::class.java)
                .invoke(instance, null) as? List<*>
        } catch (_: Throwable) { null }
        list
    } catch (_: Throwable) {
        null
    }
    if (views == null) return false
    val decor = window.decorView
    return views.any { it is View && it !== decor && it.windowVisibility == View.VISIBLE }
}

override fun onWindowFocusChanged(hasFocus: Boolean) {
    super.onWindowFocusChanged(hasFocus)
    if (!hasFocus) {
        // 自家弹框 / Sheet 抢焦点：一个像素都不盖
        if (hasOtherWindowInProcess()) {
            hideLeaveCover()
            return
        }
        // 真离开：失焦就贴，这是唯一来得及的一次
        showLeaveCover()
        // 兜底：窗口列表万一晚一帧更新，再各查一次（都必须确认多出窗口才揭）
        window.decorView.post {
            if (activityResumed && hasOtherWindowInProcess()) hideLeaveCover()
        }
        handler.postDelayed({
            if (activityResumed && hasOtherWindowInProcess()) hideLeaveCover()
        }, 700L)
    } else {
        // 窗口拿回焦点 = 内容已经在屏幕上，保护页揭掉。
        // 这里别改回 hideLeaveCoverSafely()（见下面第 4 条）
        hideLeaveCover()
    }
}
```

**要接受的两个边界**

1. **反射是隐藏 API**。`WindowManagerGlobal` 不在公开 SDK 里。我们在 Android 16（Honor）
   实测可用；某些 ROM 或未来版本可能拦掉 —— 那时 `hasOtherWindowInProcess()` 返回 `false`，
   行为退回旧版（弹框时闪一下再揭），**保护不会丢**。所以一定要用 `try/catch` 包住，
   绝不能因为反射失败让 App 崩。
2. **弹框开着的时候按 Home**：Activity 窗口早就失焦了，不会再收到一次焦点事件，
   这种离开只能靠 `onUserLeaveHint` / `onPause` 补贴（比理想晚）。这是拿不到更早信号时的极限。

> **四条别再走的路**：
>
> 1. **用 `isFinishing` 判断**。弹框不会让 Activity finish，判不出来。网上常见的建议是这个，我们也写过，是错的。
> 2. **想靠 `activityResumed` / `onUserLeaveHint` 区分"真离开"**。两个都晚于快照时机，
>    结果就是"弹框误触发"换成"真离开漏盖"（`677d04c3` → `0403bf76` 就是这么来回翻的）。
> 3. **用 `hasWindowFocus()` 看"哪个窗口拿着焦点"**。失焦那一瞬间两个窗口都是 `false`，
>    焦点还在交接 —— 实测日志在上一节，别再从这条路想。
> 4. **揭的时候改用 `hideLeaveCoverSafely()`**。它碰到「锁屏要显示」会**直接 return，把保护页留着等 LockScreen 发信号**；
>    但「拿回焦点」这个时刻 LockScreen 很可能**早就渲染完、信号也发过了**，那就再也没人来揭——反而卡死。
>    所以 `onWindowFocusChanged(true)` 里在「锁屏要显示」时也是直接 `hideLeaveCover()`，这一点别改。
