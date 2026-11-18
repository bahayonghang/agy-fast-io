# 快速检索与读写规则（fast-io）

目标：用最少的回合完成任务。每多一个回合要多等 10 秒以上；桌面版里，白名单外的命令和每次改文件还要多等约 2 秒安全审查。

## 1. 找代码：先定位，再精读

- 不知道代码在哪时，先用一条命令拿到文件和行号，不要靠逐个打开文件来找。按 `run_command` 声明的操作系统与 Shell 选择命令（写明要搜的源码目录如 `Sources Tests`，不要对 `.` 或含构建产物的目录无过滤递归）：
  - 按文件名找文件（三端通用）：`git ls-files -- ':(icase)*name*'`（非 Git 目录：macOS / Linux 用 `ls`，Windows 用 `Get-ChildItem -Recurse -File -Filter '*name*' Sources | Select-Object -First 100`）。
  - **macOS / Linux（`bash` / `zsh`）**：
    - 搜内容：`grep -rn 'Pattern' Sources Tests`；只要文件名：`grep -rln 'Pattern' Sources`。
    - 看结构：`ls Sources/Module`、`git ls-files Sources/Module | head -100`。
  - **Windows（`pwsh` / `powershell`）**：默认无 `grep`/`head`/`tail`。
    - 搜内容：`git grep -n 'Pattern' -- Sources Tests`（只要文件名：`git grep -l 'Pattern' -- Sources`，限定后缀：`'Sources/*.swift'`；非 Git 目录加 `--no-index`，或对明确源码目录用 `Select-String -Path 'Sources/*.swift' -Pattern 'Pattern'`，严禁在根目录跑 `Get-ChildItem -Recurse | Select-String`）。
    - 看结构：`ls Sources/Module`、`git ls-files Sources/Module | Select-Object -First 100`。
- 有了行号后，用 `read_file` 带 `StartLine` / `EndLine` 读命中位置附近，一次读够：一段读 150–200 行（命中行之前少一些、之后多一些），宁可多读，也不要读完再回头补前后几十行。300 行以内的文件直接整份读。大文件不要从第 1 行整份读，也不要按 800 行一页往下翻。
- 同一轮里，把已经确定要读的文件、互不依赖的搜索一次性并行发出。不要读一个、想一下、再读下一个。
- 同一个文件要看几处时，在同一轮里并行读这几段；几处离得近（相距 200 行以内）就读一段把它们都覆盖住。不要一轮只读一段。
- 只读和问题直接相关的文件。先看搜索命中了哪些文件，再挑最相关的读；不要为了“全面”把整个目录读一遍。
- 读过且没改过的内容不要重读。

## 2. 命令写法

桌面版会把白名单外的命令送去远程审查，每次约 2 秒。命令分两类处理。

### 检索和查看：只用免审与系统原生只读命令

- **macOS / Linux（`bash` / `zsh`）**只用白名单内的命令：
  `ls` `cat` `head` `tail` `wc` `grep` `diff` `stat` `du` `jq` `which` `pwd` `echo` `basename` `dirname` `realpath`，以及 `git status`、`git diff`、`git log`、`git show`、`git ls-files`、`git rev-parse`。
  - 这些命令之间可以用 `|` 串起来，例如 `grep -rn 'x' Sources | head -50`。
  - 检索时不要用 `rg` `fd` `find` `sed` `awk` `sort` `uniq` `xargs` `python`。
  - 不要写 `&&` `||` `;` `>` `2>/dev/null` `$(...)` 和反引号，命令前不要加 `VAR=value`，命令本身只写裸命令名（如 `grep`，不写 `/usr/bin/grep`）。不写 `cd`；不在当前工作目录时，直接把加引号的目标路径传给命令（如 `grep -rn 'x' '/abs/path/Sources'`），路径中不要带 `..`。
  - glob 和正则一律加引号，例如 `--include='*.swift'`。
  - git 参数用长写法：`git log --max-count=5 --oneline`，不写 `-5`。
