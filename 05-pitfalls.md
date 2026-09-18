# 5. 踩坑总结

> 下面每一条都是实机出过问题才发现的。有些坑搜遍全网找不到答案，有些"不可能"最后证明是测试方法有问题。如果你正在做类似的事情，这张表能帮你绕过我们走过的弯路。

## 踩坑表格

| # | 坑 | 现象 | 原因 | 解法 |
|---|-----|------|------|------|
| 1 | **setRecentsScreenshotEnabled 干扰 coverView** | coverView 贴上了但卡片还是黑屏 | 两个机制冲突：系统被告知"不要拍"，所以 coverView 再好看也拍不到 | 去掉 setRecentsScreenshotEnabled(false)，让系统正常拍 |
| 2 | **Compose 帧时钟在 onPause 后停止** | 在 onPause 里改 Compose 状态想显示遮罩，但画面不变 | Activity paused 后 Compose 的 frame clock 停了，状态改了也不会触发重组 | 用真 View（ImageView），不用 Compose |
| 3 | **WindowManager.addView 创建的是独立窗口** | 用 WindowManager 挂的遮罩窗口不出现在任务快照里 | 任务快照只采集主窗口缓冲区，独立窗口不在范围内 | 用 window.addContentView（加到 DecorView 里） |
| 4 | **用 Flow 传递锁屏状态会晚一帧** | 回到 App 时先闪一帧主页，然后才上锁屏 | Flow 需要一次协程调度，晚于第一帧绘制 | 用 mutableStateOf，同步读取 |
| 5 | **ensureResolvedForForeground 不幂等会误判熄屏** | 设了「熄屏不算离开」，但熄屏回来还是被要求输密码 | onStart 和 onResume 都调了 ensureResolved，第二次调用时 screenWasOff 标记已被消费，走进超时判定 | 加 foregroundResolved 标记，算过就不再算 |
| 6 | **「立即锁」不清 unlockedThisProcess** | 锁屏挂着的时候切到别的 App 再切回来，锁屏自己打开了 | 熄屏「立即锁」只设了 shouldShowLockScreen=true 但没清 unlockedThisProcess，回来时超时判定以为"已验证过" | 立即锁时同时清 unlockedThisProcess = false |
| 7 | **冷启动后短暂离开绕过锁屏** | 进程被杀后重启，切走再切回来（没超时），不上锁 | 新进程的 unlockedThisProcess 默认 false，但如果不检查这个就直接走超时判定，短暂离开会被放行 | computeLockState 里显式检查 unlockedThisProcess |
| 8 | **锁屏已显示时被重新计算抹掉** | 某些时序下锁屏自己消失 | 回前台重算 computeLockState 时，如果没超时就 return false，把已显示的锁屏抹掉了 | 加规则：_shouldShowLockScreen.value == true 时一律 return true |
| 9 | **TaskDescription 的 icon/颜色在 Honor 上不生效** | 设了自定义底色和图标，任务卡片完全没变化 | Honor 的任务切换器不使用 TaskDescription 的 icon 和 backgroundColor | 代码留着（换设备可能有效），但不依赖它 |
| 10 | **onPause 里翻 visibility 在某些设备来不及** | coverView 翻成 VISIBLE 了但卡片上还是原画面 | 系统拍快照的时机可能早于 onPause 回调完成 | 加 onWindowFocusChanged(false)，比 onPause 更早触发 |
| 11 | **DataStore 在 onCreate 时还没就绪** | 冷启动时读设置拿到的是默认值 | DataStore 的第一次 emit 需要时间 | 状态机默认锁屏（fail-safe），等设置到位补算 |
| 12 | **onWindowFocusChanged 被 Dialog 触发** | 弹出系统权限对话框也触发了 coverView | onWindowFocusChanged(false) 在弹 Dialog 时也会调用 | coverNeeded() 会检查锁是否开着，没开锁不盖；如果需要更精确，可以检查 isFinishing |
| 13 | **回来时直接揭 coverView 会闪一帧内容** | 超时锁屏回来时先看到聊天内容再弹锁屏 | coverView (GONE) 和 Compose LockScreen 渲染之间有一帧空档 | 锁屏要显示时不揭 coverView，在 `onWindowFocusChanged(true)` 时揭（此时 LockScreen 已渲染完毕）。之前试过 Compose `SideEffect` 但不可靠——LockScreen 如果离开前就在显示，回来时没有新组合，SideEffect 不执行 |
| 14 | **lockEnabled 关着时 coverView 永远不揭** | 锁屏功能关掉后切后台再回来，保护页一直挡着 | shouldShowLockScreen 初始值 true（冷启动安全默认），lockEnabled=false 时 LockScreen 不渲染、signalLockScreenDrawn 不会被调 | hideLeaveCoverSafely 同时检查 lockEnabled 和 lockPinHash，不只看 shouldShowLockScreen |
| 15 | **SideEffect 不会在回前台时重新执行** | 用 Compose SideEffect 揭 coverView，第一次有效，之后回来就卡在保护页 | SideEffect 只在新组合时执行；如果 LockScreen 离开前就在显示，回来时状态没变、没有新组合 | 改用 onWindowFocusChanged(true) 揭 coverView，不依赖 Compose 组合时机 |
| 16 | **Activity 不是 FragmentActivity → 指纹点了没反应** | 点锁屏上的图标，**什么都不发生**：不弹框、不报错、不崩溃 | `BiometricPrompt` 的构造函数只接受 `FragmentActivity`。`ComponentActivity` 不是它的子类，`context as? FragmentActivity` 静默返回 null，于是整段逻辑被跳过 | 把 Activity 基类改成 `FragmentActivity`（它是 `ComponentActivity` 的子类，其他用法不受影响）。排查靠把中间结果落文件，**不要等日志**——这台设备上 `Log.d` 根本读不到 |
| 17 | **PromptInfo.build() 抛 `Negative text must be set and non-empty`** | 点图标直接崩溃 | 库里两条互斥校验：验证方式**不含**「设备密码」时负按钮文字必填；**含**时反过来绝不能填。只调 `setAllowedAuthenticators(BIOMETRIC_WEAK)` 而不给负按钮文字就命中第一条 | 不含设备密码 → 加 `setNegativeButtonText("取消")`；含设备密码（`DEVICE_CREDENTIAL`）→ 绝不能加 |
| 18 | **给序列化的设置加字段是安全的，改类型不是** | 老用户升级后设置回默认值，或者整个设置读不出来 | 设置整体是一个 JSON 存在 DataStore 里，字段类型一变老 JSON 解析失败 | **只加新字段**并给安全默认值；要改语义就加新字段 + 用哨兵值（比如 `-1`）表示"没设过"，读的时候换算。老字段留着不动 |

## 设计原则总结

1. **Fail-safe 优先**：不确定要不要上锁时，选择上锁。多输一次密码比泄露一次内容好。

2. **同步优先**：安全相关的状态判断不能等协程。生命周期回调是同步的，你的判断也必须是同步的。

3. **不要信任时序**：Android 的生命周期回调顺序在不同设备和 ROM 上可能不一样。多挂几个 hook，先到的算。

4. **实机测试**：模拟器的任务切换器行为和真机不一样，各品牌 ROM 也不一样。至少在一台真机上测过。

5. **两样东西不要互相干扰**：setRecentsScreenshotEnabled 和 coverView 各自都能工作，但一起开就互相打架。安全相关的东西组合起来要单独测。

6. **系统不让你做的事，别硬做**：指纹界面不能自绘、FLAG_SECURE 下拍不到自己的保护页、某些 ROM 不认 TaskDescription——这些都是系统层面的限制，绕过它们花的时间远多于收益。**先去确认"这件事到底能不能做"**，再决定怎么实现。判断不了就翻系统/库的源码或文档，别靠试。
