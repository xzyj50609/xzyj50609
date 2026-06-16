# xzyj50609

考研在读，业余折腾 AI 工具链和各种自动化。日常环境是 **Windows 11 + PowerShell**，
喜欢把踩过的坑写成"排查路径"而不是"标准答案"——因为下次遇到的现象大概率不一样，但收敛思路是通用的。

---

## 项目

| 仓库 | 简介 |
|------|------|
| [pdfdir-ai](https://github.com/xzyj50609/pdfdir-ai) | AI 辅助生成 PDF 目录书签 |
| [kaoyan-question-classification](https://github.com/xzyj50609/kaoyan-question-classification) | 考研真题自动分类 |
| [cc-switch-bypass-permissions](https://github.com/xzyj50609/cc-switch-bypass-permissions) | 修复 CC Switch 打开 Claude Code 不是完全访问模式 |
| [cc-switch-chinese-path-crlf-bug](https://github.com/xzyj50609/cc-switch-chinese-path-crlf-bug) | CC Switch 中文路径报 `/d` 不是命令——换行符 bug 根因与修复 |

---

## 文章

**[终于查明白了：CC Switch 打开的 Claude Code 为什么一直不是完全访问模式](https://github.com/xzyj50609/cc-switch-bypass-permissions)**

Fix CC Switch launching Claude Code without bypassPermissions on Windows —— 含完整排障过程、一键安装脚本、回滚方法。

**[CC Switch 打开中文路径报 '/d' 不是命令——换行符 bug 完整排查](https://github.com/xzyj50609/cc-switch-chinese-path-crlf-bug)**

CC Switch 3.16.3 的启动 bat 混合换行（LF + CRLF），在中文多字节路径下导致 cmd.exe 行解析错位。含源码分析、复现矩阵、三种修复方案。

---

## 打赏

如果我写的东西帮到了你，欢迎请我喝杯咖啡。

<table>
<tr>
<td align="center"><b>微信支付</b></td>
<td align="center"><b>支付宝</b></td>
</tr>
<tr>
<td><img src="./assets/wechat-pay.jpg" width="200"></td>
<td><img src="./assets/alipay.jpg" width="200"></td>
</tr>
</table>
