# 3. 锁屏与验证方式

## 给人看的：锁屏需要做什么

锁屏的核心就三件事：

1. **让你输密码**——显示一个键盘，收集输入
2. **验证对不对**——对了放行，错了告诉你
3. **防暴力破解**——错够次数锁定一段时间

其他的——锁屏长什么样、错误消息怎么递进、解锁之后播什么动画——都是你自己的设计，不是必须的。这一章讲必须做对的部分，以及**怎么再加一把指纹**（每天输密码真的很烦）。花活儿见 [06-extensions.md](06-extensions.md)。

---

## 给 AI 看的：实现细节

### 整体结构

```kotlin
@Composable
fun LockScreen(
    securitySetting: SecuritySetting,
    onUnlocked: () -> Unit,             // 解锁成功
    onVerifyPin: suspend (String) -> Boolean,
    onRecordFailed: suspend () -> Int,   // 返回剩余次数，0 = 锁定
    getRemainingLockoutSeconds: () -> Int,
    onUnlockStarted: () -> Unit,         // 开始过渡，主界面可以提前组合
) {
    var pin by remember { mutableStateOf("") }
    var isLockedOut by remember { mutableStateOf(false) }
    
    // 满位自动验证
    LaunchedEffect(pin) {
        if (pin.length == 6) {
            delay(200)  // 等最后一个圆点动画完成
            val correct = onVerifyPin(pin)
            if (correct) {
                onUnlockStarted()
                onUnlocked()
            } else {
                val remaining = onRecordFailed()
                if (remaining == 0) isLockedOut = true
                pin = ""
            }
        }
    }
}
```

### PIN 输入

不设确认按钮。输满位数（我们用 6 位，你可以改）自动触发验证。延迟 200ms 是为了让最后一个圆点动画完成，视觉上不突兀。

### 验证流程

验证本身在 AppLockManager 里做（SHA-256 比对），锁屏 UI 只负责调用和处理结果：

- 对了 → 调 `onUnlocked()`，状态机把 `shouldShowLockScreen` 设为 false
- 错了 → 调 `onRecordFailed()` 拿到剩余次数。次数用完就进入锁定状态

### 暴力破解锁定

错够设定次数（比如 5 次）后，进入冷却期（比如 30 秒）。冷却期内键盘不可用，显示倒计时。

```kotlin
// AppLockManager 里
suspend fun recordFailedAttempt(): Int {
    failedAttempts++
    if (failedAttempts >= securitySetting.lockMaxAttempts) {
        lockoutUntil = System.currentTimeMillis() + 
            (securitySetting.lockLockoutDurationSeconds * 1000L)
        return 0  // 0 = 已锁定
    }
    return securitySetting.lockMaxAttempts - failedAttempts
}

fun getRemainingLockoutSeconds(): Int {
    val now = System.currentTimeMillis()
    if (now >= lockoutUntil) return 0
    return ((lockoutUntil - now) / 1000).toInt()
}
```

锁屏 UI 里用一个 `LaunchedEffect` 每秒检查一次，倒计时归零后解除锁定、重置失败次数。

### PIN 哈希存储

PIN 不存明文，存 SHA-256 哈希。验证时把输入也哈希一遍再比对。

```kotlin
companion object {
    fun hashPin(pin: String): String {
        val bytes = pin.toByteArray(Charsets.UTF_8)
        val digest = java.security.MessageDigest.getInstance("SHA-256")
        return digest.digest(bytes).joinToString("") { "%02x".format(it) }
    }
}
```

> 本地应用锁用 SHA-256 够了——攻击者要拿到哈希得先 root 手机。如果你的场景安全要求更高，可以换 bcrypt/scrypt 或者用 Android Keystore。

### vol. 天数计算（如果你的锁屏也想要这种效果）

我们的锁屏上有一行 "vol. 361 · september"，天数从一个锚点日期算起，每天 +1。这不是必须的，只是我们的设计选择。如果你也想要：

```kotlin
private fun calculateVolNumber(anchorDate: String): String {
    val format = SimpleDateFormat("yyyy-MM-dd", Locale.US)
    val anchor = format.parse(anchorDate) ?: Date()
    val days = ((Date().time - anchor.time) / (1000 * 60 * 60 * 24)).toInt()
    val month = SimpleDateFormat("MMMM", Locale.ENGLISH).format(Date()).lowercase()
    return "vol. $days · $month"
}
```

---

# 3.1 再加一把指纹（生物识别）

## 给人看的：为什么要做、做成什么样

6 位密码每天输十几遍，很快就烦了。指纹的价值不是"更安全"，是**不用输**。

**做法是把指纹挂在锁屏上已有的某个元素上**——比如锁屏顶部那个图标，点一下就弹系统的指纹框。这样：

- 锁屏的样子**一点不用改**，不用多一个"指纹"按钮破坏画面
- 别人拿起手机看到的就是一把普通的密码锁，不会觉得"这锁还支持指纹"
- 想输密码随时还能输，两条路并存，指纹只是快捷方式

### 三件你必须先知道的事

**① 指纹界面不能用你自己画的。** 系统层面不允许第三方 App 自绘指纹界面——指纹采集发生在受信任环境里，必须由一个"系统提供的框"接收结果。这是**防钓鱼**设计：如果 App 能自己画指纹框，就能画个假的骗指纹。网上那些看起来像自绘的界面，其实都是系统框套了层皮肤（或者根本是手机厂的系统 App）。

