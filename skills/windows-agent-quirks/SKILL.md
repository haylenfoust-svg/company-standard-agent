---
name: windows-agent-quirks
description: "Use when running terminal commands or file ops on Windows."
version: 1.0.0
author: company-shared
license: MIT
platforms: [windows]
metadata:
  hermes:
    tags: [windows, shell, powershell, git-bash, encoding, paths, troubleshooting]
---

# Windows 环境避坑通则

## When to Use

在 Windows 主机上执行任何终端命令、文件操作、安装软件之前加载本技能并遵循其规则;
排查 Windows 上的「命令无法识别」、乱码、找不到文件、文件被占用类报错时也加载。

四大高频事故:命令方言混用(PowerShell/bash)、路径错误、中文乱码(GBK/UTF-8)、软件找不到。

## 0. 第一原则

Windows 上并存三种 shell:cmd、PowerShell、git-bash(MSYS),外加 WSL。模型训练数据里的命令
绝大多数是 Linux/bash 语法,直接照搬必错。**执行任何命令前,先确认当前终端是哪种 shell,
只用那种 shell 的语法。** 不确定时先探测:

```bash
echo "$BASH_VERSION"   # git-bash 中非空;PowerShell/cmd 中为空或报错
```

## 1. Shell 方言规则

- Hermes 的 terminal 工具在 Windows 上默认走 **git-bash**:统一用 POSIX/bash 语法
  (`&&`、`$HOME`、单引号、POSIX 风格命令)。**禁用** PowerShell cmdlet
  (`Get-ChildItem`、`Select-String`、`$env:FOO`)——在 bash 里全部报错。
  文件查找/读取优先用 Hermes 原生工具(search_files / read_file),从根上避开方言问题。
- 若会话确实处于 PowerShell:旧版 5.1 不支持 `&&`(改用 `;` 或分开执行);
  `curl`/`wget` 是 `Invoke-WebRequest` 的别名,参数完全不同(改用 `curl.exe` 显式调用真 curl)。
- 有非交互路径(CLI flags、`--with-token`、配置文件、设备码+轮询)时,**永远优先**,
  不要驱动交互式提示。
- PTY 交互提示:Windows 上回车是 `\r`。用 process_manage 的 `submit` 应答,
  **不要用 `write` + 裸 `\n`**——孤立的 `\n` 不作为行结束符,子进程的提示会永远不返回。

## 2. 路径规则

- **原生 Windows 程序**(git、node、python、soffice…)不认 MSYS 路径(`/c/Users/...`、`/tmp`),
  必须传 `C:/Users/...` 正斜杠风格;只有 bash 内建命令(`cd`、`echo`)接受 MSYS 路径。
- 反斜杠 `\` 是转义符:代码、JSON、命令参数里一律用正斜杠 `C:/...`。
- 含空格的路径(`Program Files`)必须整体加引号。
- 临时文件写 `$TMPDIR`(Hermes scratch)或 `$LOCALAPPDATA/Temp`,**绝不写裸 `/tmp`**
  (原生程序找不到)。
- Windows 文件名不分大小写、Linux 分:始终按真实大小写书写路径,避免部署到服务器才炸。
- 默认存在 260 字符路径上限(MAX_PATH):避免深层嵌套目录、超长文件名。

## 3. 编码(中文乱码)

- 中文 Windows 控制台/文件默认 GBK(cp936),AI 生态默认 UTF-8。读写文件**显式指定**
  `encoding='utf-8'`(Python `open(path, encoding='utf-8')`),不依赖系统默认。
- 命令输出乱码 = GBK/UTF-8 错位:git-bash 里可 `export PYTHONIOENCODING=utf-8`、
  `PYTHONUTF8=1`;cmd/PowerShell 里 `chcp 65001` 切 UTF-8 代码页。
- 读带 BOM 的文件用 `encoding='utf-8-sig'`。
- 配置文件(YAML/JSON)绝不能用记事本保存——记事本写入 UTF-8 BOM,会导致解析失败
  (如 HTTP 400 "No models provided")。用 `hermes config edit` 或写无 BOM 的工具。

## 4. 换行符(CRLF/LF)

- bash 执行的 `.sh` 脚本必须 LF 行尾,CRLF 会报 `$'\r': command not found`。
  用 write_file 生成的脚本默认 LF;从网上下载的脚本先检查。
- git 的 "LF will be replaced by CRLF" 警告无害;仓库用 `.gitattributes` 规范化,
  不要让编辑器把提交文件自动转成 CRLF。

## 5. 软件发现(装了但找不到)

- GUI 安装的软件经常**不进 PATH**:`command -v xxx` 查不到 ≠ 未安装。按序搜常见位置:
  1. `C:/Program Files/` 和 `C:/Program Files (x86)/`
  2. 其他盘的同名目录(如 `D:/Program Files/`,用户常把大软件装到 D 盘)
  3. `$LOCALAPPDATA/Programs/`(用户级安装,VS Code、Python 等)
- 可执行文件常在安装目录的子目录里(如 LibreOffice 的 `soffice.exe` 在 `program/` 下)。
- 找到后:用**完整路径**调用,并把「软件名 → 完整路径」写入长期记忆,下次不再搜。
- 装新软件:优先 `winget`,其次 scoop/choco;开源工具的安装脚本大多假设 Linux/macOS,
  Windows 上先找官方 `.exe`/`.msi` release 或 WSL 替代,不要硬跑 `install.sh`。

## 6. 文件占用 / 杀毒 / 权限

- Windows 上被占用的文件**删不掉、移不走**(Linux 可以):报 "Permission denied /
  另一个程序正在使用" 时,先找占用进程(tasklist / 停掉自己刚起的后台进程)再重试,
  不要循环硬试。
- 写 `C:/Program Files`、`C:/Windows` 需要管理员权限(UAC):普通会话改到用户可写目录
  (`$LOCALAPPDATA`、用户主目录)。
- Defender/杀软可能拦截新下载的 exe 或 agent 生成的脚本(表现为拒绝访问或文件凭空消失):
  提示用户检查安全软件记录,**不要替用户关闭杀毒软件**。

## 7. 跨平台可移植

- 最终要部署到 Linux 服务器的代码:不硬编码盘符和 Windows 路径,用 `pathlib`/环境变量;
  路径分隔符交给运行时。
- 同一份脚本要在双端跑:写成 Python/Node 等跨平台语言,不写 `.bat`/`.ps1`/`.sh` 三套。

## 维护规则(共享源)

- 本技能是公司共享技能仓库的 canonical 源,经 `hermes skills tap` 分发给全员,
  `hermes skills update` 持续更新。
- 踩到**新**的 Windows 坑并解决后:用 skill_manage patch 把经验补进对应小节,
  一条规则 = 症状 → 原因 → 修法;不写日期、不写事故叙述。
