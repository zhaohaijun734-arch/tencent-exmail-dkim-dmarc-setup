---
name: tencent-exmail-dkim-dmarc-setup
description: 为腾讯企业邮箱 / 企业微信邮箱配置 DKIM + DMARC 的完整流程（DNS 托管在 DNSPod 时）。当用户要给企业邮箱域名补邮件认证、提升送达率、避免进垃圾箱，或提到 DKIM / DMARC / SPF / 发信认证 / 邮件被拒收 / mail-tester 时使用。含非直觉的入口路径与 5 个高频踩坑点。
agent_created: true
---

# 腾讯企业邮箱 DKIM + DMARC 配置

## 适用场景

- 腾讯企业邮箱（含企业微信邮箱版）需要补 DKIM / DMARC
- DNS 托管在 DNSPod / 腾讯云
- 目标：邮件不被 Gmail / Yahoo / Outlook 判为垃圾。2024 年起 Gmail 对批量发信强制要求「SPF 或 DKIM 至少一项 + 一条 DMARC 记录」

## 前置检查

```bash
dig +short NS <domain>
dig +short TXT <domain>
dig +short TXT _dmarc.<domain>
```

- NS 显示 `*.dnspod.net` → DNS 托管在 DNSPod，本流程适用
- 腾讯 SPF 标准值：`v=spf1 include:spf.mail.qq.com ~all`
- 腾讯 MX：`mxbiz1.qq.com`(5) / `mxbiz2.qq.com`(10)

## 步骤

### 1. 腾讯后台生成 DKIM —— 入口位置是关键

```
企业微信管理后台 → 协作 → 邮件 → 【安全管理】标签 → 「DKIM 验证」卡片 → 「前往」/「配置」
```

入口**不在**「邮箱域名」标签下（那里只有域名列表 + 域名指向信息）。

弹窗给出三项：
- 记录类型 `TXT`
- 主机记录 `<random>._domainkey`（如 `trym2609._domainkey`，腾讯企业微信版用**随机 selector**）
- 记录值 `v=DKIM1; k=rsa; p=<长 base64>`

### 2. DNSPod 添加记录

入口：`console.dnspod.cn` 或 腾讯云控制台搜索「DNS 解析 DNSPod」→ 我的域名 → 点域名 → 记录管理 → 「添加记录」

| 记录 | 主机记录 | 类型 | 记录值 |
|---|---|---|---|
| DKIM | `<random>._domainkey` | TXT | `v=DKIM1; k=rsa; p=...` |
| DMARC | `_dmarc` | TXT | `v=DMARC1; p=none; rua=mailto:<报告邮箱>; fo=1` |

线路类型默认、TTL 默认即可。

### 3. 回腾讯后台验证

DKIM 弹窗点「已完成配置，立即验证」→ 状态变「已验证」。失败等 10 分钟再点（DNS 传播延迟属正常）。

### 4. 独立复核（不要只信后台显示）

```bash
for r in 8.8.8.8 1.1.1.1; do dig +short TXT <random>._domainkey.<domain> @$r; done
dig +short TXT _dmarc.<domain> @8.8.8.8
```

三处解析器返回值一致 = 全球生效。DKIM 值应以 `IDAQAB` 结尾（完整 RSA 公钥的 base64 特征）。

### 5. 端到端验证（推荐 mail-tester.com）

1. 打开 `https://www.mail-tester.com/` → 页面给一个临时收件地址
2. 从企业邮箱发一封**真实内容**的邮件过去
3. 回页面点 check → 得 0-10 分 + SPF / DKIM / DMARC / 黑名单 / 内容逐项报告，满分 10

## 踩坑清单（全部真实遇到）

| # | 坑 | 正解 |
|---|---|---|
| 1 | 在「邮箱域名」标签下翻遍找不到 DKIM | 入口在 **「安全管理」** 标签（标签栏第 5 个），页内卡片叫「DKIM 验证」 |
| 2 | 猜常见 selector（default / dkim / s1 / tencent …）去 dig，全部落空 | 腾讯企业微信版用**随机 selector**，必须去后台看生成的主机记录 |
| 3 | 打开腾讯云**轻量应用服务器**的「域名解析」页当作 DNS 管理 | 那里只能加 A 记录（域名→IP）；加 TXT 必须去 **DNSPod** |
| 4 | 主机记录填成 `xxx._domainkey.<domain>` | 只填前缀，DNSPod 自动补后缀，否则变成 `...<domain>.<domain>` 永不生效 |
| 5 | 按屏幕显示把 DKIM 记录值分多行填 / 手工抄写 | 必须**单行**；长 base64 一律用页面「复制」按钮，绝不手抄或 OCR 转录 |

