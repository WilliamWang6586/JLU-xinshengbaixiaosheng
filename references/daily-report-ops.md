# 每日科技日报 · 运维要点（校园网直连时代）

> 脚本：`<本地资料库>`（调度）+ `daily_tech_report_v2.py`（生成）
> 调度：**Windows 任务计划程序**（不是 Hermes cron）——`Hermes每日科技日报`，每天 18:30
> 外置知识：记忆外置库 `详细知识\daily_report.md`

## ⚠️ 目录重组后的连带修复清单（2026-10-18 实测）

仓库重组把日报挪进 `03_工具与项目\` 时，光 `Move-Item` 会**静默打断**日报。必查四处：

1. **`daily_tech_report_v2.py` 的 `OUTPUT_ROOT`** = `dirname(__file__)`，跟着脚本走，不用改；但 `assets/` 是相对它的 → 目录里**必须留 `assets/theme_*.jpg`**（5 张主题兜底图），否则每期都要联网重下、漏图就空板块。历史配图（2000+ 张、500MB）可以整批归档到 `_脚本存档\14_日报历史产物\`，只留 `theme_*`。
2. **写死绝对路径的四处**：`daily_pipeline.ps1`（`$script`/`$qqfile`/`$out`）、`push_wechat.py`（`base` + 推文里的路径文案）、`账单识别\final_build.py`/`make_final.py`（`BASE` + 输出 xlsx 路径）。移动前先
   `Select-String -Pattern 'Hermes文件库\\' -SimpleMatch -List` 扫一遍所有 `.py/.ps1/.bat`，命中就改。
3. **Windows 定时任务**：`Get-ScheduledTask -TaskName 'Hermes每日科技日报'` 里 `Actions[0].Arguments` 存的是绝对路径，用
   ```powershell
   $t=Get-ScheduledTask -TaskName "Hermes每日科技日报"; $a=$t.Actions[0]
   $a.Arguments=$a.Arguments.Replace("<旧路径>","<新路径>")
   Set-ScheduledTask -TaskName "Hermes每日科技日报" -Action $a
   ```
   改完 `(Get-ScheduledTask ...).Actions[0].Arguments` 读回确认。**不改就明晚静默不跑**。
4. **验证方式**：`importlib.util.spec_from_file_location` 加载 `daily_tech_report_v2.py`，断言 `OUTPUT_ROOT` 等于脚本所在目录 + `assets` 目录存在 + `THEME_POOL` 里的 `_fetch_theme` 可调用。缺 `theme_*.jpg` **不算失败**（会自动下载），别写死"缺图=冒烟失败"那种假判据。

## ⚠️ 代理策略（2026-09-13 改，旧版是坑）

**正确策略：有代理就用代理，没代理就直连。绝不硬编码死代理兜底。**

旧版 `daily_pipeline.ps1` 里有这么一段兜底：

```powershell
if (-not $env:HTTP_PROXY) {
    $env:HTTP_PROXY = "http://127.0.0.1:7892"   # ← 硬编码夜煞云端口，已删
    $env:HTTPS_PROXY = "http://127.0.0.1:7892"
}
```

关掉 VPN 后 7892 端口已死，但这段逻辑会把**本来能直连的源（HN/GitHub/翻译）也导向死代理**
→ 日报整体挂掉。已删除，改为尊重"无代理"状态走直连。

## 校园网（无 VPN）下各源实测（2026-09-13）

| 源 | 状态 | 说明 |
|------|------|------|
| Hacker News / HN-Algolia | ✅ 直连 | **日报主力** |
| GitHub API / Trending | ✅ 直连 | 主力 |
| 东方财富（财经） | ✅ 直连 | |
| bing 图片 / pexels（配图） | ✅ 直连 | |
| mymemory（翻译） | ✅ 直连 | |
| ❌ arxiv（学术） | ❌ 超时 | **有本地兜底 `SEMICONDUCTOR_DATA`，不影响出报** |
| ❌ V2EX / 36kr | ❌ 超时 | 非核心板块，跳过 |
| ❌ 新浪行情 | ⚠️ 403 | 需 Referer，部分可用 |

**结论：校园网无 VPN 下日报正常生成、质量无损**（实测对比 18:30 有 VPN 版 vs 23:16 无 VPN 版，
内容甚至更丰富，因为 HN/GitHub 才是主力源）。所有抓取函数都有 `try/except` + 8-25s 超时，
死源只是被跳过、不会中断整个生成。

## ❌ arxiv 别折腾了（试过的路全堵）

`export.arxiv.org` 直连不通；换域名 `arxiv.org/api/query` → **302 跳回 export.arxiv.org**，死循环；
ghproxy 转发 → **403（只放行 github 域名）**；gh-proxy / ghfast.top / jina / x-mol 全不通。
→ arxiv 板块靠脚本内 `SEMICONDUCTOR_DATA` 本地静态数据兜底（来源爱集微等国内站），
渲染成"前沿论文/存储芯片/应用芯片"等板块，**够用，不要再去接镜像**。

## PowerShell 坑（`-replace` 第2参数）

`daily_pipeline.ps1` 往 HTML 里插吉大通知那块曾写错：

```powershell
# ❌ 报 "The -ireplace operator allows only two elements to follow it, not 4"
$h = $h -replace '(<body[^>]*>)', '$1' + "`n" + $jluHtml

# ✅ 用双引号插值拼成一个替换串（$1 要转义成 `$1，否则被 PS 当自己的变量）
$h = $h -replace '(<body[^>]*>)', "`$1`n$jluHtml"
```

## 其他已知坑

- `print(emoji)` 在无控制台环境崩 → 跑前设 `$env:PYTHONUTF8="1"`
- `pipeline_log.txt` 是 **UTF-16**，读取要 `Get-Content -Encoding Unicode`
- 日报开头会前置**吉大每日通知**（`jlu_notices.py`，抓 OA `jldxList`，按学生事务关键词过滤
  + 排除学院党建/研究生噪声；`KWS`/`EXCLUDE` 可调）
