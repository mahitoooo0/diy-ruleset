# Sing-box eBPF 代理项目交接文档

> 本文件是项目的完整上下文交接文档，换电脑/新会话时，让 AI 读取本文件即可完全恢复项目状态。
> 配合 `singbox/config.json`（配置模板）一起使用。

---

## 一、项目概述

- **设备**：一加 Ace5 至尊（联发科天玑 9400+），已 root，Android
- **代理 App**：BoxProxy（`com.boxproxy.box`），使用 CHIZI-0618/sing-box **testing-ebpf 分支**（魔改 eBPF 版，schema 来自 `https://raw.githubusercontent.com/CHIZI-0618/sing-box/testing-ebpf/docs/schema.json`）
- **工作模式**：eBPF 透明代理（非 TUN）+ evaluate DNS
- **机场**：train.suuwu.de（anytls 协议）

## 二、核心配置架构（singbox/config.json）

### DNS
- **evaluate 模式**（非 fakeip）：先通过 dns-proxy 查询，响应 IP 命中 ChinaIP/内网则改走 dns-direct 重查
- **分组**：
  - `dns-proxy` = cloudflare(1.1.1.1) + google(8.8.8.8)，走 `📡 DNS出站` 代理
  - `dns-direct` = tencent(doh.pub) + ali(dns.alidns.com)，直连
- `strategy: ipv4_only`（强制 IPv4，避免代理节点 IPv6 回程问题）
- SVCB/HTTPS 查询类型 → NOTIMP 阻断（防 ECH 绕过）
- `evaluate` 响应匹配：`ChinaIP` + `ip_is_private` → 改走 dns-direct
- `final: dns-direct`

### eBPF inbound
- `dns_mode: hijack`
- `bypass_rule_set: ["ChinaIP"]`（国内 IP 内核层旁路，性能最优）
- `shared_network.include_interface`: wlan0, wlan1, rmnet_data0, rmnet_data1
- 已删除冗余字段：network/udp_timeout/cgroup/exclude_package/map_capacity（用默认值）

### 路由
- QUIC UDP 443 → reject（强制回落 TCP）
- IP 复用：resolve 后按 telegram_ip/google_ip/twitter_ip 二次匹配
- per-service 分流：AI/Google/Telegram/YouTube/Twitter/Facebook/GitHub/微软/苹果/抖音/哔哩哔哩/GoogleFCM
- 广告：`reject`（仓库构建）+ `webrtc` 规则
- `final: 🌐 漏网之鱼`

### 其他
- NTP 同步（ntp.aliyun.com，1d）
- `urltest_unified_delay: true`
- 所有 selector `interrupt_exist_connections: true`
- direct 出站 `tcp_keep_alive`
- provider P1 更新 12h；rule_set 更新 **3h**

## 三、仓库与备份位置

| 位置 | 内容 | 说明 |
|------|------|------|
| `shuaiyuanj-netizen/diy-ruleset` main 分支 | `singbox/config.json` | **配置模板备份**（占位符版，22011 字节） |
| 同仓库 `docs/HANDOFF.md` | 本交接文档 | 项目上下文 |
| 同仓库 publish 分支 | `singbox/{tag}.srs` | 规则集产物，Actions 每 3 小时自动重建 |
| 本地 `Downloads\config-MoooooO.json` | 配置模板母版 | 改配置用这个 |
| 本地 `Documents\config-MO.json` | 工作版 | 与母版同步 |
| 手机 BoxProxy | 配置（**真实订阅链接**） | 日常使用 |

### 规则集源
```
https://v6.gh-proxy.org/https://raw.githubusercontent.com/shuaiyuanj-netizen/diy-ruleset/publish/singbox/{tag}.srs
```
标签：`reject, webrtc, google_ip, twitter_ip, telegram_ip, private_ip, ai`（update_interval 3h）

其他源：Repcz（Ads_AWAvenue/AppleCN/ChinaDomain/ChinaIP/Proxy）、MetaCubeX geosite（google/youtube 等 13 个）

## 四、Token 安全铁律（重要）