## 注意事项

- DKIM 弹窗里记录值**显示成 2-3 行只是排版**，不是真实值的一部分
- 私钥由腾讯保管，用户只发布公钥，无需自行生成密钥对
- `p=none` 先只观察不拦截，跑 2 周看 DMARC 报告再考虑升级 `quarantine`
- `rua=` 建议用独立/公共邮箱收报告（报告量可能很大），别塞进正在使用的业务邮箱
- 免费版即使页面提示「开通高级功能」，实战验证 **DKIM 仍可用**，不必升级

## 参数建议

- 起步：`v=DMARC1; p=none; rua=mailto:<报告邮箱>; fo=1`（`fo=1` = 任一机制失败即发报告，便于排障）
- 升级路径：`p=none` → 观察 2 周 → `p=quarantine` → 成熟后 `p=reject`

## 后续：读 DMARC 聚合报告（用户一定会回来问「Google 发的这些邮件是什么」）

配置完 DMARC 后，收件邮箱会定期收到 `noreply-dmarc-support@google.com` 的邮件，主题形如
`Report domain: <你的域名> Submitter: google.com Report-ID: <19位数字>`。
**这不是验证码、不是告警、也不是被黑** —— 是 Google（Gmail）把当天收到的、声称来自你域名的邮件做了一次统计回执，俗称 DMARC 聚合报告。

### 报告的实际结构

- **正文是空的**（没有 text/plain，也没有有效 HTML）→ 用 `read_mail_body.py` 会读到空白，**别以为邮件坏了**
- 真正内容在**附件**：`google.com!<域名>!<起始时间戳>!<结束时间戳>.zip`（约 700 B）
- zip 里只有一个 XML，字段：
  - `report_metadata`：org_name / report_id / **date_range**（起止 Unix 时间戳，UTC 全天）
  - `policy_published`：你 DNS 上那条记录的回显（`p=none` / `aspf=r` / `pct=100`）
  - `record[]` × N：每条 = 一个来源 IP 的汇总
    - `row/source_ip`、`row/count`（邮件数）
    - `auth_results/spf/result`、`auth_results/dkim/result`（原始校验结果）
    - `row/policy_evaluated/{spf,dkim,disposition}` → **DMARC 最终判定**，看这个最直观
    - `identifiers/header_from`（信头里的发件域）

### 判读口诀

- `disposition=none` → 放行；`quarantine` → 进垃圾箱；`reject` → 拒收
- `spf=pass dkim=pass` → 正常。**两个都 fail 且 IP 不认识 → 有人在冒用你的域名发信，重点排查**
- `count` 长期为 0 或个位数属正常（只有发到 Gmail/Workspace 的邮件才计入）
- 来源 IP 记得反查 PTR：腾讯企业邮箱的海外发信节点 PTR 是 `smtpbg*.qq.com`（AWS IP 段），看到这个就是**自己的正常外发**，不用紧张

### 解析时的三个坑（都会让你的脚本直接崩）

1. **腾讯 exmail 的 IMAP `SEARCH` 全部失效** —— 不只是 `SINCE`，`FROM "xxx"` 也照样返回收件箱全部邮件。必须**拉全量再在客户端按发件地址做字符串过滤**
2. **`m.get("From")` 可能返回 `email.header.Header` 对象**，直接对它用 `in` 会抛 `TypeError: argument of type 'Header' is not iterable`。解码函数里先 `isinstance(v, email.header.Header)` 转 `str`
3. 头部编码按 `enc → utf-8 → gb18030 → latin-1` 逐级 fallback，直接 `t.decode(enc)` 会踩 `LookupError: unknown encoding: unknown-8bit`

### 重复投递（正常现象，别以为被邮件轰炸）

同一天的报告可能一次来 **4 封**完全相同的：Message-ID 一模一样，但 `Received` 路径不同 ——
Google 侧不同的发送主机（mail-qv1-f73 / -f74 / mail-vk1-f201）分别打进了腾讯不同的入信节点
（bizmx52 / 60 / 68 / 65），腾讯入库未去重导致。**去重办法：按 `report_id` 集合去重后再解析**。
