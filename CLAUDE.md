# Swell — iOS 代理客户端

面向零技术基础用户的 iOS 代理 App。Swift + sing-box(Libbox)。

---

## 1. 项目结构

```
SwellApp/
├── CLAUDE.md              ← 本文件
├── design/                ← UI 规格(设计已封板)
│   ├── Swell-Prototype.dc.html   ★ 可交互原型 = 唯一真相
│   ├── Swell-UI.dc.html          概览画布(带设计标注)
│   ├── TabBar.dc.html
│   ├── ios-frame.jsx             原型依赖,勿删
│   └── support.js                原型依赖,勿删(HTML 用 ./support.js 同目录引用)
├── specs/                 ← 业务规格(HTML 里没有这些信息)
└── Swell/                 ← Xcode 工程
```

### 设计规格怎么读

`design/Swell-Prototype.dc.html` 是**唯一真相**,不是 `Swell-UI.dc.html`。

- 原型包含全部**状态与交互**;画布只有静态图,仅供概览
- 原型顶部有屏号索引;`captions` 开关可显示设计标注
- 原型的 Tweaks 里有 `国家 = 中国 / 伊朗` 开关,用于验证国际化文案替换
- **本地用浏览器打开即可**,不需要起服务

> ⚠️ **HTML → SwiftUI 是照着重写,不是转换。** 从 HTML/CSS 里读精确的色值、字号、间距、圆角,用 SwiftUI 重新实现。不要试图移植 DOM 结构。

---

## 2. 内核

**不要用官方 sing-box。** 用我们的 fork:

- 仓库:https://github.com/SwellProxy/sing-box
- 默认分支:`swell_v1.13.14`(基于官方 tag `v1.13.14`)
- 引入方式:**SwiftPM binaryTarget**
  - URL:`https://github.com/SwellProxy/sing-box/releases/download/1.13.14-swell.1/Libbox.xcframework.zip`
  - 版本:`1.13.14-swell.1`
- 产物由 fork 里的 `.github/workflows/build-libbox.yml` 自动构建(macOS runner + gomobile)

### fork 相对官方的唯一改动

`common/urltest/urltest.go` 的 `URLTest()`:复用同一条连接再请求一次,取**热 RTT**(排除 TCP/TLS/代理握手)。函数签名未变,无需改任何调用方。

**为什么:** 官方测的是冷启动全链路,数字比 Karing/mihomo 高 2–4 倍,用户会以为我们慢。

### 升级内核的流程

```bash
git checkout -b swell_v1.13.15 v1.13.15
git cherry-pick <urltest 补丁 commit>
git push -u origin swell_v1.13.15
# 推 tag swell_v1.13.15 → Action 自动出 xcframework 并发 Release
```

---

## 3. 🔴 内核红线(做不到的事,别承诺)

这些是**已核实的 sing-box 事实**,不是保守估计。UI 文案已按此校准过,**不要"优化"回去**。

| 想做的 | 现实 |
|---|---|
| **iOS 按 App 分流** | ⛔ **做不到**。`process_name`/`process_path` 仅 Linux/Windows/macOS;`package_name` 仅 Android。iOS 只能按域名/IP。**所有文案用「网站与服务」,禁止用「应用」** |
| 自动挑"能解锁 Netflix"的线路 | ⛔ 内核只测延迟。**v1 不承诺解锁**,UI 里不得出现「解锁」字样 |
| 自动挑"带宽最大" | ⛔ 只有延迟,没有带宽概念。所以叫「自动」不叫「最快」 |
| 又要自动选最快、又要 IP 不变 | ⛔ **互斥**。🔒 = 锁定单条(IP 不变);🔄 = 自动(会切换,IP 会变)。这套图标语言全局一致 |
| 内核知道线路在哪个国家 | ⛔ **不知道**。地区靠解析线路名,必然有「未识别地区」,兜底入口不能删 |
| 内核有"倍率"字段 | ⛔ **没有**。只能解析线路名 |
| 屏蔽视频贴片广告 | ⛔ 只能按域名拦,YouTube 广告与正片同域 |
| **未连接时测速** | ⚠️ v1 **测速会自动连接**(libbox 架构:core 跑在 NE 里,App 通过 CommandClient 通信)。所以未连接态的延迟必须标「· N 分钟前测」 |
| 链式代理 | ⚠️ v1 **不做** |

### iOS 平台约束

