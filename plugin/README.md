# fast-io

Antigravity 本地插件，只包含一份规则（`rules/AGENTS.md`），不带工具、Hook 或 MCP。

作用是让 Agent：

- 先用 `grep`（macOS / Linux）或 `git grep` / `git ls-files`（Windows）拿到文件和行号，再按行号一次读够（同一文件几处并行读），少开文件、少翻页；
- 同一轮并行读文件、并行搜索，减少模型回合；
- 检索只用免审或系统原生只读命令（避开 Windows 缺失的 POSIX 命令、PowerShell 别名和 `python3` 商店占位符）；跑脚本、构建、测试则合并一次跑完（少审）；
- 改文件用 `edit_file` 合并小改（跨多文件机械替换用单次 UTF-8 脚本批处理），控制回答字数，不为小任务写计划文档。

## 安装

优先用 `agy` 命令行（全平台通用）：

```bash
agy plugin install ./plugin
```

也可以直接复制到全局插件目录（支持首次安装与覆盖升级）：

**macOS / Linux：**

```bash
mkdir -p ~/.gemini/config/plugins
rm -rf ~/.gemini/config/plugins/fast-io
cp -R plugin ~/.gemini/config/plugins/fast-io
```

**Windows（PowerShell 5.1 / 7）：**

```powershell
New-Item -ItemType Directory -Force "$HOME\.gemini\config\plugins" | Out-Null
if (Test-Path "$HOME\.gemini\config\plugins\fast-io") { Remove-Item -Recurse -Force "$HOME\.gemini\config\plugins\fast-io" }
Copy-Item -Recurse "plugin" "$HOME\.gemini\config\plugins\fast-io"
```

装好后在 Antigravity 的 Plugins 面板里能看到 Fast IO（没出现就重启一次），可以随时开关；命令行也可以：

```bash
agy plugin disable fast-io
```

```bash
agy plugin enable fast-io
```

## 效果

实测效果、原理和局限见仓库首页的 README。
