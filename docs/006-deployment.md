# 006 部署

使用本指南安装、配置和运营 dsh-superagent。它涵盖账号、启动检查、升级和排障，建立在 [001 领域模型](001-domain.md) 到 [005 面板](005-panel.md) 之上。

## 运行什么

- **一个专用的 DeepSeek Harness 配置文件（profile）**，本指南中称为 `superagent`。从 Web 配置文件模板创建它。它加载 bundle 并在 `127.0.0.1` 上提供其面板。把它与你工作的配置文件分开：插件会更改进程范围的 `PATH` 和 git 设置，其进程需要独立的重启和升级。
- **一个智能体 GitHub 账号。** 智能体以该账号评论、推送分支和发起拉取请求，通过一个只以该账号登录的 GitHub CLI。
- **可选：** 一个用于聊天请求的飞书机器人。

插件配置保存状态目录和机器限制。其余一切用面板：Sources 用于仓库和账号，Policy 用于允许名单，Execution 用于 worker 目录和命令。

### 权限和隔离

bundle 为执行器定义 `dsh-superagent-worker`，为主 Session 定义 `dsh-superagent-router`。它们出现在 DeepSeek Harness 预设列表中，但只供 dsh-superagent 使用。不要为个人 Session 选择它们，也不要禁用它们的配置行。应用需要它们来创建和恢复它的 Session。

在 Web UI 打开一个空白受管 Session 可以选择配置文件的默认预设。dsh-superagent 在第一个回合前恢复其角色专属预设。

两个预设都以 `danger-full-access` 作为运营者的 OS 用户运行，并读取订阅线程的消息。账号包装器默认选择智能体的凭据；它们不隔离进程。worker 可以按路径执行程序，并读取运营者的主目录。要隔离，请使用一个只包含智能体凭据的专用 OS 账号。

## 前置条件

- Node.js 22.19+ 或 24+、pnpm、git 2.32+、Python 3.8+ 和 tmux
- GitHub CLI（`gh`）2.48 或更高。来源 CLI 字段必须指向 `gh` 或 `lark-cli` 包装器；宿主依赖它们的子命令和输出格式。
- 一个 github.com 上的仓库；不支持 GitHub Enterprise Server。要为每个工单创建检出，请保留一个本地 clone，让 prepare 命令可以从中添加 worktree。如果该仓库使用 Git LFS，请安装 `git-lfs`。
- 为 DeepSeek Harness 配置一个模型提供商。dsh-superagent 使用配置文件的默认模型，除非 Eval 或 Execution 设置指定另一个。
- 飞书：`lark-cli`（在 1.0.90 上测试）和一个带机器人的飞书或 Lark 应用

## 账号

### 智能体 GitHub 账号

1. 为智能体创建一个 GitHub 用户账号，并给它仓库写权限。GitHub App 不能作为智能体，因为 dsh-superagent 读取该账号的通知。
2. 给它一个只使用自己凭据的 GitHub CLI：

   ```sh
   mkdir -p ~/.local/bin
   cat > ~/.local/bin/gh-agent <<'EOF'
   #!/bin/sh
   GH_CONFIG_DIR="$HOME/.config/gh-agent" exec gh "$@"
   EOF
   chmod +x ~/.local/bin/gh-agent
   gh-agent auth login
   gh-agent api user --jq .login
   ```

   以智能体账号登录；最后一条命令打印它的登录名。dsh-superagent 以同样方式读取登录名。

### 飞书机器人（可选）