- **没有可靠的后台定时任务。** `BGAppRefreshTask` 由系统调度、不保证准时、用户可关闭。订阅更新实际发生在**用户打开 App 时**。文案必须是「约每 24 小时更新一次,**或在你打开 App 时**」
- **Network Extension 内存上限约 50MB。** 所以 iOS 构建带 `with_low_memory` 标记。**urltest 组必须按地区复用**,多个服务选同一地区时共用一个组,不要 per-service 建组
- **iOS 只允许一个 packet tunnel。** 启动会踢掉其他 VPN

---

## 4. 🔴 订阅解析规则(详见 `specs/订阅解析规格-给开发.md`)

已用两家真实机场订阅验证过。以下每条都有实测依据:

1. **两种格式都要支持:** 完整 sing-box config JSON / base64 分享链接列表。**别信 Content-Type**(有机场返回 `text/html` 但内容是 base64)
2. **格式 A 只取 `outbounds`,其余整份丢弃。** 机场会连 DNS/route/TUN 一起给你(包括 `{"protocol":"quic","action":"reject"}`),用了就是让机场绑架用户的分流设置
3. **必须过滤 info 条。** 机场把广告和账户信息塞成假节点(`🇭🇰 官网:xxx.com`、`剩余流量:990.38 GB`)。其中 **server = `127.0.0.1` 的会在 urltest 里秒回极低延迟 → 被选成"最快节点"** → 用户连上一条广告。这是 bug 不是优化
4. **绝不用 geoip 查 server 地址判断地区。** 实测:一家机场八个国家的节点共用同一个新加坡入口;另一家 28 条节点全挂在两个域名上。**入口 ≠ 出口,这是系统性错误**
5. **倍率两种写法都要吃:** `[5.0倍消耗]` 和 `0.1x`。解析不到或 =1x → 不显示
6. **`subscription-userinfo` 是约定不是标准。** 缺失时可从 info 条名里解析流量/到期作为 fallback;再拿不到 → 订阅卡走精简态(整块隐藏)

---

## 5. UI 硬规则(容易被"优化"掉的)

| 规则 | 原因 |
|---|---|
| 倍率 **> 1x 用警告色,< 1x 用成功色(绿)** | `0.5x`/`0.1x` 是**省流量**,是卖点。标黄等于说"这条有问题",方向反了 |
| 显示名要**剥掉**倍率标记 | 否则第一行 `[5.0倍消耗]` 和第二行 `5x 倍率` 重复。剥不掉则原样显示 |
| 「服务例外」不是「服务分流」,**没有「全部 N 项」** | 心智是**例外清单**不是配置清单。智能分流已处理好一切 |
| 默认预设**只有 5 条**:Claude / ChatGPT / Gemini / X / TikTok,**全部 🔒 锁定** | 门槛是"不预设就会出问题"(AI 与 X 的 IP 风控、TikTok 的内容池),不是"常用" |
| 推荐库里**不得出现浏览器和国内 App** | Chrome 不是分流对象,iOS 上不存在「Chrome 的流量」 |
| 流量数字必须**绑定时间戳** | 那是机场服务端的账,有延迟,本地跑的流量不会实时反映 |
| **优先级链(唯一,勿各说各的):** | `场景规则 > 用户自定义 > 拦截 > 服务例外 > 地区/兜底` |
| 国际化:**用具体国名,不用「国内/国外」** | 「国内」假设用户人在该国。回国线路场景就错了 |
| 界面语言 ≠ 分流国家 | 两个独立设置 |

---

## 6. 参考文档

| 文件 | 内容 |
|---|---|
| `specs/内核能力与修改需求-v1.md` | 内核能匹配什么/能去哪/做不到什么(设计边界) |
| `specs/第二轮复核-v2.md` | 设计评审记录 |
| `specs/第三轮复核与封板-v3.md` | 封板前的问题与状态清单 |
| `specs/订阅解析规格-给开发.md` | **实现订阅解析前必读** |
| `specs/机场接入规范-v1.md` | 给机场的对外规范(`/swell/v1/meta`) |

---

## 7. 许可证

sing-box 是 **GPL-3.0**,Libbox 链接进 App 即产生 GPL 义务 → **本仓库保持公开**。

---

## 8. 给 Claude Code 的建议

- 动 UI 前先在浏览器打开 `design/Swell-Prototype.dc.html` 点一遍,别只看截图
- 遇到设计稿没画的状态,**先问,不要自己编**——设计已封板,原型里应该有
- 上面 §3 的红线如果和某个需求冲突,**红线优先**,并指出冲突
- 内核相关改动去 `SwellProxy/sing-box` fork,不要在 App 里 hack
