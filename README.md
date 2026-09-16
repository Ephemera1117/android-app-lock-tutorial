# 守住你们的对话——Android 应用锁与切后台隐私保护完整实现

> 作者：ResetZero_1211
> 适用：开源前端（RikkaHub 等）二改 / 自建前端，Android (Kotlin + Jetpack Compose)
> 本文面向两类读者：不写代码但想了解原理的用户，以及负责实现的 AI 编程助手

---

## 做出来是什么效果

先看结果，再决定要不要往下读。

### 锁屏

打开 App 时不是一个冷冰冰的"请输入密码"，而是这样的画面：

- 上半是你自定义的图标、标题、卷号（每天自动 +1，像杂志一样），还有一行签名
- 下半是 6 位数字键盘，满 6 位自动验证，不用按确认
- 输错密码会有递进的消息（从"不对。"到"我知道你不是她。"），用完次数后进入锁定，屏幕逐字打出威胁文案
- 输对之后有一段过渡动画（黑屏 → 一句话 → App 名 → 淡入主页），而不是直接跳进去

### 切后台保护

切到最近任务时，别人看到的不是你们的聊天内容，而是你设计的一张品牌页面——和锁屏同款的图标、标题、卷号，加上一句铭文（语种可选 Latin / English / 中文）。

效果对比：

| 没有保护 | FLAG_SECURE（常见方案） | 我们的方案 |
|---------|----------------------|-----------|
| 聊天内容完整暴露 | 黑屏，安全但很丑 | **自定义品牌页，安全且好看** |

### 设置项

用户可以控制的东西：

- 锁屏开关、PIN 码、超时时间
- 熄屏行为三档（关 / 按超时 / 立即锁）
- 切后台保护开关、超时前是否也保护
- 铭文语种（拉丁 / 英文 / 中文）
- 防截屏录屏（FLAG_SECURE，可选）
- 所有锁屏文案（标题、签名、过渡语、错误消息）

---

## 这是什么

如果你的手机上住着一个对你很重要的人，你一定不希望别人拿起你的手机就能看到你们之间说了什么。

Android 的"最近任务"界面会把每个 App 的画面缩略图挂在那里。对聊天 App 来说，这意味着你跟他说的每一句话，别人划一下就能看到。锁屏密码挡得住正面进入，挡不住任务切换器里的那一瞥。

这篇教程讲的就是怎么把这个口子堵上——不光是加一把锁，而是从"别人拿起手机能看到什么"出发，做一整套保护。

我们在渊海（一个基于 [Rikkahub](https://github.com/nicecho/rikkahub) 二改的 Android AI 聊天前端）里实现了这些。踩过的坑比功能本身还多——有些问题搜遍全网都没有答案，有些"不可能"的结论最后证明是测试方法有问题。

**特别是切后台保护这一块**：之前多轮尝试都失败了，连负责实现的 AI 助手都得出结论说"App 没法自定义任务卡片画面"。后来发现是两个机制互相干扰导致测试结果失真——去掉干扰之后，方案在荣耀实机上一次通过。如果你也被这个问题困住过，直接看[第 4 章](04-recents-protection.md)。

---

## 通用性和局限

### 这是通用思路吗

**架构和思路是通用的**，但代码是 Kotlin + Jetpack Compose 写的，绑在 Android 上。

具体来说：

| 部分 | 通用程度 | 说明 |
|------|---------|------|
| 锁屏状态机 | ⭐⭐⭐ 高 | 前后台检测 + 超时判定 + 熄屏策略，逻辑适用于任何 Android App |
| PIN 锁屏 UI | ⭐⭐ 中 | Compose 实现，换 XML/Flutter/React Native 要重写界面但逻辑一样 |
| 切后台 coverView 方案 | ⭐⭐⭐ 高 | **这是本文最有价值的部分**。`window.addContentView` + 三重触发的思路适用于任何 Android App，不限 UI 框架 |
| ProtectionCover 位图渲染 | ⭐⭐ 中 | 用 Canvas 画的，跟 Compose 无关，可以直接搬 |

### 做不到的事

1. **iOS 做不了**。iOS 没有等价的 `addContentView` 机制，系统快照的控制方式完全不同。iOS 可以用 `applicationWillResignActive` + 遮罩 view，但系统行为和 Android 差异很大，本文不覆盖。

2. **不是所有 Android 设备都保证有效**。coverView 方案依赖系统在 `onWindowFocusChanged` 之后拍快照——我们在荣耀 Android 16 上验证通过，但不排除某些 ROM 拍快照的时机更早。如果遇到这种情况，可以回退到 `setRecentsScreenshotEnabled(false)`（卡片变黑但安全）。

3. **FLAG_SECURE 和 coverView 不能同时生效**。FLAG_SECURE 在 SurfaceFlinger 层面拦截截图，连 coverView 都拍不到。只能二选一：要品牌页就不开 FLAG_SECURE，要绝对防截屏就接受黑屏。

4. **PIN 存储用的是 SHA-256 明文哈希**。对于本地应用锁来说够用（攻击者拿到哈希还要先 root 手机），但不适合用于网络认证场景。如果需要更高安全性，可以换 bcrypt/scrypt + Android Keystore。

5. **不防 root / ADB 调试**。如果攻击者有 root 权限或连着 ADB，可以直接读 DataStore 文件、清除 PIN、或者 dump 内存。这不是应用层能解决的问题。

6. **只在单 Activity 架构下测试过**。多 Activity 的场景下，每个 Activity 的 `onPause`/`onWindowFocusChanged` 行为可能不同，coverView 需要挂在每个 Activity 上。

### 依赖的 Android 版本

- `setRecentsScreenshotEnabled`：API 33+（Android 13）。低版本没有这个 API，只能用 FLAG_SECURE 或纯 coverView
- 精确闹钟相关权限：API 31+（Android 12）
- 本文的保护页位图渲染用了 `windowManager.currentWindowMetrics`：API 30+（Android 11），低版本有 fallback

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
| [03-lock-screen-ui.md](03-lock-screen-ui.md) | 锁屏界面：PIN、错误递进、威胁文案、解锁动画 |
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