1. 创建一个带机器人能力的自建应用。授予它接收消息事件、读取和发送聊天消息、读取聊天成员的权限，然后发布应用版本。
2. 给机器人它自己的 `lark-cli` 配置，限制为机器人身份，以及一个包装器：

   ```sh
   LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config init --app-id <cli_…> --app-secret-stdin
   LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent" lark-cli config strict-mode bot --global
   cat > ~/.local/bin/lark-bot <<'EOF'
   #!/bin/sh
   export LARKSUITE_CLI_CONFIG_DIR="$HOME/.lark-cli-dsh-superagent"
   exec lark-cli "$@"
   EOF
   chmod +x ~/.local/bin/lark-bot
   ```

   第一条命令从标准输入读取应用密钥。使用单独的配置目录：`lark-cli` 每个 app id 只允许一个配置文件，你的个人配置可能已包含该应用的用户登录。

   宿主拒绝一个能以用户身份行动的 CLI。它从机器人配置读取 app id、飞书/Lark 品牌和机器人 `open_id`。

   两个配置共享应用密钥的 keychain 条目。要只移除机器人配置，删除它的目录；不要运行 `lark-cli config remove`，那会影响共享密钥。

## 安装

在没有 `superagent` 配置文件的机器上，运行一次这段命令：

```sh
npm install -g @deepseek-ai/dsh@0.2.1-rc.3
git clone <this repository> dsh-superagent && cd dsh-superagent
pnpm install && pnpm build
dsh superagent --from-default-profile web --dump-config > /dev/null
dsh plugin --profile superagent add "link:$PWD"
```

上面的固定版本是公开的；其他版本必须满足 `package.json` 中的 `engines.dsh`。

第四条命令从 DeepSeek Harness Web 模板创建 `~/.dsh/profiles/superagent`。第五条安装这个 bundle。设置 `DSH_HOME` 可迁移默认的 `~/.dsh` 目录。

只在全新安装时运行配置文件创建命令。如果配置文件已存在，该命令会替换它并删除它的 `cordis.patch.yml`，包括机器配置。升级时不要使用它。

## 机器配置

把 [examples/profile.cordis.patch.yml](../examples/profile.cordis.patch.yml) 复制到 `~/.dsh/profiles/superagent/cordis.patch.yml`（替换该文件），并替换每个 `<placeholder>`。对于飞书，还要在那里启用 `dsh-superagent-feishu` 条目。`webserver` 条目设置面板的端口。

| 键 | 必填 | 含义 |
| --- | --- | --- |
| `stateDir` | 是 | 绝对路径：数据库、锁、`worker-bin` 中的命令 shim、智能体的 git 配置，以及默认情况下 `work` 中的工单目录。升级前备份它 |
| `promptMaxChars`、`admissionContextMaxPages`、`admissionContextTimeoutMs` | 否 | 可保存的最大提示词配置，以及一次研判在什么时间内读取多少个线程页 |

首次启动后，在 **Settings → Deployment** 或 DeepSeek Harness 插件管理器中编辑机器限制。变更无需重启即生效。`stateDir` 不能在那里更改。无效值或未知配置键会阻止启动；日志会指出该键。

## 启动和停止

```sh
scripts/dsh-superagent start
scripts/dsh-superagent status
```

`start` 在 tmux 会话 `dsh-superagent` 中运行 `dsh --profile superagent`，对其他配置文件为 `dsh-<profile>`。它最多等待一分钟，让运行时就绪或报告设置要求。首次运行时会打印缺失的设置信息。它拒绝重复实例，包括在本脚本之外启动的。

`stop` 和 `restart` 请求优雅关停，并最多等待执行器五分钟。它们绝不强杀进程或触碰其他 tmux 会话。

`status` 打印进程 id、健康和设置问题。对于不同的配置文件或端口，使用 `--profile` 和 `--url`，或设置 `DSH_SUPERAGENT_PROFILE` 和 `DSH_SUPERAGENT_URL`。

`status`、`start` 和 `restart` 以四个退出码之一退出：

| 码 | 含义 |
| --- | --- |
| 0 | 服务正常运行 |
| 1 | 它没有运行、运行了两次、不应答，或其运行时报告错误；重启可能修复它 |
| 2 | 它在运行，但运行时之外的部分（例如来源或策略同步 `sync:<source id>`）报告错误；重启不能修复它 |
| 3 | 它在运行并等待设置；重启不能修复它 |

仍在启动的来源算正常操作。设置无法检查的飞书机器人也算。

