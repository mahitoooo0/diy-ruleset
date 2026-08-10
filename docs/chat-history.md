# Sing-box 项目会话记录（2026-08-09 ~ 08-10）

> 本文件记录与用户的实际对话过程、讨论内容和决策思路。
> 配合 `docs/HANDOFF.md`（技术状态）+ `singbox/config.json`（配置模板）使用。
> 新 AI 读取本文件即可理解"我们是怎么一步步走到现在的"。

---

## 会话一：配置优化

**用户**：config-MO.json 有什么优化？魔改 sing-box eBPF，一加 Ace5 至尊 root 能用。

**过程与决策**：
- 分析出 7 个可优化点：eBPF map_capacity、DNS timeout 3s、provider 24h、store_rdrc、冗余规则等
- 用户提到"其他节点里有法国和德国"——确认 exclude 正则正常工作
- 用户问"你改了吗"，随后授权我直接修改文件
- 应用了首批优化（map_capacity 131072、store_rdrc false、timeout 1s、provider 12h、加移动数据接口）
- **用户反馈**：改完出了问题（download_detour 报错、DNS 泄露），反复调整

**教训**：用户对"越改越差"有顾虑，后续改动要克制、要验证。

## 会话二：download_detour 报错

**问题**：加了 `download_detour: "📡 DNS出站"` 后报 unknown field。

**结论**：CHIZI-0618 魔改 eBPF 分支**不支持 download_detour**（rule_set 层和 route 层都报错）。删除即可。

## 会话三：DNS 泄露排查（反复多次）

**用户多次贴 browserleaks 测试结果**，经历：泄露 → 修复 → 又泄露 → "又不漏了"。

**关键认知**（重要，别重复踩坑）：
1. **Google LLC 的 172.253.x/74.125.x/2404:6800** 大部分是**代理节点（GoMami HK）的上游 DNS**，属于**误报**，不是用户泄露
2. **真泄露**是国内运营商 DNS（Chinanet/Tianjij/BJTEL），来自 eBPF `bypass_rule_set` 内核旁路，53 劫持管不到
3. Chrome 内置 DoH（走 443）绕过 53 劫持 → 需阻断 dns.google 域名 + 8.8.8.8 IP + 853 端口
4. 最后用户说"又不漏了"，且**换配置后结论**：很多时候泄露是**网络切换/系统 DNS 缓存**导致的暂时现象，不一定是配置问题

**最终处理**：添加了 dns.google 域名阻断、DoT 853 阻断、8.8.8.8 IP 阻断（在 config-MO.json 里，部署版可能没含）。

## 会话四：五个配置对比与融合

**用户提供 5 个配置**：config-MO（用户的）、CHIZI-0618 官方 gist、D:\config_ebpf_redacted、singbox-ebpf.json、config_ebpf.json。

**对比结论**：
- D:\ config_ebpf_redacted **最先进**（evaluate DNS + cnip/privateip bypass + IP 复用）
- CHIZI 官方最简（IP 直连 DoH）
- 用户 config-MO 有完整 per-service 分流

**融合方案**（用户确认后实施）：
- **fakeip → evaluate DNS**（最大变化）：先查后判，响应 IP 命中 CN 走直连
- DNS group 分组：dns-proxy（CF+Google，走代理）/ dns-direct（tencent+ali，直连）
- CF/Google DoH 改 **IP 直连**（1.1.1.1/8.8.8.8）
- SVCB/HTTPS → NOTIMP 防 ECH
- QUIC UDP 443 → reject
- resolve 后 IP 复用（telegram_ip/google_ip/twitter_ip）
- eBPF inbound 精简 7 个冗余字段
- 用户先问"融合后走的是什么"，我详细讲解了 evaluate 流程后才动手

## 会话五：rule_set 源问题（踩坑最多）

**大量源不可用**，逐一排查：
- samqvz `cnip` → 404
- X-Shelby geoip-cn → release CDN 被墙
- DustinWin privateip/telegramip → 下载失败
- Repcz 扩展 tag（AI/Apple/Game 等）→ 不存在
- MetaCubeX geosite cn-lite/cn/private → 不存在
- gh-proxy.com → 不通，改用 **v6.gh-proxy.org**

