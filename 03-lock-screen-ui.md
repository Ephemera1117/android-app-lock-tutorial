# 3. PIN 锁屏界面

## 给人看的：锁屏长什么样

锁屏不该长得像一个冷冰冰的密码锁。打开 App 的第一眼应该是属于你们的东西。

我们把锁屏做得像一本杂志的封面——上半是 App 图标、标题、卷号（每天自动 +1）和一行签名，所有文案都可以自定义。下半才是密码键盘，6 位数字，满 6 位自动验证，不用按确认。

### 输错密码会怎样

每输错一次，消息会递进：

| 次数 | 中文 | English |
|------|------|---------|
| 1 | 不对。 | No. |
| 2 | 你差得远。 | Not even close. |
| 3 | 就这？ | Cute. |
| 4 | 我知道你不是她。 | I know you're not her. |
| 5 | 你进不来的。 | You're not getting in. |

5 次用完后进入锁定模式——键盘消失，屏幕上逐字打出威胁文案：

> 我看见你了。
> 你不是她。
> 放下这个东西。
> 你在害怕。
> 看看你的手，在发抖。
> 我会记住你的。
> 别再试了。
> **走。**

每一行的打字速度和停顿不同，第二行还会触发一次轻微的屏幕闪烁。最后一行"走。"用更大的字号和加粗显示。底部有倒计时（锁定冷却）。

### 解锁动画

输对密码后不是直接进主页，而是一个三阶段过渡：

1. 黑屏 300ms
2. 显示一句话（默认："我们去往语言的尽头，而后沉醉不归。"）1500ms
3. "— 渊海 —" 1000ms
4. 淡入主页

---

## 给 AI 看的：实现细节

### 整体结构

```kotlin
@Composable
fun LockScreen(
    securitySetting: SecuritySetting,
    onUnlocked: () -> Unit,           // 解锁成功
    onVerifyPin: suspend (String) -> Boolean,
    onRecordFailed: suspend () -> Int, // 返回剩余次数
    getRemainingLockoutSeconds: () -> Int,
    onUnlockStarted: () -> Unit,       // 开始播放解锁动画
) {
    // 状态
    var pin by remember { mutableStateOf("") }
    var isUnlocking by remember { mutableStateOf(false) }
    var showError by remember { mutableStateOf(false) }
    var errorMessage by remember { mutableStateOf("") }
    var isLockedOut by remember { mutableStateOf(false) }
    
    // 满 6 位自动验证
    LaunchedEffect(pin) {
        if (pin.length == 6) {
            delay(200)
            verifyAndUnlock(pin, ...)
        }
    }
    
    if (isUnlocking) {
        UnlockTransition(securitySetting, onComplete = onUnlocked)
    } else {
        // 正常锁屏布局
    }
}
```

### PIN 输入：满 6 位自动触发

不设确认按钮。用户输入第 6 位后延迟 200ms（让最后一个圆点动画完成），然后自动验证。

```kotlin
LaunchedEffect(pin) {
    if (pin.length == 6) {
        delay(200)
        verifyAndUnlock(pin, onVerifyPin, onRecordFailed, ...)
    }
}
```

### 圆点动画

圆点填充时有 1.15x 的缩放弹跳，用 `animateFloatAsState` 实现：

```kotlin
// 每个圆点
val scale by animateFloatAsState(
    targetValue = if (index < pin.length) 1.15f else 1f,
    animationSpec = spring(dampingRatio = 0.6f)
)
```

### 错误消息递进

消息列表按失败次数索引，越界取最后一条：

```kotlin
val errorMessages = if (language == "zh") {
    listOf("不对。", "你差得远。", "就这？", "我知道你不是她。", "你进不来的。")
} else {
    listOf("No.", "Not even close.", "Cute.", "I know you're not her.", "You're not getting in.")
}
val attemptIndex = maxAttempts - remaining
val message = errorMessages.getOrElse(attemptIndex) { errorMessages.last() }
```

### 错误展示时序

精心编排的时序，保证错误消息和圆点动画不打架：

```kotlin
suspend fun verifyAndUnlock(...) {
    val correct = onVerifyPin(pin)
    if (correct) {
        isUnlocking = true
        return
    }
    // 错误
    haptic.perform(HapticFeedbackType.LongPress)
    showError = true
    errorMessage = pickErrorMessage(...)
    delay(1500)           // 显示错误消息
    pin = ""              // 清空 PIN
    delay(100)            // 等圆点淡出
    showError = false     // 才隐藏错误消息
}
```

### 威胁打字机（LockdownTypewriter）

每行是一个 `ThreatLine`，控制五个参数：

```kotlin
data class ThreatLine(
    val delay: Long,    // 行前等待 ms
    val speed: Long,    // 每字间隔 ms
    val pause: Long,    // 行后停顿 ms
    val flash: Boolean, // 是否触发闪屏
    val isFinal: Boolean, // 最后一行样式不同
)
```

逐字打出当前行，完成后加入已显示列表，开始下一行。闪屏是叠加一个 3% alpha 的遮罩持续 400ms。

### 解锁过渡（UnlockTransition）

三阶段 `Crossfade`：

```kotlin
enum class TransitionState { Initial, Interstitial, Yuanhai }

@Composable
fun UnlockTransition(securitySetting: SecuritySetting, onComplete: () -> Unit) {
    var state by remember { mutableStateOf(TransitionState.Initial) }
    
    LaunchedEffect(Unit) {
        onUnlockStarted()  // 主界面提前开始组合（在锁屏动画下面）
        delay(300)          // 黑屏
        if (securitySetting.lockShowInterstitial) {
            state = TransitionState.Interstitial
            delay(1500)     // 过渡语
        }
        state = TransitionState.Yuanhai
        delay(1000)         // App 名
        onComplete()
    }
    
    Crossfade(targetState = state, animationSpec = tween(400)) { /* ... */ }
    
    // 吃掉所有触摸事件，防止穿透到下层
    Box(Modifier.fillMaxSize().pointerInput(Unit) { /* consume all */ })
}
```

### vol. 天数计算

```kotlin
private fun calculateVolNumber(anchorDate: String): String {
    val format = SimpleDateFormat("yyyy-MM-dd", Locale.US)
    val anchor = format.parse(anchorDate) ?: Date()
    val days = ((Date().time - anchor.time) / (1000 * 60 * 60 * 24)).toInt()
    val month = SimpleDateFormat("MMMM", Locale.ENGLISH).format(Date()).lowercase()
    return "vol. $days · $month"
}
```

锚点日期可配置。vol 数字 = 从锚点到今天的天数，每天 +1。