1. **GitHub/本地 = 占位符模板**，真实订阅链接只存在于手机 BoxProxy
2. 推 GitHub 前必须确认：`token=__REPLACE_WITH_NEW_P1_TOKEN__` 还是占位符
3. **手机上的真实配置永不推 GitHub**
4. token 唯一被覆盖的场景：手动把占位符模板导入手机
5. 若误推真实 token：立即去机场网站重置订阅 token

## 五、更新配置流程

1. 用本地占位符版 `config-MoooooO.json` 修改
2. 推 GitHub 备份（`singbox/config.json`）
3. 手机 BoxProxy 只改所需部分，token 保持原样
4. 改完配置建议删手机 `cache.db` 再重启

## 六、gh CLI 配置（本机）

- 路径：`C:\Program Files\GitHub CLI\gh.exe`（**不在 PATH，需全路径调用**）
- 登录账号：shuaiyuanj-netizen，token scopes：gist, read:org, repo, workflow（存 Windows 凭据管理器，跨会话持久）
- **网络**：`api.github.com` 可直连；`github.com` 需走系统代理 `http://127.0.0.1:7897`（设置 `HTTPS_PROXY`/`HTTP_PROXY` 环境变量）
- 常用操作：`gh api repos/shuaiyuanj-netizen/diy-ruleset/contents/{path}`
- 推送文件用 `-X PUT` + `--input body.json`，body 含 `{message, content(base64), sha}`，**JSON 文件必须无 BOM**（用 `[System.IO.File]::WriteAllText($p,$body,(New-Object System.Text.UTF8Encoding $false))`）

## 七、需要替换的占位符（部署到手机时）

- `__REPLACE_WITH_NEW_CLASH_API_SECRET__`（clash_api 面板密钥）
- `__REPLACE_WITH_NEW_SERVICES_API_SECRET__`（services API 密钥）
- `__REPLACE_WITH_NEW_P1_TOKEN__`（机场订阅 token）

## 八、踩坑记录（排错经验）

1. **download_detour**：魔改版不支持（rule_set 层和 route 层都报 unknown field），删掉即可
2. **不存在的 rule_set tag**：Repcz 的 AI/Apple/Game/Google 等扩展 tag 不存在；MetaCubeX geosite 无 cn-lite/cn/private；samqvz 的 cnip 404；DustinWin 的 privateip/telegramip 下载失败 → 全部改用自建仓库 publish 分支
3. **gh-proxy.com 不通**：改用 `v6.gh-proxy.org`（实测 200）
4. **X-Shelby geoip release CDN 被墙**（重定向到 release-assets.githubusercontent.com）：改用自建仓库的 cn_ip
5. **ccmni0 接口**：非 Ethernet 类，eBPF 报错，不能加进 include_interface
6. **IPv6 问题**：Claude 走 IPv4 正常、ChatGPT/Gemini 走 IPv6 不通 → 改 `strategy: ipv4_only` 解决
7. **切配置后**：删 `cache.db` 避免旧缓存冲突
8. **AI 规则**：MetaCubeX `category-ai-!cn` 覆盖不全，用仓库合并的 `ai.srs`（含 OpenAI/Gemini/Claude）
9. **DNS 泄露**：Chrome DoH 走 443 绕过 53 劫持 → 阻断 dns.google 域名 + 8.8.8.8 IP + 853 端口；国内运营商 DNS 泄露来自 eBPF ChinaIP bypass（可接受，或改 private_ip / 加 iptables）
10. **JSON 校验**：PowerShell 读 UTF-8 用 `[System.IO.File]::ReadAllText($p,[System.Text.Encoding]::UTF8)`，否则中文会乱码误报

## 九、仓库自动更新机制

- `run.yml` cron：`0 */3 * * *`（每 3 小时，北京时间约 8:00/11:00/14:00...）
- 规则源（Repcz/MetaCubeX/自建）在手机端 `update_interval: 3h`，到点自动拉新
- 改 run.yml 频率需 workflow 权限（gh token 已有）
- 仓库规则集更新 → 手机每 3h 自动拉取，无需手动操作

## 十、完整话题历史记录（本次会话全部内容）

> 本会话从"config-MO.json 有什么优化"开始，到配置融合、仓库化、备份结束。按时间顺序记录每个话题的结论。

