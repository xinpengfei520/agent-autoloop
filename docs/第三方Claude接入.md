# 用 rules.json 接第三方 Claude

日期：2026-11-04。这是本机接法笔记，不是上游协议。协议里 Agent 的接法见 [protocol.md](protocol.md) 的「接一个新 Agent」。

这里的「第三方」指：本机装的是官方 `claude` 命令，但没有 Claude 订阅，请求发到自己的兼容网关。令牌留在 `~/.zsh/secrets.env`，不写进 `rules.json`。

## 结论

中台认的是 `rules.json` 里登记过的 Agent，不认终端里的 shell 函数。

能接上的写法是：`use` 保持 `"command"`，`command` 指向一个小脚本。脚本先放上和某个 shell 函数相同的环境变量，再执行 `claude -p`。规则的 `to` 填这个 Agent 的编号。

下面两处改动接不上：

| 想改的字段 | 实际会怎样 |
|---|---|
| 只把规则的 `to` 改成 `txclaudeflash` | 检查报「不是已登记的 Agent」，旧规则继续跑 |
| 把已登记 Agent 的 `use` 改成 `txclaudeflash` | 检查能通过。领活时去找 `plugins/txclaudeflash/agent.ts`，找不到就失败 |

`to` 选择 `agents` 里的哪一条。`use` 选择用哪个插件来启动。现成插件是 `claude`、`codex`、`command`。`txclaudeflash` 是 `~/.zsh/claude.zsh` 里的函数，只有交互式终端会加载，系统里也没有这个可执行文件。那个函数最后跑的是交互式 `claude --dangerously-skip-permissions`，不接收中台给的提示词，也不会打完就退出。

## 为什么裸的 claude 会报 OAuth 过期

内置 `use: "claude"` 直接执行本机的 `claude -p`，登录走 Claude Code 自己的 OAuth。本机当时的状态是：

- `claude auth status` 显示未登录
- `~/.claude/.credentials.json` 里仍留着 Claude.ai 的 OAuth
- `~/.claude/settings.json` 没有第三方地址
- 网关地址和令牌只在 shell 函数里，中台启动时不读 `.zshrc`，也不会调用这些函数

所以进程一启动就去刷新过期登录，得到 `Failed to authenticate: OAuth session expired and could not be refreshed`。请求没有发到网关。

中台启动 Agent 前还会去掉环境里继承来的 `CLAUDE_CODE_*`。`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`、`CLAUDE_CODE_SUBAGENT_MODEL` 这类要写在脚本里。写在服务的环境变量里到不了 `claude`。

## 本机实际接法

脚本：`~/.agent-autoloop/bin/txclaudeflash.sh`。它对应 shell 函数 `txclaudeflash`（腾讯 Token Hub 上的 DeepSeek V4.1 Flash）。令牌变量名是 `TX_TOKEN`。

```bash
#!/bin/zsh
set -eo pipefail
source "$HOME/.zsh/secrets.env"
if [[ -z "${TX_TOKEN:-}" ]]; then
  echo "TX_TOKEN 没有配在 ~/.zsh/secrets.env" >&2
  exit 1
fi
export ANTHROPIC_AUTH_TOKEN="$TX_TOKEN"
export ANTHROPIC_BASE_URL="https://tokenhub.tencentmaas.com"
export ANTHROPIC_MODEL="deepseek/deepseek-flash[1m]"
export ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek/deepseek-flash[1m]"
export ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek/deepseek-flash[1m]"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="deepseek/deepseek-flash[1m]"
export CLAUDE_CODE_SUBAGENT_MODEL="deepseek/deepseek-flash[1m]"
export ENABLE_TOOL_SEARCH=false
export CLAUDE_CODE_AUTO_COMPACT_WINDOW=786432
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
export CLAUDE_CODE_WORKFLOWS=1
exec "$HOME/.local/bin/claude" "$@"
```

`~/.agent-autoloop/rules.json` 里登记一条，`use` 保持 `command`：

```json
"txclaudeflash": {
  "name": "腾讯 DS V4.1 Flash",
  "worker": "mbp",
  "use": "command",
  "command": [
    "/Users/vancexin/.agent-autoloop/bin/txclaudeflash.sh",
    "-p",
    "{prompt}",
    "--permission-mode",
    "acceptEdits"
  ]
}
```

规则的 `to` 写 `"txclaudeflash"`。保存后运行 `agent-autoloop check`。几秒内生效。已经在排队、还没领走的任务，下次领取会改用这个 Agent。

换别的供应商时，照 `~/.zsh/claude.zsh` 里对应函数抄环境变量，换成那个函数使用的令牌变量名。交互式函数本身不用改，这套只给中台用。

## 命令行里必须守住的几点

- `{prompt}` 单独做数组里的一项。中台不做 shell 展开。
- 要有 `-p`，干完才退出，最后一条回复才交得回中台。
- 不要加 `--output-format json`。`use: "command"` 把整个标准输出当成回复正文。官方 `claude` 插件才会解析 JSON。
- 权限用 `--dangerously-skip-permissions`。无界面运行时没有弹窗可点，`acceptEdits` 只放行工作目录里的修改，读运行记录、拉资讯和调用知识库都会被拒绝。这条命令只给中台用。
- 网关可能往标准错误打模型名兼容提示。中台只把标准输出当回复，这条提示不影响回执。若它出现在标准输出的第一行，第一行结论会被带偏，回执代码块仍能生效。

单独验证脚本时，不要把令牌打出来：

```bash
"$HOME/.agent-autoloop/bin/txclaudeflash.sh" -p "只回复一个字：好。不要调用工具。" --permission-mode plan
```

本机这次返回了「好」，说明认证走的是 Token Hub，不再刷新 Claude.ai。

## 这条接法换不到的能力

官方 `use: "claude"` 会解析 JSON、续上会话、上报 Token 和费用。`use: "command"` 做不到这三件。会话断了就靠资料文件接着做。费用在 `agent-autoloop status` 里会显示没上报。

若以后要续会话和用量，再写一个 `plugins/<名字>/agent.ts`，`command()` 仍调用这个脚本，`read()` 照官方 Claude 插件解析 JSON。只为换接口时不必加这一层。

## 试跑时容易误看成「还是 OAuth」

`agent-autoloop try` 只要看见规则「试一下」有一轮还在排队、而且已经失败过，就会立刻打印那一轮留下的旧错误，不会等下一次领取。

本机的情况是：规则改成 `txclaudeflash` 之前，第 1 轮已经用官方 Claude 失败了四次，并排到 11:50 才重试。11:30 之后再跑 `try`，只是把新事件并进这一轮，然后读到旧的 OAuth 错误。当时脚本还没被启动。

确认脚本本身能通之后，用 `agent-autoloop stop 1` 停掉那一轮，再跑 `agent-autoloop try`。停掉之前，`agent-autoloop why 1` 能看出每次领取的仍是「Claude Code」。
