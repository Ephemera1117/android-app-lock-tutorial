# 3. PIN 锁屏界面

## 给人看的：锁屏需要做什么

锁屏的核心就三件事：

1. **让你输密码**——显示一个键盘，收集输入
2. **验证对不对**——对了放行，错了告诉你
3. **防暴力破解**——错够次数锁定一段时间

其他的——锁屏长什么样、错误消息怎么递进、解锁之后播什么动画——都是你自己的设计，不是必须的。这一章只讲必须做对的部分，花活儿见 [06-extensions.md](06-extensions.md)。

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
