# 6. 可以拓展什么

> 上面五章讲的是"必须做对的事"。这一章讲的是"做了会更好的事"——都是我们自己加的花活儿，不影响核心功能，但能让体验好很多。你可以挑感兴趣的做，也可以全部跳过。

---

## 错误消息递进

不是每次都弹同一句"密码错误"，而是随着错误次数递进，语气越来越不友好。我们做了中英双语各 5 条，按失败次数索引取：

```kotlin
val errorMessages = listOf("不对。", "你差得远。", "就这？", "我知道你不是她。", "你进不来的。")
val attemptIndex = maxAttempts - remaining
val message = errorMessages.getOrElse(attemptIndex) { errorMessages.last() }
```

你完全可以写自己的文案，或者做成可配置的。关键是这个递进机制——让入侵者感受到"这个锁不是摆设"。

---

## 锁定期间的威胁文案（打字机效果）

错够次数进入冷却后，键盘消失，屏幕上逐字打出一系列文案。每一行的打字速度、打字前的等待、打完后的停顿都可以单独设。我们用一个数据类控制：

```kotlin
data class ThreatLine(
    val delay: Long,      // 这行开始前等多久 (ms)
    val speed: Long,      // 每个字之间间隔多久 (ms)
    val pause: Long,      // 这行打完之后停多久 (ms)
    val flash: Boolean,   // 打完要不要闪一下屏
    val isFinal: Boolean, // 是不是最后一行（样式不同）
)
```

逐字打出当前行，完成后加入已显示列表（永久留在屏幕上），然后开始下一行。最后一行用更大的字号。其中一行还会触发一次屏幕闪烁（叠加一个低透明度遮罩 400ms）。

你可以写自己的文案序列，调节节奏感。重点是让等待冷却的这段时间不是干看倒计时，而是一段有压迫感的体验。

---

## 解锁过渡动画

输对密码后不直接跳进主页，而是播一段过渡。我们做了三个阶段：

1. **黑屏** 300ms
2. **一句话**（可配置，比如"我们去往语言的尽头，而后沉醉不归。"）1500ms
3. **App 名**（"— 渊海 —"）1000ms
4. 淡入主页

用 Compose 的 `Crossfade` 切换，每个阶段之间有 400ms 的交叉淡入淡出。

```kotlin
enum class TransitionState { Initial, Interstitial, Yuanhai }

LaunchedEffect(Unit) {
    onUnlockStarted()   // 让主界面在动画下面提前开始组合
    delay(300)
    state = TransitionState.Interstitial
    delay(1500)
    state = TransitionState.Yuanhai
    delay(1000)
    onComplete()
}
```

> 有个细节：动画开始时就调 `onUnlockStarted()`，让主界面在锁屏动画下面提前组合。这样动画结束揭开的时候主页已经准备好了，不会白屏。动画期间用一个透明的 `pointerInput` 吃掉所有触摸事件，防止穿透点到下面。

过渡语和 App 名都可以在设置里自定义，也可以关掉过渡直接进。

---

## 圆点输入动画

PIN 输入时，每填一位，对应的圆点有一个 1.15x 的弹跳放大效果：

```kotlin
val scale by animateFloatAsState(
    targetValue = if (index < pin.length) 1.15f else 1f,
    animationSpec = spring(dampingRatio = 0.6f)
)
```

很小的细节，但让输入过程不那么干巴巴的。

---

## 错误消息和圆点的时序编排

错误消息和圆点共用同一块空间，需要精心编排切换时序，避免两个东西同时出现打架：

```kotlin
// 验证失败后的时序
haptic.perform(HapticFeedbackType.LongPress)  // 触觉反馈
showError = true                                // 显示错误消息
errorMessage = pickErrorMessage(...)
delay(1500)                                     // 让错误消息停留够久
pin = ""                                        // 清空 PIN（圆点消失）
delay(100)                                      // 等圆点淡出完
showError = false                               // 再隐藏错误消息（圆点淡入）
```

错误消息和圆点用 `AnimatedVisibility` 互斥切换——一个显示时另一个隐藏，有各自的淡入淡出动画。

---

## 这些都不是必须的

上面每一项都可以不做，App 照样能用。没有递进消息，就显示固定的"密码错误"；没有打字机效果，锁定期间就只显示倒计时；没有过渡动画，验证通过直接进主页。

但如果你想让你的锁不只是一把锁，而是你和他之间的一道门——值得花时间在这些细节上。