### 1. 初始配置分析（config-MO.json）
- 用户：魔改 sing-box eBPF，一加 Ace5 至尊 root
- 分析发现：map_capacity/cgroup/exclude_package 冗余、DNS timeout 3s、provider 24h 等
- 首批优化：map_capacity 131072、store_rdrc false、DNS timeout 1s、provider 12h、include_interface 加 rmnet_data0/1

### 2. download_detour 报错
- 添加 `download_detour: 📡 DNS出站` 后报 unknown field
- **结论**：CHIZI-0618 魔改版不支持 download_detour（rule_set 层和 route 层都不支持），删除即可

### 3. DNS 泄露问题（browserleaks.com/dns 多次测试）
- 首次泄露：Chinanet/Tianjij/BJTEL（国内运营商 DNS）+ Google LLC（Chrome DoH/Google DNS）
- 添加了 3 层阻断：DoT 853 reject、dns.google 域名 reject、8.8.8.8/8.8.4.4/IPv6 reject
- **重要发现**：Google LLC 的 172.253.x/74.125.x 是代理节点（GoMami HK）的上游 DNS，**不是用户泄露**（误报）
- 真泄露来源：国内 DNS 被 eBPF `bypass_rule_set: ChinaIP` 内核旁路，53 劫持管不到
- 修复方案：阻断 dns.google + 8.8.8.8 + 853（已做）；彻底消除需改 bypass 或 iptables（用户选择接受）

### 4. tencent DNS detour 导致超时
- 给 tencent DoH 加 `detour: 📡 DNS出站` 后，ipip.net 获取公网 IP 超时
- **结论**：doh.pub 是国内服务器，不需要代理，还原 detour

### 5. 五个配置对比
| 配置 | 特点 |
|------|------|
| 用户 config-MO | fakeip + 完整 per-service + 地区 urltest |
| CHIZI-0618 官方 gist | 最简，IP 直连 DoH，无 per-service |
| D:\ config_ebpf_redacted | **evaluate DNS + cnip/privateip bypass + IP 复用，最先进** |
| singbox-ebpf.json | evaluate + ghost-proxy + ipv4_only |
| config_ebpf.json | evaluate + Repcz 统一规则 |

### 6. 配置融合（最终架构）
- **DNS：fakeip → evaluate**（先查后判，响应 IP 命中 CN 走直连）
- DNS group 分组：dns-proxy（CF+Google）/ dns-direct（tencent+ali）
- CF/Google DoH 改 IP 直连（1.1.1.1/8.8.8.8），不依赖 hosts
- SVCB/HTTPS → NOTIMP 防 ECH 绕过
- QUIC UDP 443 → reject 强制回落 TCP
- resolve 后 IP 复用（telegram_ip/google_ip/twitter_ip）
- NTP 同步、urltest_unified_delay、interrupt_exist_connections、tcp_keep_alive

### 7. eBPF inbound 精简
- 删除 7 个冗余字段：network/udp_timeout/cgroup_enabled/cgroup_path/exclude_package/map_capacity×4
- bypass_rule_set 最终定为 `["ChinaIP"]`

### 8. rule_set 源问题排查（大量踩坑）
- samqvz cnip → 404
- X-Shelby geoip-cn → release CDN 被墙
- DustinWin privateip/telegramip → 下载失败
- Repcz 扩展 tag（AI/Apple/Game/Google 等）→ 不存在
- MetaCubeX geosite cn-lite/cn/private → 不存在
- **解决**：用用户自建仓库 diy-ruleset publish 分支解决全部

### 9. diy-ruleset 仓库方案
- 用户 fork samqvz/diy-ruleset（规则构建引擎），Actions 自动从上游拉取+去重+输出 singbox 格式
- publish 分支产物：reject/webrtc/google_ip/twitter_ip/telegram_ip/private_ip/ai/cn 等
- 解决：cnip→cn_ip、privateip→private_ip、telegramip→telegram_ip
- 链接：`https://v6.gh-proxy.org/https://raw.githubusercontent.com/shuaiyuanj-netizen/diy-ruleset/publish/singbox/{tag}.srs`