**最终方案**：用户自建 `diy-ruleset` 仓库（fork samqvz/diy-ruleset），Actions 自动构建所有规则集到 publish 分支。全部规则源指向自己的仓库，彻底解决 404/被墙问题。

**用户问过**"你能看我的 GitHub 仓库吗"、"链接保证能下载成功吗"——我实测了 raw 和 jsDelivr 都能下，SRS 魔数合法。

## 会话六：AI 服务不可用（Gemini/ChatGPT）

**现象**：Claude 能对话，GPT/Gemini 不行。

**日志分析**（关键）：Claude 走 IPv4（160.79.104.10）正常，GPT/Gemini 走 **IPv6**（Cloudflare/Google）不通。同一个节点，IPv4 通 IPv6 不通。

**修复**：
1. `category-ai-!cn` → 仓库合并的 `ai.srs`（更全）
2. DNS `strategy: prefer_ipv4 → ipv4_only`
3. 后续：AI 出站固定 `🇺🇸 美国节点`（香港 PublicNET 被 OpenAI 封）

**用户确认**：机场支持 IPv6（有 IPv6 连接能通），是特定 IPv6 路由/节点 IP 被风控。

## 会话七：GitHub 仓库搭建

- 用户 fork `samqvz/diy-ruleset`，Actions 每 3 小时重建规则到 publish 分支
- cron 从每天一次改为 `0 */3 * * *`（每 3 小时）
- 手机 rule_set `update_interval` 改为 3h 对齐

**gh CLI 授权**（折腾）：
- gh 不在 PATH，在 `C:\Program Files\GitHub CLI\gh.exe`
- token 缺 workflow scope 无法改 workflow 文件
- 系统代理 127.0.0.1:7897 可到 github.com（直连不通）
- 用户授权设备码后获得 workflow 权限
- **凭证存 Windows 凭据管理器，跨会话/跨话题持久**

## 会话八：配置备份到 GitHub

- 推到 `diy-ruleset/singbox/config.json`（占位符版）
- 用户担心"别人能看见吗"——公开仓库，但占位符无泄露
- 用户要求把机场 URL 改空白 → 已处理（`"url": ""`）
- 创建 `docs/HANDOFF.md` 交接文档

## 会话九：自动备份 + Telegram 通知

**需求**：忘记备份、想自动通知。

**实现**：
- `sync-backup.ps1` 每 4 小时备份（脚本 + schtasks）
- `watch-starred.ps1` 标星仓库监控
- Telegram bot `@mahito963_bot` 通知

**踩坑**：
- PowerShell 5.1 读 UTF-8 无 BOM 文件按 GBK，中文/emoji 会破坏解析 → **脚本必须纯 ASCII**
- `Register-ScheduledTask` 的 `RepetitionDuration 3650天` 触发 Windows 限制，**重复不生效** → 改用 `schtasks /sc HOURLY`（可靠）
- Telegram 通知初始"只发变化/失败"，用户以为坏了 → 改成**每次运行都通知**

## 会话十：用户名变更

- `shuaiyuanj-netizen` → `mahitoooo0`（网页改的）
- 我更新了全部引用：本地配置、脚本、HANDOFF、GitHub 备份
- 用户问"旧电脑还要重新验证吗"——不需要，凭证按账号 ID 存储，跨改名有效

## 会话十一：当前状态与持续话题

- AI 偶尔不可用 → 日志排查 → 多数是节点问题（换美国节点解决）
- 标星监控改为每 10 分钟（接近实时）
- 用户关心：新 AI 能否读取 GitHub 上所有上下文 → 只需读 HANDOFF.md + config.json 两个文件即可

---

## 给用户的提醒

- 新会话时对 AI 说："读取 mahitoooo0/diy-ruleset 的 docs/HANDOFF.md、docs/chat-history.md 和 singbox/config.json"
- 三个文件一起读 = 技术状态 + 对话历程 + 配置模板，完整上下文