- **Windows（`pwsh` / `powershell`）**：
  - 不要盲调默认缺失的 `grep` `head` `tail` `wc` `stat` `du` `which` `basename` `dirname` `realpath`；截断与计数用 `| Select-Object -First 50`、`| Select-Object -Last 20`、`| Measure-Object -Line`，查命令位置用 `Get-Command`（或 `where.exe`）。
  - 避开 PowerShell 别名陷阱：`diff` 是 `Compare-Object`（比较路径字符串而非文件内容），对比文件用 `git diff --no-index -- a b`；`ls`、`cat` 只传裸路径，不要传 `-la`、`-n` 等 POSIX 短参数。
  - 默认 Shell 已是 `pwsh` 时直接写裸命令，不要套 `pwsh -c "..."` 外层引号（合并成单 token 会导致白名单前缀匹配失效）；跨目录直接传加引号路径（如 `'D:/path/Sources'`），路径中不要带 `..`。glob 和正则加引号，git 参数同样用长写法（如 `git log --max-count=5 --oneline`）。（若用户规则已显式要求统一调用 Git Bash `bash.exe -lc "..."`，则按用户规则在 bash 内使用 POSIX 命令。）

### 跑脚本、构建、测试：能合并就一次跑完

`python3` / `python`、`node`、`npm`、`swift`、`ego-browser`、项目脚本这类白名单外的命令，每次调用都要送审；合并在一次里跑也只审一次。每多拆一轮，还要多等一个模型回合（10 秒以上）。

- Python 命令：macOS / Linux 用 `python3`；Windows 用 `python`（或 `py -3`，不要调 `python3`，避免命中 `WindowsApps\python3.exe` 商店占位符）。
- 互不依赖、不需要先看上一步结果的几步，合在一次 `run_command` 里跑完：
  - `bash` / `zsh` 与 `pwsh` 7 用 `&&` 串起来（某一步失败也要继续跑后面的，就用 `;`）；Windows PowerShell 5.1 不支持 `&&`，改用 `;`（需遇错即停时前置 `$ErrorActionPreference = 'Stop';`）。丢弃输出在 Windows 用 `$null`（不用 `/dev/null`）。
  - 也可写一个小脚本一次做完。
- 要根据上一步的输出决定下一步时（比如先看报错再改），照常分开跑。
- 合并后输出会变长：步骤之间用 `echo '== 步骤名'` 分隔，每步只留关键几行（macOS / Linux 接 `| tail -20`，Windows 接 `| Select-Object -Last 20`）。
- 这一条只适用于本来就会送审的命令；单纯检索和查看仍按上一节，只用只读白名单或系统原生命令。

## 3. 改文件

- 小改动用 `edit_file`，不要用 `write_file` 重写整个文件。
- 改名、替换这类改动：搜索输出的那一整行已经能在文件里唯一定位时，直接拿这一行做 Target 调 `edit_file`，不用先 `read_file`。
- 同一文件里相邻的几处修改，合成一次 `edit_file`（Target 取覆盖这几处的连续片段）。
- 互不依赖的修改在同一轮并行发出；跨 5 个以上文件的同类机械替换，改用一次 Python 脚本批量完成（macOS / Linux 用 `python3`，Windows 用 `python`），读写文本文件时显式指定 `encoding='utf-8', newline=''`（防中文 Windows 默认 GBK 解码报错及把 LF 换行静默改成 CRLF），避免多次排队送审。

## 4. 回答要短

最后一轮回答往往是整个任务里最慢的一步：模型每写 1,000 字大约要多花 10 秒。

- 先给结论，再列要点。每条一两行，写清文件路径和函数名就够了。
- 不贴大段代码，不复述读到的代码，不写背景介绍和总结段落。
- 一般问题的回答控制在 1,500 字以内；用户明确要求详细说明时再展开。

## 5. 收尾

- 找到答案就直接回答，不为了确认再多翻文件。
- 小任务不要创建计划、任务清单或总结类文档，除非用户要求。
