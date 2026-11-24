# ly-rsbuild-spa

可独立运行的 Vue 3 + TypeScript + Rsbuild 单页应用脚手架。提供公共构建配置、基础组件、版本更新提示、通用工具以及测试和 Git Hooks。UI 框架按项目选择，接入步骤见 [UI 框架接入](docs/ui-integration.md)。

将搭建任务交给其他 AI 时，可以使用 [新项目搭建说明](docs/ai-project-setup.md)。

## 快速开始

使用 Node.js 22 和 pnpm 11。

```bash
pnpm install
pnpm dev
```

开发服务器默认使用 `http://localhost:3000`。可用 `pnpm dev --port 3100` 指定端口。开发、构建和预览命令均使用 `build/` 下的配置。

示例路由为 `/`（首页与环境信息）和 `/components`（LucideIcon 与 CircleLoading）。路由从 [src/router/index.ts](src/router/index.ts) 统一注册，按模块定义并懒加载页面。

| 命令                | 用途                                 |
| ------------------- | ------------------------------------ |
| `pnpm dev`          | 启动开发服务器                       |
| `pnpm typecheck`    | 检查 TypeScript 和 Vue 类型          |
| `pnpm format`       | 使用 Prettier 格式化代码和文档       |
| `pnpm format:check` | 检查代码和文档是否符合 Prettier 格式 |
| `pnpm test`         | 运行 Rstest 单元测试                 |
| `pnpm test:watch`   | 监听改动并重新运行相关测试           |
| `pnpm build`        | 构建到 `dist/`                       |
| `pnpm preview`      | 本地预览生产构建                     |
| `pnpm clean`        | 清理 `dist/`                         |

Git 仓库运行 `pnpm install` 后，Husky 通过 `prepare` 配置钩子。提交前会运行类型检查和 Rstest。Rstest 复用公共构建配置，在 Node 环境运行 `tests/unit/` 下的测试。

## 环境与部署

| 变量                    | 用法                                                                               |
| ----------------------- | ---------------------------------------------------------------------------------- |
| `APP_ENV`               | 由对应的 `.env.*` 提供，应与构建命令的环境模式一致                                 |
| `API_BASE_URL`          | 接入后端时填写公开 API 地址，缺省为空                                              |
| `OFFLINE_CACHE_ENABLED` | 默认 `false`。在生产配置设为 `true` 并重新构建即可启用。开发服务器始终跳过离线缓存 |

业务代码从 [src/env/index.ts](src/env/index.ts) 调用 `getAppEnv()`、`isOfflineCacheEnabled()` 和 `isDev()`。其中 `isDev()` 使用 Rsbuild 内置的构建标志，`isOfflineCacheEnabled()` 返回实际生效的布尔值。

构建后的 `dist/` 可部署到静态服务器。启用 History 路由时，服务器需将前端路由回退到 `index.html`。版本检查会读取 `version.json`；部署时让 HTML、版本清单和 `sw.js` 可及时更新，带内容哈希的静态资源可设置长期缓存。

## 基础模块

| 模块                                                          | 用法                                                                                                |
| ------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| [应用更新](src/bootstrap/app-update.ts)                       | 应用启动后检查版本，发现更新时提示是否刷新。自定义设置页可注入 `APP_VERSION_CHECK_KEY` 触发手动检查 |
| [LucideIcon](src/components/base/lucide-icon/index.vue)       | 使用 `<lucide-icon icon="House" :size="20" />`，也可传入按需导入的图标组件                          |
| [CircleLoading](src/components/base/circle-loading/index.vue) | 使用 `<circle-loading :size="24" />`，颜色继承主题主色或当前文字色                                  |
| [工具入口](src/utils/index.ts)                                | 通过 `@/utils` 导入 JSON 与查询参数处理、深拷贝、网络监听和 localStorage 工具                       |

网络提示可在应用启动时按需接入。`initNetworkNotifier((message, status) => { ... })` 返回清理函数，可交给 `app.onUnmount`。回调可以对接所选 UI 框架的消息组件。

Element Plus 项目可以按 [多主题实施指南](docs/element-plus-themes.md) 接入浅色、暗色与首屏偏好恢复。

复制工程时修改 `package.json` 的应用名和版本，以及公共构建配置的 HTML 标题。版本清单和离线缓存使用包名标识应用。开发约定见 [AGENTS.md](AGENTS.md)。
