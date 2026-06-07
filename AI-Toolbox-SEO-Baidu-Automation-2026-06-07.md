# AI Toolbox SEO、站长平台验证与百度主动推送自动化总结

日期：2026-06-06 至 2026-06-07  
项目：[`talang-tech/ai-toolbox`](https://github.com/talang-tech/ai-toolbox)  
站点：<https://tools.talang.fun>

---

## 1. 背景目标

近期围绕 AI Toolbox 做了站点 SEO、搜索引擎收录、百度站长平台验证、Sitemap 提交和主动推送自动化配置。

核心目标：

- 确保 `robots.txt` 和 `sitemap.xml` 支持 Google、Bing、百度等搜索引擎抓取。
- 重新完成百度搜索资源平台站点所有权验证。
- 修复 sitemap 响应头中可能影响收录的配置。
- 建立百度 URL 主动推送能力。
- 通过 GitHub Actions 实现后续自动化提交。
- 强化 AI Toolbox 的核心定位：免费、隐私优先、浏览器本地处理的在线开发者/办公工具箱。

---

## 2. 站点抓取与 Sitemap 检查

检查地址：

```text
https://tools.talang.fun/robots.txt
https://tools.talang.fun/sitemap.xml
```

检查结论：

- `robots.txt` 返回 `200`。
- `sitemap.xml` 返回 `200`。
- `sitemap.xml` XML 可正常解析。
- sitemap URL 数量：`128`。
- sitemap 中 URL 均属于：

```text
https://tools.talang.fun/
```

`robots.txt` 核心配置：

```txt
User-agent: *
Allow: /
Crawl-delay: 1

Sitemap: https://tools.talang.fun/sitemap.xml
```

该配置允许 Googlebot、Bingbot、Baiduspider 等普通搜索爬虫抓取。

同时保留了对部分 AI 训练爬虫的屏蔽，例如：

```txt
User-agent: GPTBot
Disallow: /

User-agent: ClaudeBot
Disallow: /

User-agent: Google-Extended
Disallow: /
```

说明：`Google-Extended` 不等于 Google Search 的 `Googlebot`，屏蔽它不会影响 Google 普通搜索抓取。

---

## 3. 修复 sitemap 的 `X-Robots-Tag: none`

发现 `_headers` 中 sitemap 曾有配置：

```txt
/sitemap.xml
  Content-Type: application/xml; charset=utf-8
  Cache-Control: public, max-age=3600
  X-Robots-Tag: none
```

问题：

- `X-Robots-Tag: none` 等价于 `noindex,nofollow`。
- 虽然它作用在 sitemap 文件本身，不一定阻止 sitemap 内 URL 被抓取，但对 Google Search Console、Bing Webmaster Tools、百度站长平台而言有潜在误判风险。

已修复为：

```txt
/sitemap.xml
  Content-Type: application/xml; charset=utf-8
  Cache-Control: public, max-age=3600
```

对应提交：

```text
4dd6d8c fix: allow sitemap indexing headers
```

验证结果：

- `https://tools.talang.fun/sitemap.xml` 返回 `200`。
- `Content-Type: application/xml; charset=utf-8`。
- 响应头中不再有 `X-Robots-Tag: none`。

---

## 4. 百度站长平台验证过程

### 4.1 文件验证尝试

百度平台提供验证文件：

```text
baidu_verify_codeva-D8uSZqQEEm.html
```

文件内容：

```text
6e4e2479b704552dbe8727e57131d13d
```

已添加到项目根目录并推送。

对应提交：

```text
e1e9ced chore: add Baidu site verification file
```

线上文件访问曾验证成功：

```text
https://tools.talang.fun/baidu_verify_codeva-D8uSZqQEEm.html
```

返回：

```text
6e4e2479b704552dbe8727e57131d13d
```

但百度后台文件验证失败，报错：

```text
未知原因:308
```

原因定位：

Cloudflare Pages 对 `.html` 路径做了跳转：

```text
/baidu_verify_codeva-D8uSZqQEEm.html
→ 308
/baidu_verify_codeva-D8uSZqQEEm
→ 200
```

百度文件验证器不接受或不能正确处理该 308 跳转。

曾尝试通过 `_redirects` 固定验证文件路由：

```text
/baidu_verify_codeva-D8uSZqQEEm.html /baidu_verify_codeva-D8uSZqQEEm.html 200
```

对应提交：

```text
5222039 chore: pin Baidu verification routes
```

但最终更稳定的方案是改用 HTML 标签验证。

### 4.2 HTML 标签验证成功

百度提供 HTML 标签：

```html
<meta name="baidu-site-verification" content="codeva-GefzDJXvPs" />
```

已加入 `build.py` 的全站 `<head>` 模板，并重新构建静态页面。

对应提交：

```text
afc78d4 chore: add Baidu HTML verification meta
f47761a chore: add Baidu meta to English static pages
```

线上首页验证：

```text
https://tools.talang.fun/
```

已能检查到：

```html
<meta name="baidu-site-verification" content="codeva-GefzDJXvPs">
```

百度搜索资源平台最终验证成功。

---

## 5. 百度主动推送 API

百度资源平台提供接口：

```text
http://data.zz.baidu.com/urls?site=https://tools.talang.fun&token=***
```

实际测试发现：

- `site=https://tools.talang.fun` 返回：

```json
{"error":400,"message":"site init fail"}
```

- `site=tools.talang.fun` 返回成功。

因此脚本默认使用：

```text
site=tools.talang.fun
```

### 5.1 新增推送脚本

新增文件：

```text
scripts/baidu_submit.py
```

功能：

- 从 `sitemap.xml` 读取 URL。
- 校验 URL 均属于 `tools.talang.fun`。
- 支持提交指定 URL。
- 支持从文件读取 URL。
- 支持 `--limit` 控制每日推送数量。
- 支持 `--mode zh-first`，优先提交中文页面。
- 支持 `--rotate`，每天轮换不同 URL 批次，避免每天重复提交同一批。
- token 通过环境变量 `BAIDU_PUSH_TOKEN` 注入，避免写入公开仓库。

文档：

```text
docs/BAIDU_SUBMIT.md
```

对应提交：

```text
1c105b1 feat: add Baidu URL submission script
```

### 5.2 已成功推送核心 URL

已用百度 API 成功推送一批核心 URL。

百度返回：

```json
{"remain":0,"success":9}
```

说明：

- 当日剩余额度：`0`
- 成功推送：`9`

另外在测试 API 参数时首页也成功推送过一次，所以当日实际成功推送约 10 条。

已推送核心页面包括：

```text
https://tools.talang.fun/
https://tools.talang.fun/tools/json-formatter
https://tools.talang.fun/tools/base64
https://tools.talang.fun/tools/url-encoder
https://tools.talang.fun/tools/hash-generator
https://tools.talang.fun/tools/password-generator
https://tools.talang.fun/tools/jwt-decoder
https://tools.talang.fun/tools/regex-tester
https://tools.talang.fun/tools/timestamp
https://tools.talang.fun/blog/pdf-tools-privacy-guide
```

---

## 6. GitHub Actions 自动化

新增 GitHub Actions 工作流：

```text
.github/workflows/baidu-submit.yml
```

功能：

- 支持手动运行。
- 支持定时运行。
- 每天北京时间 09:15 自动提交 URL。
- 默认提交 10 条。
- 默认 `zh-first` 模式，优先中文页面。
- 定时任务使用 `--rotate`，每天轮换不同 URL 批次。

对应提交：

```text
0388732 ci: automate Baidu URL submission
```

GitHub Secret 配置要求：

```text
Name: BAIDU_PUSH_TOKEN
Value: 百度主动推送 token
```

配置路径：

```text
GitHub repository → Settings → Secrets and variables → Actions → New repository secret
```

注意事项：

- Secret 中只填 token 值。
- 不要填完整接口 URL。
- 不要带 `token=` 前缀。
- token 不写入前端、不写入 Markdown 页面、不提交到公开仓库。

---

## 7. SEO 与内容资产推进

### 7.1 隐私说明增强

为敏感工具页增加中英双语隐私说明，突出：

- 浏览器本地处理。
- 输入内容、文件、JSON、JWT、图片、PDF、密码、哈希或文本不会上传服务器。
- 只记录基础访问量和转化点击，不追踪用户输入和文件内容。

覆盖工具包括：

- JSON Formatter
- JWT Decoder
- Hash Generator
- Password Generator
- URL Encoder
- Text Dedup
- 图片处理类工具
- PDF 处理类工具

对应提交：

```text
212af15 feat: surface privacy notices on sensitive tools
```

### 7.2 PDF 隐私主题博客

新增 PDF 工具隐私相关博客，用于中文 SEO 长尾内容：

```text
blog/pdf-tools-privacy-guide.html
blog/posts/zh/pdf-tools-privacy-guide.md
blog/posts/en/pdf-tools-privacy-guide.md
```

对应提交：

```text
c415c30 feat: add PDF privacy guide
```

### 7.3 工具脚本稳定性改进

改进部分前端工具脚本：

- `hash-generator.js`：补充纯 JS MD5 实现，修复 Web Crypto 不支持 MD5 的问题。
- `password-generator.js`、`text-dedup.js`、`timestamp.js`、`url-encoder.js`：增加 strict mode 和错误处理。

---

## 8. 已执行验证

执行过的验证包括：

```bash
python3 build.py
python3 scripts/run_tests.py
python3 scripts/seo_analyzer.py
git diff --check
node --check assets/tools/hash-generator.js
node --check assets/tools/password-generator.js
node --check assets/tools/text-dedup.js
node --check assets/tools/timestamp.js
node --check assets/tools/url-encoder.js
```

测试结果：

- 构建成功。
- 生成 128 个 HTML 页面 + sitemap + robots。
- `scripts/run_tests.py`：9/9 通过。
- SEO 分析总体评分：95%。
- SEO / Performance / Best Practices 均为 100%。
- 主要提醒：可访问性中图片 alt 检查提示仍可继续优化。

---

## 9. 后续计划

### P0：确认自动化链路完成

- 在 GitHub 仓库配置 `BAIDU_PUSH_TOKEN` Secret。
- 手动触发一次 `Baidu URL Submit` workflow。
- 如果当天额度已用完，确认日志为 `over quota` 即可，次日自动运行。
- 次日检查百度平台 API 推送记录。

### P1：继续中文 SEO 内容资产

优先新增面向百度中文搜索的长尾内容：

- 在线 JSON 格式化工具为什么要本地处理？
- JWT 在线解码安全吗？如何避免 token 泄露？
- PDF 合并/拆分工具如何做到文件不上传？
- Base64 编解码常见场景与隐私风险。
- 时间戳转换、Cron 解析、正则测试等开发者工具长尾教程。

### P1：继续提交搜索平台

- Google Search Console 提交：

```text
https://tools.talang.fun/sitemap.xml
```

- Bing Webmaster Tools 提交同一 sitemap。
- 百度资源平台保持 sitemap + 主动推送双通道。

### P2：进一步提升收录与转化

- 首页增加更醒目的“隐私优先 / 本地处理”信任区块。
- README 增强中文关键词与工具列表。
- 工具页增加内链和 FAQ。
- 优化图片 alt 或移除误报，提高可访问性评分。
- 增加“提交工具需求 / 定制工具 / 私有化部署”入口的转化跟踪。

---

## 10. 关键提交列表

```text
0388732 ci: automate Baidu URL submission
1c105b1 feat: add Baidu URL submission script
f47761a chore: add Baidu meta to English static pages
afc78d4 chore: add Baidu HTML verification meta
5222039 chore: pin Baidu verification routes
e1e9ced chore: add Baidu site verification file
4dd6d8c fix: allow sitemap indexing headers
c415c30 feat: add PDF privacy guide
212af15 feat: surface privacy notices on sensitive tools
```

---

## 11. 当前状态

截至 2026-06-07：

- 百度站点验证：已成功。
- 百度主动推送：已成功调用，当前日额度已用完。
- 自动化 workflow：已提交到仓库。
- robots.txt：允许普通搜索爬虫抓取。
- sitemap.xml：可访问、可解析、无 `X-Robots-Tag: none`。
- AI Toolbox 中文 SEO 基础设施：基本完成。

下一步重点是持续内容生产、百度/Google/Bing 提交状态观察，以及百度主动推送自动化运行验证。
