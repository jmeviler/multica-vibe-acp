# Multica ↔ Mistral Vibe ACP

一个薄的 stdio 适配层：让 Multica 通过 ACP 使用 Mistral Vibe，并支持模型目录和模型切换。

```text
Multica（Hermes 兼容接入类型）
             ↕
      multica-vibe-acp
             ↕
    Mistral 官方 vibe-acp
             ↕
       Vibe 配置的后端
```

**这是非官方集成，不是 Hermes Agent，也不是模型 API 代理。**
Vibe CLI 与 `vibe-acp` 由 [Mistral AI](https://github.com/mistralai/mistral-vibe) 维护；
这个仓库只负责 Multica 与 Vibe 之间的协议适配。

## 做了什么

- 将 Multica 的 `session/set_model` 转为 Vibe 的 `session/set_config_option`，配置项为 `model`。
- 原样透传其他 JSON-RPC 消息和请求 ID；模型目录来自 Vibe 的真实 ACP 响应。
- 子进程 stdout、stderr 直接继承，不伪造模型列表、成功响应或退出状态。
- 当 Multica 提供 `HERMES_HOME/skills` 时，将任务目录的 `.vibe/skills` 链接到它。
- 复用旧聊天目录时更新旧的技能符号链接；不删除实际目录或技能内容。

真实模型调用、授权、上下文窗口、自动压缩和工具执行仍由 Vibe 及其后端负责。

## 运行要求

- Linux 或 macOS；使用 Unix stdio 管道。
- Python 3.12+；适配脚本只使用标准库，无额外 Python 依赖。
- 已安装并完成授权的 Mistral Vibe，`vibe-acp` 在 daemon 的 PATH 中。
- 支持自定义 ACP 运行时 profile 的 Multica。

已有部署使用 Multica CLI **0.12.1**、Vibe **3.1.4** 完成过模型目录、两种模型真实任务、
旧聊天连续两轮回复验证；这不是所有版本的兼容性保证。

## 使用

先按官方说明安装、授权 Vibe；本仓库不提供安装器，不保存或复制登录凭据。

将本仓库脚本加入执行 daemon 的机器的 PATH，例如：

```bash
mkdir -p "$HOME/.local/bin"
ln -s "$PWD/multica-vibe-acp" "$HOME/.local/bin/multica-vibe-acp"
export PATH="$HOME/.local/bin:$PATH"
multica-vibe-acp --version
```

如果 Vibe 不在 PATH 中，在 **daemon 的启动环境**指定实际路径：

```bash
export VIBE_ACP="/absolute/path/to/vibe-acp"
```

仅支持 `acp`、`--version` 和 `-v` 三种调用；不要传入 Hermes Agent 的专用参数。

### 接入 Multica

让 daemon 继承上述 PATH 和可选的 `VIBE_ACP`，为执行机器创建专用 profile：

```bash
multica --workspace-id <workspace-id> runtime profile create \
  --daemon-id <daemon-id> \
  --runtime-type hermes \
  --command-name multica-vibe-acp \
  --display-name "Mistral Vibe ACP"
```

这里的 `hermes` 是 Multica 的协议接入类型，不表示安装或运行 Hermes Agent。
本方案不需要设置 `MULTICA_HERMES_PATH`，避免同时注册一个额外的原生 Hermes 类型条目。

等待 daemon 发现 profile 后，通过 `runtime list` 查找对应运行时，再绑定已有 Agent：

```bash
multica --workspace-id <workspace-id> runtime list
multica --workspace-id <workspace-id> agent update <agent-id> \
  --runtime-id <runtime-id>
```

模型选项由 Vibe 的 `session/new` 响应提供。界面选中某个模型后，
Multica 发送 `session/set_model`，此脚本转译请求，由真实 Vibe 应用设置。
不把服务端完整模型目录直接当成当前账号可用列表。

## 边界与限制

- 不新增 HTTP 服务、外网端口、API Key 存储或账号体系。
- 不把 Multica 聊天记录的字符大小当成 Vibe 的实际上下文 token 数。
- 遇到无效 JSON、无效模型请求、已有真实技能目录冲突或 Vibe 启动失败，直接报错。
- `HERMES_HOME` 中的技能目录由 Multica 管理；Vibe 登录配置仍由 Vibe 管理。
- 不包含任何具体服务器的启动脚本、工作区 ID 或认证文件。
