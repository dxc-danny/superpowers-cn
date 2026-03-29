# Claude Code 跨平台多语言钩子

Claude Code 插件需要能在 Windows、macOS 和 Linux 上工作的钩子。本文档解释了这一使这成为可能的多语言包装器技术。

## 问题所在

Claude Code 通过系统的默认 shell 运行钩子命令：
- **Windows**: CMD.exe
- **macOS/Linux**: bash 或 sh

这带来了几个挑战：

1. **脚本执行**：Windows CMD 无法直接执行 `.sh` 文件 — 它会尝试在文本编辑器中打开它们
2. **路径格式**：Windows 使用反斜杠（`C:\path`），Unix 使用正斜杠（`/path`）
3. **环境变量**：CMD 中不支持 `$VAR` 语法
4. **PATH 中没有 `bash`**：即使安装了 Git Bash，当 CMD 运行时 `bash` 也不在 PATH 中

## 解决方案：多语言 `.cmd` 包装器

多语言脚本在多种语言中同时是有效的语法。我们的包装器在 CMD 和 bash 中都有效：

```cmd
: << 'CMDBLOCK'
@echo off
"C:\Program Files\Git\bin\bash.exe" -l -c "\"$(cygpath -u \"$CLAUDE_PLUGIN_ROOT\")/hooks/session-start.sh\""
exit /b
CMDBLOCK

# Unix shell runs from here
"${CLAUDE_PLUGIN_ROOT}/hooks/session-start.sh"
```

### 工作原理

#### 在 Windows (CMD.exe) 上

1. `: << 'CMDBLOCK'` — CMD 将 `:` 视为标签（如 `:label`），忽略 `<< 'CMDBLOCK'`
2. `@echo off` — 抑制命令回显
3. bash.exe 命令运行：
   - `-l`（登录 shell）以获取包含 Unix 工具的正确 PATH
   - `cygpath -u` 将 Windows 路径转换为 Unix 格式（`C:\foo` → `/c/foo`）
4. `exit /b` — 退出批处理脚本，在此停止 CMD
5. `CMDBLOCK` 之后的所有内容 CMD 永远无法到达

#### 在 Unix (bash/sh) 上

1. `: << 'CMDBLOCK'` — `:` 是空操作，`<< 'CMDBLOCK'` 开始 here-document
2. 直到 `CMDBLOCK` 的所有内容都被 here-document 消耗（忽略）
3. `# Unix shell runs from here` — 注释
4. 脚本使用 Unix 路径直接运行

## 文件结构

```
hooks/
├── hooks.json           # 指向 .cmd 包装器
├── session-start.cmd    # 多语言包装器（跨平台入口点）
└── session-start.sh     # 实际钩子逻辑（bash 脚本）
```

### hooks.json

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/session-start.cmd\""
          }
        ]
      }
    ]
  }
}
```

注意：路径必须加引号，因为 `${CLAUDE_PLUGIN_ROOT}` 在 Windows 上可能包含空格（例如 `C:\Program Files\...`）。

## 要求

### Windows
- **Git for Windows** 必须已安装（提供 `bash.exe` 和 `cygpath`）
- 默认安装路径：`C:\Program Files\Git\bin\bash.exe`
- 如果 Git 安装在其他位置，需要修改包装器

### Unix (macOS/Linux)
- 标准 bash 或 sh shell
- `.cmd` 文件必须有执行权限（`chmod +x`）

## 编写跨平台钩子脚本

你的实际钩子逻辑放在 `.sh` 文件中。为确保它在 Windows 上通过 Git Bash 工作：

### 建议做法：
- 尽可能使用纯 bash 内置命令
- 使用 `$(command)` 而不是反引号
- 引号所有变量展开：`"$VAR"`
- 使用 `printf` 或 here-document 输出

### 避免：
- 外部命令可能不在 PATH 中（sed、awk、grep）
- 如果必须使用，它们在 Git Bash 中可用，但确保 PATH 已设置（使用 `bash -l`）

### 示例：不使用 sed/awk 的 JSON 转义

不要用：
```bash
escaped=$(echo "$content" | sed 's/\\/\\\\/g' | sed 's/"/\\"/g' | awk '{printf "%s\\n", $0}')
```

使用纯 bash：
```bash
escape_for_json() {
    local input="$1"
    local output=""
    local i char
    for (( i=0; i<${#input}; i++ )); do
        char="${input:$i:1}"
        case "$char" in
            $'\\') output+='\\' ;;
            '"') output+='\"' ;;
            $'\n') output+='\n' ;;
            $'\r') output+='\r' ;;
            $'\t') output+='\t' ;;
            *) output+="$char" ;;
        esac
    done
    printf '%s' "$output"
}
```

## 可重用的包装器模式

对于有多个钩子的插件，你可以创建一个接受脚本名称作为参数的通用包装器：

### run-hook.cmd
```cmd
: << 'CMDBLOCK'
@echo off
set "SCRIPT_DIR=%~dp0"
set "SCRIPT_NAME=%~1"
"C:\Program Files\Git\bin\bash.exe" -l -c "cd \"$(cygpath -u \"%SCRIPT_DIR%\")\" && \"./%SCRIPT_NAME%\""
exit /b
CMDBLOCK

# Unix shell runs from here
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
SCRIPT_NAME="$1"
shift
"${SCRIPT_DIR}/${SCRIPT_NAME}" "$@"
```

### 使用可重用包装器的 hooks.json
```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" validate-bash.sh"
          }
        ]
      }
    ]
  }
}
```

## 故障排除

### "bash is not recognized"
CMD 找不到 bash。包装器使用完整路径 `C:\Program Files\Git\bin\bash.exe`。如果 Git 安装在其他位置，请更新路径。

### "cygpath: command not found" 或 "dirname: command not found"
Bash 没有作为登录 shell 运行。确保使用 `-l` 标志。

### 路径中有奇怪的 `\/`
`${CLAUDE_PLUGIN_ROOT}` 展开为以反斜杠结尾的 Windows 路径，然后附加了 `/hooks/...`。使用 `cygpath` 转换整个路径。

### 脚本在文本编辑器中打开而不是运行
hooks.json 指向 `.sh` 文件。请改为指向 `.cmd` 包装器。

### 在终端中工作但作为钩子不工作
Claude Code 可能以不同方式运行钩子。通过模拟钩子环境进行测试：
```powershell
$env:CLAUDE_PLUGIN_ROOT = "C:\path\to\plugin"
cmd /c "C:\path\to\plugin\hooks\session-start.cmd"
```

## 相关问题

- [anthropics/claude-code#9758](https://github.com/anthropics/claude-code/issues/9758) - .sh 脚本在 Windows 上在编辑器中打开
- [anthropics/claude-code#3417](https://github.com/anthropics/claude-code/issues/3417) - 钩子在 Windows 上不工作
- [anthropics/claude-code#6023](https://github.com/anthropics/claude-code/issues/6023) - CLAUDE_PROJECT_DIR 未找到