`status` 在 `errors` 下返回失败的部分及其详情，在 `setup` 下返回设置要求。`start` 和 `restart` 把这些打印为行；如果服务始终没有起来，它们退出 1。只有有意的 `stop` 会暂停自动恢复。

每个配置文件把日志和控制文件存在 `.artifacts/` 下：

- 启动日志：`dsh-superagent-<profile>-<date>-<time>-<pid>.log`，其中 pid 属于脚本。保留最新的 20 个。
- 控制文件：`control/profiles/<profile>/`，包含停止标记、锁和进程 id。

打开日志中打印的 URL，然后在侧边栏选择 **Superagent**。

### 每小时检查

在你平时使用的 DeepSeek Harness 配置文件里创建每小时检查，而不是在 `superagent` 里。它必须在智能体服务停机时仍能运行。让该配置文件里的一个智能体用这个提示词创建每小时计划，填好检出路径：

```text
Check dsh-superagent: run `scripts/dsh-superagent --profile superagent status` in <checkout>. Exit 0: do nothing and say nothing. Exit 2: do not restart; tell me each part under `errors` with its detail, unless you already told me about the same ones. Exit 3: do not restart; tell me what `setup` says setup needs, unless you already told me. If it reports manually_stopped: true, leave it stopped. Otherwise (exit 1) run `scripts/dsh-superagent --profile superagent --respect-stop restart`, which exits the same way: 0 means it recovered, so say nothing; 2 and 3 are reported as above; if it exits 1 or fails otherwise, read the newest .artifacts/dsh-superagent-superagent-*.log and the Troubleshooting section of docs/006-deployment.md, fix what you can, and tell me what was wrong and what you did.
```

`--respect-stop` 保留有意的停止。只有 `stop` 写标记；`start` 清除它。`restart` 不写标记，所以检查可以恢复一个关停超时的重启。

## 设置

在轮询或工作开始之前，设置必须成功。Overview 列出缺失的要求并链接到它们的设置。设置变更会再次运行设置。如果你在应用之外修复了什么（例如 CLI 认证），选择 **Check again（再次检查）**。

1. **Sources。** 设置 GitHub 仓库（`owner/name`）和智能体 CLI，例如 `/home/me/.local/bin/gh-agent`。请求通常通过智能体账号的通知到达。一个提及该账号的新 Issue 或评论在下一次轮询时到达；通过编辑已有 Issue 正文来添加提及不会到达。改为发一条新评论。对于飞书，设置机器人 CLI、群聊、每个群聊的全量接收或仅提及模式，以及是否接收私聊。
2. **Policy。** 两个示例规则可以编辑或移除。在 **On each Source** 下，添加可以请求工作的 GitHub 账号和飞书用户；在名单里有人之前，任何请求都无法成为工单。对于飞书，可以选择一个群聊，在启动时和 **Sync now** 时从中同步名单。
3. **Execution。** 目录根和一个 prepare 命令。要给每个工单一个仓库检出：

   ```sh
   git -C /home/me/src/widgets fetch --no-tags "https://github.com/$DSH_SUPERAGENT_GITHUB_REPOSITORY.git" "${DSH_SUPERAGENT_GITHUB_PR_HEAD:-HEAD}:refs/dsh-superagent/$DSH_SUPERAGENT_TICKET" &&
   git -C /home/me/src/widgets worktree add -b "agent/$DSH_SUPERAGENT_NAME" "$DSH_SUPERAGENT_DIR" "refs/dsh-superagent/$DSH_SUPERAGENT_TICKET"
   ```

   把个人 CLI（例如登录到你自己的账号的）列为阻止的命令。
4. **Prompts。** 设置团队部分：GitHub 披露行、评审机器人、提交作者和自动合并条件。

### 管理 API

面板使用一个本地 HTTP API，智能体也可以调用它。`SETUP.md` 用它保存设置，服务脚本读取 `GET state`。

向 `http://127.0.0.1:<port>/orchestrator/api/` 发送请求，带 `x-orchestrator-request: 1`。宿主必须是 `127.0.0.1:<port>` 或 `localhost:<port>`。缺失的头部或来自其他站点的 `Origin` 会收到 403。

