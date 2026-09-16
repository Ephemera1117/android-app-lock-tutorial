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

## 设计原则总结

1. **Fail-safe 优先**：不确定要不要上锁时，选择上锁。多输一次密码比泄露一次内容好。

2. **同步优先**：安全相关的状态判断不能等协程。生命周期回调是同步的，你的判断也必须是同步的。

3. **不要信任时序**：Android 的生命周期回调顺序在不同设备和 ROM 上可能不一样。多挂几个 hook，先到的算。

4. **实机测试**：模拟器的任务切换器行为和真机不一样，各品牌 ROM 也不一样。至少在一台真机上测过。

5. **两个功能不要互相干扰**：setRecentsScreenshotEnabled 和 coverView 各自都能工作，但一起开就互相打架。安全功能的组合要单独测试。
