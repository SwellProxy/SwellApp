# Swell

**面向零技术基础用户的 iOS 代理客户端。**

基于 [sing-box](https://github.com/SagerNet/sing-box) 内核,用人话代替术语 —— 全 App 没有一个「规则」「出站」「urltest」这样的词。

---

## 🎨 设计

设计已封板,以下两份可以直接在浏览器打开:

| | 说明 |
|---|---|
| **[▶ 交互原型](https://swellproxy.github.io/SwellApp/design/Swell%20%E4%BA%A4%E4%BA%92%E5%8E%9F%E5%9E%8B.html)** | **开发以此为准。** 可点击走完整流程,含全部空态 / 加载 / 失败 / 边界态。顶部有屏号索引,Tweaks 面板可切换「国家 = 中国 / 伊朗」验证国际化 |
| **[◻ UI 设计稿](https://swellproxy.github.io/SwellApp/design/Swell%20UI%20%E8%AE%BE%E8%AE%A1%E7%A8%BF.html)** | 全部屏幕平铺在一张画布上,带设计意图标注。**仅供概览** |

> 两份的差异:原型有交互和状态,画布只有静态图。**冲突时以原型为准。**

截图速览:[`design/screenshots/`](design/screenshots)

---

## 🧩 项目结构

```
SwellApp/
├── CLAUDE.md              # 给 AI 协作者:技术栈决策 + 内核红线 + 硬规则
├── design/                # 设计规格(已封板)
│   ├── Swell 交互原型.html   ★ 唯一真相
│   ├── Swell UI 设计稿.html   概览画布
│   └── screenshots/
├── specs/                 # 业务规格(HTML 里没有这些信息)
│   ├── 01-内核能力与修改需求-v1.md
│   ├── 02-第二轮复核-v2.md
│   ├── 03-第三轮复核与封板-v3.md
│   ├── 04-订阅解析规格-给开发.md     ← 实现订阅解析前必读
│   └── 05-机场接入规范-v1.md         ← 给机场的对外规范
├── SwellApp/              # 源码(UIKit)
└── SwellApp.xcodeproj
```

---

## ⚙️ 技术栈

| 项 | 选择 |
|---|---|
| UI | **UIKit**(SwiftUI 仅按需嵌入,理由见 [CLAUDE.md §0](CLAUDE.md)) |
| 内核 | [SwellProxy/sing-box](https://github.com/SwellProxy/sing-box) — 官方 sing-box 的 fork |
| 引入 | SwiftPM binaryTarget,版本 `1.13.14-swell.1` |

### 为什么要 fork 内核

官方 sing-box 的 `urltest` 测的是**冷启动全链路**延迟(含 TCP / TLS / 代理握手),数字比 Karing、mihomo 高 2–4 倍,用户会以为 Swell 慢。

我们的 fork 只改一个函数:复用同一条连接再请求一次,取**热 RTT**。函数签名未变,不影响任何调用方。

xcframework 由 fork 的 GitHub Actions 自动构建并发布到 Release。

---

## 🚧 v1 范围

**做:** 智能分流 / 全局代理 / 全局直连三选一 · 服务例外(AI 与 X 锁定单条线路)· 场景规则(蜂窝 / 低数据模式 / Wi-Fi 直连)· 订阅管理 · 国际化(国家可选)

**不做:** 链式代理 · 流媒体解锁检测 · 落地 IP 检测(留到 v1.1)

---

## ⛔ 内核红线

这些是**已核实的 sing-box 事实**,UI 文案已按此校准。完整列表见 [CLAUDE.md §3](CLAUDE.md)。

| 想做的 | 现实 |
|---|---|
| iOS 按 App 分流 | **做不到**。iOS 不告诉隧道流量属于哪个 App,只能按域名 / IP。所以全 App 用「网站与服务」,不用「应用」 |
| 自动挑「能解锁 Netflix」的线路 | 内核只测延迟。v1 不承诺解锁 |
| 又要自动选最快、又要 IP 不变 | **互斥**。🔒 = 锁定单条(IP 不变);🔄 = 自动(会切换) |
| 内核知道线路在哪个国家 | **不知道**。靠解析线路名 —— 实测某机场八个国家的节点共用同一个新加坡入口 |
| 内核有「倍率」字段 | **没有**。只能解析线路名 |

---

## 🤝 给机场

想为 Swell 用户提供更好体验?见 **[机场接入规范](specs/05-机场接入规范-v1.md)**。

- **零破坏** —— 你现有的订阅一个字节都不用改,其他客户端完全不受影响
- **渐进增强** —— 什么都不做也能用;做得越多体验越好
- 提供 `region`(落地地区)和 `multiplier`(倍率)后,你的线路名可以彻底变干净,公告也不用再伪装成假节点

---

## 📄 许可证

sing-box 采用 **GPL-3.0**,Libbox 链接进本 App 即产生 GPL 义务,因此本仓库保持公开。
