# ly-rsbuild-spa 开发约定

本仓库是 Vue 3 + TypeScript + Rsbuild 单页应用脚手架。应用从 [src/index.ts](src/index.ts) 启动，公共、开发和生产构建配置统一放在 [build/](build/)。

开发时按任务查阅下表对应规则。目录树中的后续能力按项目需求接入，涉及 UI 库、鉴权、服务层和持久化的规则在实际接入后适用。UI 框架的依赖、导入和样式组织见 [UI 框架接入](docs/ui-integration.md)。

| 任务                   | 规则                                                                                                           |
| ---------------------- | -------------------------------------------------------------------------------------------------------------- |
| 目录与文件命名         | [目录](.rules/directory.md) · [命名](.rules/naming.md)                                                         |
| Vue 组件与 TypeScript  | [Vue 3](.rules/vue3.md) · [TypeScript](.rules/typescript.md)                                                   |
| 请求、状态与缓存       | [Axios](.rules/axios.md) · [Pinia](.rules/pinia.md) · [存储](.rules/storage.md)                                |
| 注释、异常与完成前自查 | [注释风格](.rules/comment-style.md) · [错误处理](.rules/error-handling.md) · [自查清单](.rules/code-review.md) |

验证命令以 [package.json](package.json) 为准。按改动范围选用 `pnpm format:check`、`pnpm typecheck`、`pnpm test` 或 `pnpm build`，页面改动可以通过 `pnpm dev` 验证。格式问题运行 `pnpm format` 按 [.prettierrc](.prettierrc) 修复。
