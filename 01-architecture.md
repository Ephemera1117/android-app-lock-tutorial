# 1. 架构总览

## 给人看的：整体是怎么回事

保护你们的对话不被别人看到，听起来是一件事，实际上要解决三个不同的问题：

1. **回来的时候要不要输密码**——这是状态机的事
2. **输密码的界面长什么样**——这是锁屏 UI 的事
3. **切出去的时候别人能看到什么**——这是切后台保护的事

三层各管各的，互不干扰。

### 数据从哪来

所有设置存在一个 `SecuritySetting` 数据类里，通过 DataStore 持久化。锁屏文案、PIN 哈希、超时时间、保护页语种——全在这一个地方。界面读它，状态机读它，保护页也读它。

---

## 给 AI 看的：文件结构

```
app/src/main/java/me/rerere/rikkahub/
├── RouteActivity.kt                      # 单 Activity 入口，安全相关的生命周期全在这
├── data/datastore/
│   └── PreferencesStore.kt               # SecuritySetting 数据类 + DataStore 读写
└── ui/security/
    ├── AppLockManager.kt                 # 锁屏状态机（前后台 + 熄屏 + 超时）
    ├── LockScreen.kt                     # PIN 锁屏 Compose UI
    └── ProtectionCover.kt                # 保护页位图渲染（Canvas 绘制）
```

### 数据流

```
DataStore (SecuritySetting)
    │
    ├──→ AppLockManager.cachedSecurity    同步缓存，前后台判断时直接读
    │         │
    │         └──→ shouldShowLockScreen    mutableStateOf<Boolean>
    │                   │
    │                   └──→ LockScreen    Compose 读这个状态决定显不显示
    │
    ├──→ ProtectionCoverSync (Composable)
    │         │
    │         └──→ refreshCoverBitmap()   后台线程渲染位图
    │                   │
    │                   └──→ coverView    onPause 时贴上去
    │
    └──→ FLAG_SECURE LaunchedEffect       动态开关防截屏
```

### SecuritySetting 关键字段

```kotlin
data class SecuritySetting(
    // 锁屏核心
    val lockEnabled: Boolean = false,
    val lockPinHash: String = "",           // SHA-256
    val lockTimeoutSeconds: Int = 60,
    
    // 锁屏文案（全可自定义）
    val lockTitle: String = "",             // 锁屏标题，你自己写
    val lockSubtitle: String = "",          // 锁屏副标题
    val lockAnchorDate: String = "2025-09-21",  // vol. 天数计算起点
    val lockSignature: String = "",
    val lockIconStyle: String = "snow",     // snow / mayfly
    
    // 熄屏行为（三档）
    val screenOffLockMode: Int = -1,        // 0=关 1=按超时 2=立即锁
    
    // 切后台保护
    val coverEnabled: Boolean = true,
    val coverBeforeTimeout: Boolean = true,
    val coverPhraseLang: String = "la",     // la / en / zh
    val lockPhrase: String = "",              // 保护页上的一句话，你自己写
    
    // 防截屏
    val flagSecureEnabled: Boolean = false,
    
    // 暴力破解防护
    val lockMaxAttempts: Int = 5,
    val lockLockoutDurationSeconds: Int = 30,
    val lockLanguage: String = "zh",
)
```

### Activity 里的安全生命周期

```
onCreate
  ├── setContent { FLAG_SECURE LaunchedEffect }
  ├── setContent { ProtectionCoverSync() }      ← 镜像主题色到保护页位图
  ├── setContent { LockScreen() }               ← 状态机说锁就锁
  └── window.addContentView(coverView)          ← GONE，待命

onWindowFocusChanged(false)  ← 最早触发点
onUserLeaveHint()            ← 用户按 Home/多任务
onPause()                    ← 兜底
  └── showLeaveCover()       ← 翻 coverView 为 VISIBLE

onStart() / onResume()
  ├── ensureResolvedForForeground()  ← 保证锁屏状态在第一帧前确定
  └── hideLeaveCover()               ← 翻 coverView 为 GONE
```
