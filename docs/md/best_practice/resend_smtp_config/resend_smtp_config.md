

> 使用 Resend + 自定义子域名 `mail.caijy.top` 配置 SMTP 邮件发送服务，适用于需要同时面向国内和海外用户的场景。

### 为什么选择 Resend

| 对比项 | Resend | 阿里云邮件推送 | Mailgun |
|--------|--------|---------------|---------|
| 国内送达率 | 好 | 优秀 | 一般 |
| 海外送达率 | 优秀 | 一般 | 优秀 |
| 免费额度 | 100封/天 | 200封/天（试用） | 100封/天（限1月） |
| 配置难度 | 简单 | 中等 | 中等 |

Resend 在国内外送达率上表现均衡，免费额度对初创项目足够，且配置流程简洁。

### 第一步：注册 Resend 并添加域名

1. 访问 [resend.com](https://resend.com) 注册账号
2. 进入 **Domains** → **Add Domain**
3. 输入 `mail.caijy.top`，选择区域（推荐 `ap-northeast-1` 亚太区域，离国内近延迟低）

### 第二步：配置 DNS 记录

Resend 会生成一组 DNS 记录，到域名 DNS 管理面板添加：

| 类型 | 主机记录 | 值 | 优先级 |
|------|----------|-----|--------|
| TXT | `resend._domainkey.mail` | `p=MIGfMA0GCSqGSIb3DQEBA...（Resend提供的DKIM公钥）` | - |
| MX | `send.mail` | `feedback-smtp.ap-northeast-1.amazonses.com` | 10 |
| TXT | `send.mail` | `v=spf1 include:amazonses.com ~all` | - |
| TXT | `_dmarc` | `v=DMARC1; p=none;` | - |

**注意事项：**

- 如果 DNS 面板会自动追加根域名（如 `.caijy.top`），主机记录只填前缀部分
- 如果 DNS 面板要求填完整域名，则分别填写：
  - `resend._domainkey.mail.caijy.top`
  - `send.mail.caijy.top`
  - `_dmarc.caijy.top`
- DKIM 的 TXT 值较长，部分 DNS 面板需要用双引号包裹
- DMARC 记录是根域名级别（`_dmarc`），对整个 `caijy.top` 生效

### 第三步：验证域名

DNS 记录添加完成后，回到 Resend 后台点击 **Verify**。

- 通常几分钟内验证通过
- 如果超过 24 小时未通过，检查 DNS 记录是否正确生效（可用 `dig` 命令验证）

```bash
# 验证 DKIM 记录
dig TXT resend._domainkey.mail.caijy.top

# 验证 SPF 记录
dig TXT send.mail.caijy.top

# 验证 MX 记录
dig MX send.mail.caijy.top
```

### 第四步：生成 API Key

Resend 后台 → **API Keys** → **Create API Key**

- 命名建议：`cc-api-smtp`
- 权限选择：`Sending access`
- 限制域名：`mail.caijy.top`
- 复制保存 API Key（格式为 `re_xxxxxxxx`，仅显示一次）

### 第五步：配置应用 SMTP

将以下信息填入你的应用 SMTP 设置：

| 配置项 | 值 |
|--------|-----|
| SMTP 服务器 | `smtp.resend.com` |
| SMTP 端口 | `465`（SSL） |
| SMTP 账号 | `resend` |
| SMTP 密码 | `re_xxxxxxxx`（你的 API Key） |
| 发件人邮箱 | `noreply@mail.caijy.top` |

### 第六步：测试验证

配置完成后发送测试邮件，建议测试以下邮箱：

- QQ 邮箱（国内主流）
- 163 邮箱（国内主流）
- Gmail（海外主流）
- Outlook（海外主流）

**检查项：**
1. 邮件能否正常收到
2. 是否进入垃圾箱
3. 发件人是否显示为 `noreply@mail.caijy.top`
4. 送达延迟是否可接受（正常 3-10 秒）

### 常见问题

**Q：邮件进了垃圾箱怎么办？**

检查 DNS 记录是否全部验证通过，尤其是 DMARC 和 SPF。可以用 [mail-tester.com](https://www.mail-tester.com) 检测邮件评分。

**Q：国内用户收不到邮件？**

确认选择了亚太区域（`ap-northeast-1`），如果用的美东区域可能会被部分国内邮箱拦截。

**Q：免费额度不够用了怎么办？**

Resend 付费计划 $20/月 = 50000 封，对于中小规模 API 服务绰绰有余。也可以考虑切换到阿里云邮件推送作为备选。