API 保存具有面板的权限。它们可以更改执行环境和设置账号字段，这与通过主 Session 工具的保存不同。

智能体可以依赖这些路由和字段。`src/app/api.ts` 中的其他路由服务于面板，并随面板变化。

| 请求 | 回答 |
| --- | --- |
| `GET state` | 每页所读的内容：`setup`，含 `problem` 和 `feishuProblem`，各为 `null` 或一个带 `place`、`code`、`values` 和 `detail` 的设置问题，以及 `accounts`（一旦设置解析出它们）；`health`，每个部分一条，如下，带 `state`（`starting`、`ready`、`paused` 或 `error`）、`detail` 和 `at`；`load`，执行器 Session 的 `running`、工单的 `queued` 和 `maxWorkers`；`paused`；以及 `masterId` |
| `GET settings` | `blocks`：`policy`、`eval` 和 `executor`，各带其 `value` 和 `revision`；`sources`：来源实例，各带 `id`、`kind`、`installed`、其块的 `value`、`{"enabled": …, "config": {…}}` 和 `revision`、其种类的表单，以及当它有时它的设置 `problem`；`policyVersion`；以及 `environment`，执行环境的 `root` 和 `variables` |
| `GET sources` | 每个来源的状态：`state`（`missing`、`off`、`error`、`setup`、`starting`、`paused` 或 `ready`）、其部分的 `health`、其设置 `problem`、最近一天的 `stats`、它在那天 `filtered` 的触发，以及它在那天 `dropped` 的消息，最新丢弃在前，各带其 `channel`、`preview`、`ticket`（`id` 和 `title`，当它是为某个工单时）、`thread`（当存储有时），以及 `dropped.at` 和 `dropped.reason`、`channelGone` 或 `channelIgnored` |
| `PUT settings/<block>`，带 `{"value": …, "revision": n}` | 保存一个块：`saved`、该块的新 `revision`、`policy_changed` 和 `restarting`。`value` 是整个块，如 `GET settings` 所持有；缺失的键取默认值 |
| `POST executor/pause`，带 `{"paused": true}` 或 `false` | 暂停或恢复处理：`paused` |
| `POST setup/check` | 通过重启 dsh-superagent 再次运行设置：`restarting: true`。当设置没有可修复的内容时被拒绝 |
| `POST policy-sync/<source id>` | 立即同步该来源的允许名单：`policy`，来源已保存的部分 |
| `GET prompts`、`PUT prompts`，带 `{"config", "baseRevision"}` | 读取提示词配置：`current`（带其 `revision` 和 `config`）、`defaults`、`sections`（按 Session 读取顺序的团队部分）、`skillName`（团队技能名的模式）、`builtin`（团队技能不能取的名字），以及 `skillPrefix`（Session 以之加载技能名）。或保存它：新的 `revision` |
| `POST tickets/<id>/retry`、`POST tickets/<id>/stop` | 重试或停止工单，如面板所做 |

健康条目按职责分组：

| 条目 | 报告 |
| --- | --- |
| `runtime` | 应用本身 |
| `models` | 等待模型预算重试的 Session |
| `evals` | 最近一小时没有后续判断或驳回的 unavailable 评估 |
| `assessment`、`launch`、`delivery`、`relay`、`notice`、`reply` | [003 执行](003-execution.md) 中描述的存储的工作；delivery 统计等待的工作，而 reply 覆盖渠道消息 |
| `worker` | 执行器 Session 事件处理；在其首次失败后出现 |
| `github`、`feishu`、`history` | 来源轮询、事件和历史恢复 |
| `sync:<source id>` | 允许名单同步 |

拒绝有状态和 `error`，其英文文本：

- 400：无效值（`issues`，各带其 `path`）、失败的账号检查（`setup`）、固定拒绝（`code` 和 `values`），或不是 JSON 或太大的正文。
- 404：不存在的东西。
- 409：过期的修订版。
- 500：其他一切。

