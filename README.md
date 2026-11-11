# company-standard-agent(公司标准 AI 工作助手 · Hermes Profile 分发包)

一条命令给员工装好公司标准 agent:统一人格(SOUL)+ 统一模型配置 + Windows 避坑技能 + 每周技能自动更新任务。

## 员工安装

```bash
hermes profile install github.com/haylenfoust-svg/company-standard-agent --alias
```

安装后填自己的 API Key(向公司管理员申请):

- Windows 上 profile 目录:`%LOCALAPPDATA%\hermes\profiles\company-standard-agent\`
- 把该目录下的 `.env.EXAMPLE` 复制为 `.env`,填入 `HERMES_CUSTOM_QWEN_API_KEY=***

然后直接使用:

```bash
company-standard-agent          # 或 hermes -p company-standard-agent
```

## 更新

```bash
hermes profile update company-standard-agent
```

你的记忆、会话、API Key 不会被更新覆盖;你自己改过的 config.yaml 也默认保留(需要恢复出厂配置时加 `--force-config`)。

## 包含内容

| 文件 | 说明 |
|---|---|
| `distribution.yaml` | 清单:名称、版本、所需环境变量 |
| `SOUL.md` | 公司标准人格:中文、结论先行、先检查后建议、安全底线 |
| `config.yaml` | 统一模型:qwen3.8-max @ 阿里云企业端点(key 走各自 .env,不入库) |
| `skills/windows-agent-quirks/` | Windows 避坑技能(首次安装兜底快照) |
| `cron/jobs.json` | 每周一 9:00 自动 `hermes skills update`;**安装后默认暂停**,确认后启用:`hermes -p company-standard-agent cron list` → resume |

## 与 company-hermes-skills 仓库的关系

技能的**权威源**是 `github.com/haylenfoust-svg/company-hermes-skills`(tap 仓库)。
建议员工同时执行:

```bash
hermes skills tap add github.com/haylenfoust-svg/company-hermes-skills
```

这样 `hermes skills update` 能持续拿到最新技能;本分发包里的 skills/ 只是首装快照。

## 维护者须知

- 发新版:改 `distribution.yaml` 的 version → `git tag vX.Y.Z` → push --tags;员工 `hermes profile update` 即同步
- **绝不提交**:`.env`、auth.json、memories/、sessions/(.gitignore 已挡,提交前仍要 `git status` 复查)
