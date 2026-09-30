# dsh-raindrop-preset

一个给 DeepSeek Harness 用的 bundle. 装进 profile 之后会多出一个 `Raindrop` agent 预设: 随发行 `standard` 预设的完整工具集, 外加 Raindrop.io 书签的 MCP 工具.

## 为什么需要它

Raindrop 的 MCP 端点只认 `Authorization: Bearer <access token>`, 而 DSH 的 `streamable-http` 传输不走 OAuth 交互流程, 所以只能配静态 token.

这条配置若留在 profile 的 patch 里, 它属于 host plane, 于是**每个会话**都会带上那 20 多个 `mcp__raindrop__*` 工具, 白白占掉每次请求的提示词空间. 这个 bundle 把它挪进一个独立的预设, 只在选了 `Raindrop` 的会话里生效.

## 前置: 一个 Raindrop access token

1. 打开 <https://app.raindrop.io/settings/integrations>
2. 先建一个 Application (还没有的话)
3. 打开这个应用的设置, 复制里面的 **Test token**

Test token 就是 access token, UUID 形态, 形如 `ae261404-11r4-47c0-bce3-e18a423da828`. 只访问自己的账号时它够用, 不必走完整 OAuth 流程 —— OAuth 换来的 access token 两周过期, 静态配置里没法续期, 而 Test token 不过期.

把值放进登录 shell 的环境变量:

```shell
export RAINDROP_MCP_TOKEN=粘贴token
```

fish 用户想让它持久化, 用 `set -Ux RAINDROP_MCP_TOKEN 粘贴token`.

桌面应用只在**启动时**读一次登录 shell 环境, 所以设完必须完整重启 DSH, 刷新窗口不够.

## 装法

在 Harness 的插件页 (或 `plugin_manager` 的 `install_bundle`) 里填:

```text
github:azazo1/dsh-raindrop-preset
```

装完开一个新会话, 在预设选择器里选 `Raindrop`. 选择器本身受通用设置里的 "代码工作工具" 开关控制.

预设只能在**空白会话**里选: 注册表会拒绝给已经跑过轮次的会话换预设.

## 内容

| 文件 | 作用 |
|---|---|
| `package.json` | 声明 `dsh.bundle.patch`; 不额外依赖任何包, `@deepseek-ai/dsh-mcp-client` 是 Desktop 安装闭包自带 |
| `cordis.patch.yml` | 插入 `preset-raindrop` 声明: 抄自同版本 `standard` 预设的完整插件列表, 末尾追加 Raindrop MCP 行 |

`cordis.patch.yml` 里的插件列表是随发行 `standard` 的**拷贝**, 不是引用. 官方升级后如果 standard 增删了行, 这份拷贝不会跟着变, 要手工同步 (对照 `packages/bundle/web-app/presets/standard.patch.yml`, 它也在 `app.asar` 的 `dsh/node_modules/@deepseek-ai/dsh-web-app/presets/` 下).

## 与 host 级配置不能并存

MCP 的 `serverName` 在同一个注册作用域内唯一. 如果 profile patch 里还留着 host 级的那条 `mcp-raindrop`, 一个会话会同时看到两份同名的 `mcp__raindrop__find_bookmarks`, 模型侧直接冲突. 换上这个 bundle 之前, 先把 host 级那条停用或删掉.