各块：

- `source:github`、`source:feishu`：`{"enabled": true, "config": {…}}`。GitHub 的 config 有 `repository` 和 `cli`，以及开关 `observeIssues`、`observePulls`、`observeComments` 和 `observeCi`，默认关闭，它们接收仓库中的每个新 Issue、新拉取请求和评论，以及默认分支的 CI；飞书的有 `cli`、`chats`（形如 `[{"chatId": "oc_…", "name": "…", "listen": "mentions"}]`，或 `"all"`）、`directMessages`，以及 `otherChats`（`"none"` 或 `"mentions"`）。
- `policy`：`{"rules": […], "sources": {"github": {"allowed": [{"id": "<login>"}], "bots": [{"id": "dependabot[bot]"}]}, "feishu": {"allowed": [{"id": "ou_…", "name": "…"}], "syncChat": "oc_…", "admission": {"oc_…": {"unlistedAddressed": "explicit"}}}}}`。`bots` 列出其消息可以成为推断候选的机器人。`admission` 行对飞书按聊天 id、`other` 或 `direct` 键控，对 GitHub 按 `issues`、`pulls`、`comments` 或 `ci` 键控；未设置的格取默认值（见 [002 授权](002-authorization.md)）。
- `eval`：`mode`、`timeoutSeconds` 和 `model`。
- `executor`：`model`、`maxWorkers`、`directory`、`prepare` 和 `blockedCommands`。处理暂停是它自己的设置 `paused`，由 `POST executor/pause` 设置。

更改仓库或 CLI 会先运行相应的设置检查。失败的检查返回 400，详情在 `setup` 中。成功的账号更改，或设置待处理时的任何保存，返回 `restarting: true`。应用重启期间 API 暂时不可用。

```sh
api=http://127.0.0.1:3082/orchestrator/api
rev=$(curl -s -H 'x-orchestrator-request: 1' $api/settings | jq '.sources[] | select(.id == "github") | .revision')
curl -s -X PUT -H 'x-orchestrator-request: 1' -H 'content-type: application/json' $api/settings/source:github \
  -d "{\"value\": {\"enabled\": true, \"config\": {\"repository\": \"acme/widgets\", \"cli\": \"$HOME/.local/bin/gh-agent\"}}, \"revision\": $rev}"
```

## 设置检查

设置使用 DeepSeek Harness 子进程服务，正如 worker shell 那样。它移除名称含 KEY、PASSWORD、SECRET 或 TOKEN 的环境变量。

设置核实 GitHub 仓库和智能体的推送访问，写入 worker 环境，并检查 git 对 github.com 只使用智能体凭据。飞书设置失败只禁用飞书来源。

设置写入的进程环境：

- **命令 shim** 在 `<stateDir>/worker-bin`，位于 `PATH` 首位：`gh` 运行智能体的 CLI；`lark-cli` 运行机器人的 CLI；阻止的命令失败。shim 在运行其目标前把自身目录从 `PATH` 移除。
- **git 配置** 在 `<stateDir>/worker-gitconfig`：替换运营者的全局和系统 git 配置。它选择智能体 CLI 作为 github.com 凭据助手，把 GitHub SSH 地址重写为 HTTPS，并定义 Git LFS 过滤器。个人 git 设置（例如 HTTP 代理）不适用。这里不设置提交身份，git 从不提示凭据，`GH_TOKEN` 和 `GITHUB_TOKEN` 被移除。

`worker-bin` 或 `worker-gitconfig` 中一个 dsh-superagent 未写入的文件会停止设置。位于一个其自身配置添加了凭据的 git 仓库内的工单目录会使工单失败。

## 试用

从一个允许的 GitHub 账号，打开一个 Issue 或发一条包含 `@<agent login> <request>` 的评论。在一个轮询间隔（默认一分钟）内，Overview 应显示一个工单。智能体确认请求，在自己的 Session 和目录里工作，并在原始会话中交付。对于飞书，在配置的群聊里提及机器人。

## 升级

