# weixin-access

这是一个基于 `@tencent-weixin/openclaw-weixin-cli` 思路整理出来的 OpenClaw 微信接入解析项目。它的重点不是替代 `wechat-claw`，而是把“微信如何接入 OpenClaw”这件事拆开讲清楚，方便后续复用到更多平台。

如果你只是想尽快把微信接入 OpenClaw，优先使用 `wechat-claw` / `@tencent-weixin/openclaw-weixin-cli` 这一套现成方案；如果你想理解微信接入链路、消息适配方式、登录流程和宿主集成原理，这个仓库更适合拿来阅读和二次开发。

## 项目解析

这个项目用于拆解微信接入 OpenClaw 的实现过程，重点是渠道接入、消息适配、会话复用和结果回传。

如果你想直接使用微信接入能力，优先使用 `wechat-claw` 或 `@tencent-weixin/openclaw-weixin-cli`；如果你想看接入原理、修改通道逻辑，阅读这个仓库会更直接。

## 主要能力

- 建立并维护 AGP WebSocket 连接，支持重连、心跳和连接状态上报
- 将 AGP prompt 转成 OpenClaw 可复用的消息上下文，复用已有会话和路由逻辑
- 处理 `session.prompt` / `session.cancel`
- 支持流式输出、工具调用透传、最终回复发送
- 在连接抖动时将最终回复写入本地队列，恢复连接后自动补发
- 缓存并补齐 `guid` / `uid` 等上报上下文，便于外部统计

## 目录说明

```text
.
├── index.ts                  # 插件入口，注册 channel / gateway / status
├── common/
│   ├── runtime.ts            # 保存 OpenClaw runtime
│   ├── message-context.ts    # 构建 OpenClaw 内部消息上下文
│   ├── report-data.ts        # 连接事件上报与上下文缓存
│   └── agent-events.ts       # Agent 事件桥接
├── websocket/
│   ├── message-handler.ts    # prompt/cancel 主处理流程
│   ├── message-adapter.ts    # AGP 消息到内部格式的适配
│   ├── terminal-response-queue.ts # 最终回复的本地补发队列
│   └── types.ts
└── openclaw.plugin.json      # 插件元数据
```

## 消息链路

1. OpenClaw 加载插件并调用 `register()`。
2. `index.ts` 读取 `channels.wechat-access` 配置并启动 WebSocket 客户端。
3. 收到 `session.prompt` 后，`websocket/message-handler.ts` 将 AGP payload 转成统一消息上下文。
4. 插件调用 OpenClaw Agent 处理请求，并把流式结果实时推回服务端。
5. 终态回复发送失败时，写入本地 WAL 队列；连接恢复后自动补发。

## 先教怎么接入 OpenClaw

如果你的目标是先把微信接到 OpenClaw，建议先走官方 CLI 流程。

### 环境要求

- Node.js >= 18
- npm >= 9
- 已安装并配置好 OpenClaw

### 1. 安装 OpenClaw 微信插件

运行：

```bash
npx -y @tencent-weixin/openclaw-weixin-cli@latest install
```

这一步通常会完成以下事情：

- 下载最新版本的 `@tencent-weixin/openclaw-weixin-cli`
- 安装 `@tencent-weixin/openclaw-weixin` 插件
- 将插件放到 OpenClaw 扩展目录
- 重启 OpenClaw Gateway
- 在终端中展示二维码，完成微信扫码授权

### 2. 验证安装

运行：

```bash
openclaw extensions list
```

如果安装成功，扩展列表里应该能看到 `openclaw-weixin`。

### 3. 微信账号登录

如果需要重新登录或刷新会话，运行：

```bash
openclaw channels login --channel openclaw-weixin
```

然后按终端中的二维码提示扫码即可。


## 当前仓库的开发方式

这个仓库不是那个“一键扫码安装”的 CLI，而是更底层的渠道插件实现与解析代码。

如果你要研究或扩展这套接入逻辑，可以在当前仓库里本地开发：

```bash
npm install
npm run typecheck
npm run build
```

适合的场景包括：

- 想看微信消息是怎么被转换成 OpenClaw 上下文的
- 想研究连接、重连、补发和状态上报逻辑
- 想把同样的接入方式复用到其他 IM 平台

## 配置示例

插件运行依赖 OpenClaw 主配置中的渠道参数，至少需要 `token` 和 `wsUrl`：

```json
{
  "channels": {
    "wechat-access": {
      "token": "your-token",
      "wsUrl": "wss://example.com/agp/ws",
      "guid": "device-guid",
      "userId": "wechat-user-id"
    }
  }
}
```

其中：

- `token`：AGP 服务鉴权令牌
- `wsUrl`：上游 WebSocket 地址
- `guid` / `userId`：用于消息信封和连接事件上报

## 原理说明

从原理上看，这个项目主要做了 4 件事。

### 1. 接收微信侧消息

外部通路把消息送到插件，当前仓库里主要通过 WebSocket 接收 `session.prompt` 和 `session.cancel`。

### 2. 转成宿主能理解的消息格式

`websocket/message-adapter.ts` 会把通路侧 payload 转成内部统一消息格式，再交给 `common/message-context.ts` 构造成 OpenClaw 标准上下文。

这一步是最关键的，因为它决定了：

- 会话怎么分配
- 消息怎么路由到具体 Agent
- 历史上下文怎么复用
- UI 如何识别消息来自外部渠道

### 3. 复用 OpenClaw 现有能力

消息一旦进入标准上下文，后面的处理就不再是“微信专属逻辑”了，而是直接复用宿主已有的：

- Agent 调度
- 路由解析
- 会话存储
- 回复格式化
- 工具调用和流式输出

这也是为什么这类项目很适合推广到其他平台。

### 4. 回写结果并处理异常

Agent 输出结果后，插件再把结果写回外部通路；如果当时断线，就先进入本地补发队列，等连接恢复后再发送。

## 整体职责边界

集成完成后，职责边界大致是：

- `wechat-claw` / OpenClaw 宿主：负责 Agent、模型、会话、路由、任务执行
- 微信接入插件：负责登录、接入、消息收发、协议适配
- 当前仓库：负责把这套接入过程拆解为可理解、可迁移、可复用的实现

所以这不是一个“只服务于 OpenClaw 的特例项目”，而是一种“外部平台如何接入 Agent 宿主”的参考实现。

## 当前状态

- 仓库还没有自动化测试
- 适合继续补充的方向包括：测试覆盖、配置文档、错误码说明和本地联调手册

