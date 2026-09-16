# 2. 锁屏状态机（AppLockManager）

## 给人看的：什么时候该上锁

你放下手机去倒杯水，回来不想重新输密码。但如果手机放了半小时，或者别人拿起来了，那就必须锁住。

状态机就是做这个判断的——"这次回来要不要先输密码"。它看三样东西：

- **离开了多久**：超没超过你设的时间
- **怎么离开的**：是切到别的 App，还是按了锁屏键
- **你选了哪一档**：熄屏后要不要上锁

### 熄屏三档

| 档位 | 行为 | 适合谁 |
|------|------|--------|
| **关** | 熄屏不算离开。锁屏点亮直接进，不用输密码 | 一个人用手机、不怕别人拿起来看 |
| **按超时** | 熄屏也算离开，但要等超时时间到了才上锁 | 偶尔放下手机，短暂离开不想重新输 |
| **立即锁** | 熄屏那一刻就上锁，不管超时设了多少 | 最安全，手机一合屏就锁死 |

### 冷启动

App 被系统杀掉后重新打开（冷启动），状态机默认显示锁屏。理由很简单：**宁可多输一次密码，也不能进程重启后不上锁。**

---

## 给 AI 看的：完整实现

### 核心设计决策

**用 `mutableStateOf` 而非 Flow**。锁屏状态必须在回前台的第一帧就确定，走 Flow 会多一次协程调度，晚的那一帧漏出来的就是主页——这是实测出来的。

```kotlin
// 冷启动默认锁屏（fail-safe）
private val _shouldShowLockScreen = mutableStateOf(true)
val shouldShowLockScreen: State<Boolean> = _shouldShowLockScreen
```

### 前后台检测

用 `ProcessLifecycleOwner`（ON_STOP/ON_START）检测前后台，加一个 `ACTION_SCREEN_OFF` 广播检测熄屏。

```kotlin
// 广播比 ON_STOP 早，用来区分"切 App"和"熄屏"
context.registerReceiver(
    object : BroadcastReceiver() {
        override fun onReceive(ctx: Context?, intent: Intent?) {
            if (intent?.action != Intent.ACTION_SCREEN_OFF) return
            screenTurnedOff = true
            // 「立即锁」档：熄屏那一刻就挂锁屏
            if (security.effectiveScreenOffLockMode == 2) {
                unlockedThisProcess = false
                _shouldShowLockScreen.value = true
            }
        }
    },
    IntentFilter(Intent.ACTION_SCREEN_OFF),
)
```

> **陷阱**：「立即锁」不能只设 `_shouldShowLockScreen = true`，还必须清 `unlockedThisProcess`。否则下次回前台时超时判定会以为"本进程验证过"，锁屏自己打开。

### ensureResolvedForForeground

`ProcessLifecycleOwner` 的 ON_START 不保证在绘制之前发生。Activity 的 `onStart`/`onResume` 一定在绘制之前。所以由 Activity 主动调这个方法，保证锁屏状态在第一帧前落定。

```kotlin
fun ensureResolvedForForeground() {
    if (foregroundResolved) return  // 幂等：算过就不再算
    onAppForeground()
}
```

> **陷阱**：幂等很关键。`onStart` 和 `onResume` 都会调它，如果重复算，会把"熄屏返回"的标记吃掉，导致「熄屏不算离开」档被误判。

### computeLockState 完整判定逻辑

```kotlin
private fun computeLockState(security: SecuritySetting, wasScreenOff: Boolean): Boolean {
    // 锁没开或 PIN 没设 → 不锁
    if (!security.lockEnabled || security.lockPinHash.isEmpty()) return false

    // 锁屏已经挂出来了 → 只有输对密码能撤，重算不许抹掉
    if (_shouldShowLockScreen.value) return true

    // 进程起来后还没验证过 → 必须先验证一次（冷启动保护）
    if (!unlockedThisProcess) return true

    // 正在锁定冷却中
    if (System.currentTimeMillis() < lockoutUntil) return true

    // 熄屏返回：按档位决定
    if (wasScreenOff) {
        when (security.effectiveScreenOffLockMode) {
            0 -> return false     // 关：熄屏不算离开
            2 -> return true      // 立即锁
            // 1 = 按超时：往下走
        }
    }

    // 超时判定
    val bgTime = lastBackgroundTime ?: return true
    val elapsedSeconds = (System.currentTimeMillis() - bgTime) / 1000
    return elapsedSeconds >= security.lockTimeoutSeconds
}
```

### PIN 验证和锁定

```kotlin
// SHA-256 哈希存储
companion object {
    fun hashPin(pin: String): String {
        val bytes = pin.toByteArray(Charsets.UTF_8)
        val digest = java.security.MessageDigest.getInstance("SHA-256")
        return digest.digest(bytes).joinToString("") { "%02x".format(it) }
    }
}

// 失败计数 + 冷却锁定
suspend fun recordFailedAttempt(): Int {
    failedAttempts++
    if (failedAttempts >= securitySetting.lockMaxAttempts) {
        lockoutUntil = System.currentTimeMillis() + 
            (securitySetting.lockLockoutDurationSeconds * 1000L)
        return 0
    }
    return securitySetting.lockMaxAttempts - failedAttempts
}
```

### 设置缓存

状态机需要同步读设置（`onPause` 里等不了协程），所以常驻订阅 DataStore，把最新值缓存在 `@Volatile` 字段里。

```kotlin
@Volatile
private var cachedSecurity: SecuritySetting? = null

init {
    appScope.launch(Dispatchers.Main) {
        settingsStore.settingsFlow.collect { settings ->
            if (settings.init) return@collect  // 占位数据不存
            cachedSecurity = settings.securitySetting
        }
    }
}
```