1. 等待没有执行器处于回合中。
2. 备份 `<stateDir>/orchestrator.sqlite`。
3. 运行 `git pull && pnpm install && pnpm build`。
4. 运行 `scripts/dsh-superagent restart` 并检查其状态。

重启结束运行中的派工，并带着恢复说明启动替代者。团队文本和被覆盖的提示词部分被保留；没有覆盖的部分跟随新默认值。升级后审查标记为 **Default updated** 的部分。

被更新版本迁移的数据库无法被旧版本打开。因此回滚需要升级前的备份。升级 DeepSeek Harness 本身之后，重启运行中的实例，让面板加载匹配的产物。

### 智能体预设

worker 和 router 预设会在 5 秒后把 shell 命令移到后台。没有其预设，Session 无法加载。在其 Session 使用期间，保留 `cordis.patch.yml` 中的 `dsh-superagent-worker` 和 `dsh-superagent-router` 行；否则恢复会以 `agent-preset/not-found` 失败。

## 排障

设置问题以面板的语言出现在概览上，带修复链接；日志和 API 显示下面的英文文本。

| 症状 | 原因和修复 |
| --- | --- |
| `The agent GitHub CLI is not set.` | 在 GitHub 卡片上设置智能体的 CLI |
| `The agent GitHub CLI was not found.` | 设置包装器的绝对路径，或 `PATH` 上的命令名 |
| `The agent GitHub CLI cannot read its account` | 用 `auth login` 登录该包装器 |
| `@… cannot read …` 或 `@… cannot push to …` | 给智能体账号仓库写权限 |
| `The GitHub Source names no repository.` | 在 GitHub 卡片上设置仓库 `owner/name` |
| 工单以 `The prepare command failed: …` 失败 | 输出指出原因，例如错误的 clone 路径；修复命令，然后重试工单 |
| `git does not read the github.com credential configuration dsh-superagent writes` | 安装 git 2.32 或更高 |
| 工单以 `The git configuration of the repository in … sets …` 失败 | 从 clone 的 `.git/config` 移除指出的凭据或 `http.*.extraheader` 设置 |
| `… was not written by dsh-superagent` | 把指出的文件移出 `<stateDir>/worker-bin` 或 `<stateDir>/worker-gitconfig`，或选择另一个 `stateDir` |
| `The bot lark-cli allows user identities` | 把飞书来源指向一个仅机器人的配置，如上文飞书机器人 |
| `The bot lark-cli does not work.` 或 `The bot identity of app … is …` | 检查配置的 app id 和密钥 |
| `The Feishu bot is not a member of the chat.` | 在群聊设置里把机器人加入群聊；下次尝试会成功 |
| `status` 退出 2 | 某部分报告错误，重启不能修复：`errors` 指出每个部分（如管理 API 所列），带其详情；每个部分在其再次工作后清除自己的条目 |
| `status` 退出 3 | 设置等待：`setup` 说明缺失什么，概览链接到修复它的设置 |
| `status` 退出 1 | 服务没有运行、运行了两次、不应答，或其运行时报告错误；输出和最新日志说明是哪种 |
| `… timed out after 90 s`、`… was stopped by …`、`… exited with code …` 或 `… could not run: …` | 某 CLI 失败且未打印原因；以运营者用户身份手动运行该程序以查看它自己的消息 |
| 面板说 dsh-superagent 不应答 | 它在重启，或因配置未能启动；DeepSeek Harness 日志指出原因 |
| `dsh-superagent configuration has keys it does not use`（日志） | 移除指出的键；消息说明每个值现在在哪里 |
| 日志说插件与运行中的 dsh 不兼容 | 安装 `engines.dsh` 范围内的 DeepSeek Harness 版本 |
| DeepSeek Harness 升级后面板显示 "Failed to load plugins" | 重启运行中的实例 |
| 提及后毫无动静 | 检查作者在来源的允许名单上、来源已开启、且处理未暂停。在 GitHub 上，通过编辑 Issue 正文添加的提及不会被接收；在评论里提及智能体 |