所以你能控制的只有：**什么时候弹、弹出的框上写什么字**。想要"图标本身就是扫描器"，做不到，别在这上面花时间。

**② 弹框要求你的 Activity 是 `FragmentActivity`。** 这是最坑的一条，因为**不满足时它不报错、不崩溃，只是什么都不发生**——你点了图标毫无反应，查半天不知道哪里的问题。详见 [05-pitfalls.md](05-pitfalls.md) #16。

**③ 弹框的"取消"按钮是条件必填的。** 验证方式里不包含"设备密码"时，必须给一个取消按钮文字，否则 `build()` 直接抛异常崩溃。详见 [05-pitfalls.md](05-pitfalls.md) #17。

---

## 给 AI 看的：实现细节

### 加依赖

```kotlin
implementation("androidx.biometric:biometric:1.2.0-alpha05")
```

> 版本号按你构建时能拉到的最新版填。1.4.x 目前只有 alpha，正式号是 1.2.0-alpha05（2026-09 时点）。

### 先判断能不能用

设备可能没传感器，或者用户根本没录指纹。**两种情况都要当成"不能用"**，不能让锁屏卡住：

```kotlin
val biometricAvailable = remember {
    val bm = BiometricManager.from(context)
    val strong = bm.canAuthenticate(BiometricManager.Authenticators.BIOMETRIC_STRONG)
    val weak = bm.canAuthenticate(BiometricManager.Authenticators.BIOMETRIC_WEAK)
    // 有些设备 STRONG 返回非 0 但 WEAK 是 0，两个都试
    strong == BiometricManager.BIOMETRIC_SUCCESS || weak == BiometricManager.BIOMETRIC_SUCCESS
}
```

> `BIOMETRIC_SUCCESS` 的值就是 **0**。别写成 `!= BIOMETRIC_SUCCESS` 之类的反逻辑，这个常量名很容易看反。

### 从 Context 里找到 FragmentActivity

`LocalContext.current` 拿到的通常就是 Activity，但**不能直接强转**——得往上找一层，而且必须确认它是 `FragmentActivity`：

```kotlin
val biometricPrompt = remember(context) {
    var ctx: android.content.Context? = context
    var activity: FragmentActivity? = null
    while (ctx is android.content.ContextWrapper) {
        if (ctx is FragmentActivity) { activity = ctx; break }
        ctx = ctx.baseContext
    }
    activity?.let { BiometricPrompt(it, ContextCompat.getMainExecutor(context), callback) }
}
```

**拿不到就返回 null，图标点了什么也不做**，密码照常能用——fail-safe，别让整把锁因为指纹不可用而废掉。

### 用哪个 Activity

**你的 `Activity` 基类必须是 `FragmentActivity`**，不能是 `ComponentActivity`：

```kotlin
class RouteActivity : FragmentActivity()   // 不是 ComponentActivity()
```

`FragmentActivity` 本身就是 `ComponentActivity` 的子类，`setContent`、生命周期、`enableEdgeToEdge` 这些用法一个都不受影响，只是多了一层 Fragment 支持。这个改动是全局的，改完别的功能理论上不受影响，但我们还是完整回归了一遍。

### 弹框

```kotlin
val promptInfo = BiometricPrompt.PromptInfo.Builder()
    .setTitle("指纹解锁")
    .setAllowedAuthenticators(BiometricManager.Authenticators.BIOMETRIC_WEAK)
    // 允许的方式里没有「设备密码」时，这行是必填，少了就崩
    .setNegativeButtonText("取消")
    .build()

biometricPrompt.authenticate(promptInfo)
```

把这个调用挂在锁屏上已有的某个元素上（我们用顶部图标）：

```kotlin
Icon(
    painter = painterResource(lockIcon),
    modifier = Modifier
        .size(120.dp)
        .clip(CircleShape)              // 顺手把点击区域圈成圆形，别是方的
        .clickable(onClick = triggerBiometric)
        .padding(16.dp),
)
```

### 失败怎么处理

回调里有**两个**不同的失败信号，别混：

```kotlin
override fun onAuthenticationFailed() {
    // 手指按上去了，但没认出来（按歪了、手湿了）
    // 这个可以计数
}

override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
    // 用户点了取消、或者系统层面出错
    // 这个不该计数——用户取消就是想输密码，别当成"试错"
}
```

我们的策略：**一次锁屏内错两次就关掉指纹，只能输密码**，免得有人拿手指一直蹭。错误话用跟密码错误**不同的措辞**（密码错是"你输错了"，指纹错是"这不是你"），文案你可以自己写。

```kotlin
override fun onAuthenticationFailed() {
    biometricFailCount++
    if (biometricFailCount >= 2) {
        biometricDisabled = true
        errorMessage = "别试了，输密码。"
    } else {
        errorMessage = "再按一次。"
    }
    showError = true
}
```

`biometricFailCount` 和 `biometricDisabled` 用 `remember` 存——**锁屏重新组合时会自动归零，也就是下次进锁屏又能用指纹了**，这正是我们要的行为。

### 加个开关

别默认开。给用户一个"指纹解锁"的开关（我们在应用锁设置页里，启用应用锁之后才显示这一行）。

⚠️ 如果这个设置是**序列化存盘的**（DataStore / SharedPreferences），**加字段是安全的，改老字段的类型不是**——老数据会解析失败。给新字段一个安全默认值（`false`）就行，不用写迁移。