### 10. AI 服务不可用（Gemini/ChatGPT）
- 现象：Claude 能对话，GPT/Gemini 不行
- 日志分析：Claude 走 IPv4（160.79.104.10）正常，GPT/Gemini 走 IPv6（Cloudflare/Google IPv6）不通
- 修复1：`category-ai-!cn` → 仓库 `ai` 规则集（更全，含 OpenAI/Gemini/Claude）
- 修复2：DNS `strategy: prefer_ipv4 → ipv4_only`（强制 IPv4，避开节点 IPv6 回程问题）
- 用户确认：机场支持 IPv6（有 IPv6 连接能通），问题在特定 IPv6 路由/节点 IP 被风控

### 11. 仓库自动更新机制
- run.yml cron 原为每天一次 → 改为 `0 */3 * * *`（每 3 小时）
- 手机 rule_set `update_interval: 24h → 3h`（对齐）
- 结论：仓库更新频率和手机拉取频率是两条独立链路，需两端对齐
- 权衡：手机 3h 拉取每天约 16-24MB 流量，用户选择 3h（流量多）

### 12. gh CLI 授权与操作
- 本机 gh 不在 PATH，在 `C:\Program Files\GitHub CLI\gh.exe`
- token 缺 workflow scope → 无法改 workflow 文件
- 系统代理 127.0.0.1:7897 可到 github.com（直连不通）
- 用户授权设备码 E3B2-2428 后获得 workflow 权限
- **凭证存 Windows 凭据管理器，跨会话/跨话题持久**，新话题不用重新授权

### 13. 配置备份到 GitHub
- 推到 `diy-ruleset/singbox/config.json`（占位符版，无真实 token）
- 用途：灾难恢复、版本回滚、跨设备部署

### 14. Token 安全与公开仓库
- 铁律：GitHub/本地=占位符模板，真实订阅链接只在手机
- 公开仓库内容安全（无真实密钥）
- **隐私加固**：GitHub 备份的机场 URL（train.suuwu.de）已改空白 `""`（commit ef52665），本地母版保留

### 15. 交接文档
- 本 HANDOFF.md 创建并推到 `docs/HANDOFF.md`

## 十二、自动备份机制（已配置，无需手动）

> 已设置 Windows 定时任务 **`singbox-auto-backup`**，每 4 小时自动把本地母版同步到 GitHub。

### 原理
- 脚本：`C:\Users\jsy09\Documents\sync-backup.ps1`
- 定时任务：每 4 小时执行一次（Windows 任务计划程序）
- 日志：`C:\Users\jsy09\Documents\backup-log.txt`
- 检测无变化会自动跳过（不产生多余提交）

### 备份内容
1. `Downloads\config-MoooooO.json` → GitHub `singbox/config.json`
   - **自动抹掉机场 URL**（改成 `""`）和所有 `token=`（改成 REDACTED），隐私安全
2. `Downloads\HANDOFF.md` → GitHub `docs/HANDOFF.md`

### 你要做的
- **什么都不用做**，聊完忘掉也没关系，每 4 小时自动备份一次
- 想看备份是否成功：打开 `backup-log.txt`，看到 `PUSHED` 就是推上去了，`SKIP(unchanged)` 表示没变化
- 手动立即备份：右键 `sync-backup.ps1` → 用 PowerShell 运行

### 新电脑重配自动备份
```powershell
# 复制 sync-backup.ps1 到新电脑对应路径，然后：
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File `"C:\Users\jsy09\Documents\sync-backup.ps1`""
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date).AddMinutes(2) -RepetitionInterval (New-TimeSpan -Hours 4) -RepetitionDuration (New-TimeSpan -Days 3650)
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -StartWhenAvailable
Register-ScheduledTask -TaskName "singbox-auto-backup" -Action $action -Trigger $trigger -Settings $settings -Description "自动备份 sing-box 配置到 GitHub" -Force
```

## 十三、给新会话的启动指令

> "读取 GitHub 仓库 shuaiyuanj-netizen/diy-ruleset 的 docs/HANDOFF.md 和 singbox/config.json，这是 sing-box eBPF 代理项目，继续维护。gh CLI 在 C:\Program Files\GitHub CLI\gh.exe，github.com 走代理 127.0.0.1:7897。"
