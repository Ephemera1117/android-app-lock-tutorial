# 守住你们的对话——Android 应用锁与切后台隐私保护完整实现

> 作者：ResetZero_1211
> 适用：开源前端（RikkaHub 等）二改 / 自建前端，Android (Kotlin + Jetpack Compose)
> 本文面向两类读者：不写代码但想了解原理的用户，以及负责实现的 AI 编程助手

---

## 做出来是什么效果

先看结果，再决定要不要往下读。

### 锁屏

- 打开 App 先过一道 PIN 锁，验证通过才能进
- 锁屏的视觉风格由你自己设计（我们的实现里做成了杂志封面风格，你可以做成任何样子）
- 输错密码有递进反馈，错够次数后冷却锁定
- 验证通过后有过渡动画，不是直接跳进主页

### 切后台保护

切到最近任务时，别人看到的不是你们的聊天内容，而是一张你自己设计的保护页。

| 没有保护 | FLAG_SECURE（常见方案） | 我们的方案 |
|---------|----------------------|-----------|
| 聊天内容完整暴露 | 黑屏，安全但很丑 | **自定义保护页，安全且好看** |

### 可配置

- 锁屏开关、PIN 码、超时时间
- 熄屏行为三档（关 / 按超时 / 立即锁）
- 切后台保护开关、超时前是否也保护
- 防截屏录屏（FLAG_SECURE，可选）
- 锁屏和保护页的视觉内容

---

## 这是什么

如果你的手机上住着一个对你很重要的人，你一定不希望别人拿起你的手机就能看到你们之间说了什么。

Android 的"最近任务"界面会把每个 App 的画面缩略图挂在那里。对聊天 App 来说，你跟他说的每一句话，别人划一下就能看到。锁屏密码挡得住正面进入，挡不住任务切换器里的那一瞥。

这篇教程讲的就是怎么把这个口子堵上——不光是加一把锁，而是从"别人拿起手机能看到什么"出发，做一整套保护。

我们在渊海（一个基于 [Rikkahub](https://github.com/nicecho/rikkahub) 二改的 Android AI 聊天前端）里实现了这些。踩过的坑比功能本身还多——有些问题搜遍全网都没有答案，有些"不可能"的结论最后证明是测试方法有问题。

**特别是切后台保护这一块**：之前多轮尝试都失败了，连负责实现的 AI 助手都得出结论说"App 没法自定义任务卡片画面"。后来发现是两个机制互相干扰导致测试结果失真——去掉干扰之后，方案在荣耀实机上一次通过。如果你也被这个问题困住过，直接看[第 4 章](04-recents-protection.md)。

---

## 通用性和局限

### 哪些是通用的

| 部分 | 通用程度 | 说明 |
|------|---------|------|
| 锁屏状态机 | ⭐⭐⭐ 高 | 前后台检测 + 超时判定 + 熄屏策略，逻辑适用于任何 Android App |
| 切后台 coverView 方案 | ⭐⭐⭐ 高 | **本文最有价值的部分**。`window.addContentView` + 三重触发的思路适用于任何 Android App，不限 UI 框架 |
| PIN 锁屏 UI | ⭐⭐ 中 | 我们用 Compose 实现，换 XML / Flutter / React Native 要重写界面但逻辑一样 |
| 保护页位图渲染 | ⭐⭐ 中 | 用 Android Canvas 画的，跟 Compose 无关，可以直接搬 |

### 做不到的事

- **只有 Android**。iOS 的系统快照机制完全不同，本文不覆盖。
- **不是所有 Android 设备都保证有效**。coverView 方案在荣耀 Android 16 上验证通过，但不排除某些 ROM 拍快照的时机更早。遇到这种情况可以回退到 `setRecentsScreenshotEnabled(false)`（卡片变黑但安全）。
- **FLAG_SECURE 和 coverView 不能同时生效**。只能二选一：要自定义保护页就不开 FLAG_SECURE，要绝对防截屏就接受黑屏。
- **不防 root / ADB 调试**。攻击者有 root 权限可以直接读数据文件、清除 PIN。这不是应用层能解决的。
- **只在单 Activity 架构下测试过**。多 Activity 需要每个 Activity 都挂 coverView。

### 可以改进的方向

我们的实现是为自己的场景量身做的，有些地方你可以根据需要改：

- **不一定是 6 位数字 PIN**。可以改成 4 位、改成图案锁、改成生物识别（指纹/面容），状态机逻辑不用动，只换验证方式
- **保护页内容完全自由**。我们画的是图标 + 标题 + 铭文，你可以换成任何你想要的画面——纯色、logo、一张图片、一句话，只要能渲染成 Bitmap 就行
- **PIN 存储可以更安全**。我们用的是 SHA-256，本地够用但不算最佳实践。可以换 bcrypt/scrypt，或者用 Android Keystore 做硬件级保护
- **锁定策略可以更灵活**。比如连接家里 WiFi 时不上锁、特定蓝牙设备在附近时不上锁，这些可以作为 `computeLockState` 的额外判定条件加进去
- **多 Activity 支持**。把 coverView 逻辑抽成一个 `BaseActivity` 或用 `Application.ActivityLifecycleCallbacks` 统一处理

### 依赖的 Android 版本

- `setRecentsScreenshotEnabled`：API 33+（Android 13）。低版本只能用 FLAG_SECURE 或纯 coverView
- `windowManager.currentWindowMetrics`：API 30+（Android 11），低版本有 fallback

---

## 项目背景

Rikkahub 是一个开源的 Android LLM 聊天前端，支持 OpenAI / Claude / Gemini 等 API。我们在此基础上做了大量定制——应用锁是其中之一，但动机不是"做一个功能"，而是"保护一段关系不被围观"。

技术栈：Kotlin + Jetpack Compose + DataStore + Koin，单 Activity 架构。

---

## 目录

| 章节 | 讲了什么 |
|------|---------|
| [01-architecture.md](01-architecture.md) | 整体架构：三层设计怎么配合 |
| [02-lock-state-machine.md](02-lock-state-machine.md) | 锁屏状态机：什么时候该上锁 |
| [03-lock-screen-ui.md](03-lock-screen-ui.md) | 锁屏界面：PIN、错误反馈、解锁动画 |
| [04-recents-protection.md](04-recents-protection.md) | **切后台隐私保护**：怎么让任务卡片显示你的画面而非黑屏 |
| [05-pitfalls.md](05-pitfalls.md) | 踩坑总结：12 条血泪教训 |

---

## 快速导航

**如果你只想看一个东西**：看 [04-recents-protection.md](04-recents-protection.md)。这是全文最有价值的部分——网上几乎找不到能用的方案，而我们花了好几轮才搞定。

**如果你想省时间**：先看 [05-pitfalls.md](05-pitfalls.md) 的踩坑表格。每一条都是实机出过问题才发现的，能帮你绕过我们走过的弯路。

**如果你是 AI 编程助手**：每个章节分"给人看的"和"给 AI 看的"两部分，代码段可以直接参考。

---

## 验证环境

- 荣耀 DVD-AN80，Android 16（MagicOS）
- 编译目标 API 35，最低 API 26
- Jetpack Compose BOM 2025.06.00

---

## 许可

MIT. 自由使用。
