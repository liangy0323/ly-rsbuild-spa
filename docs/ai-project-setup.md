# 交给 AI 的新项目搭建说明

请用现有的 `ly-rsbuild-spa` 脚手架搭建一个能独立安装、启动和构建的 Vue 3 + TypeScript + Rsbuild SPA。先把脚手架跑通，再实现我在下方填写的项目内容。

## 项目输入

| 项目 | 本次填写 |
| --- | --- |
| 脚手架源码 | `D:/github-project/ly-rsbuild-spa`。在其他机器上，用同一份脚手架的源码目录或仓库地址替换 |
| 新项目目录 | `<填写目标目录>` |
| npm 包名 | `<填写英文包名>`，初始版本沿用 `0.1.0` |
| 浏览器标题和页面名称 | `<填写项目名称>` |
| 要实现的页面和交互 | `<填写需求>`。缺省时先保留两个示例页面 |
| UI 框架 | 缺省无 UI 框架。可选 Element Plus 或 Vant |
| 公开 API 地址 | 缺省为空。开发和生产地址按各自环境填写 |
| 部署基路径 | 缺省 `/` |
| 离线缓存 | 缺省关闭 |

目标目录是必填项。包名和标题空缺时，可从目标目录名推导合法的 npm 包名和可读标题。访问源码后，请先读它的 `AGENTS.md`、`README.md` 和 `package.json`。如果源目录不可访问，请说明需要的脚手架位置。目标中已有文件时，先核对同名文件并保留已有工作。下面的操作都在目标目录执行。

## 复制并运行基础工程

1. 从源码复制 `build/`、`src/`、`public/`、`tests/`、`docs/` 中的公开文档、`.rules/`、`.husky/pre-commit` 和根目录的工程文件。根目录文件包括 `AGENTS.md`、`README.md`、`package.json`、`pnpm-lock.yaml`、`pnpm-workspace.yaml`、`tsconfig.json`、`env.d.ts`、`rstest.config.ts`、`index.html`、`.gitignore`、`.prettierrc` 及 `.env.example`、`.env.development`、`.env.production`。对环境文件只保留适用于新项目的公开配置。复制范围以这份清单为准，`docs/local-*.md` 属于源项目本地笔记。
2. 使用 Node.js 22 和 pnpm 11。独立新项目可以先运行 `git init`，使 `pnpm install` 的 `prepare` 能安装 Husky 钩子。依赖沿用脚手架的精确版本与锁文件，先运行 `pnpm install --frozen-lockfile`。
3. 修改 `package.json` 的 `name`，以及 `build/rsbuild.base.config.ts` 的 `html.title`、`src/layouts/index.vue` 的可见标题。版本清单和 Worker 缓存标识从包名读取。按新项目内容更新 README 和应用图标。
4. 用 `pnpm dev` 打开 `/` 和 `/components`。首页显示当前环境，组件页用来验证 `LucideIcon`、`CircleLoading` 及路由跳转。先完成这一步，再添加业务或更换示例页。

## 按需求扩展

- **页面与路由**。应用在 `src/index.ts` 挂载。公共布局放在 `src/layouts/`，页面放在 `src/views/<业务域>/`。路由定义放在 `src/router/modules/`，由 `src/router/index.ts` 统一注册。沿用 `import()` 懒加载，按需设置路由标题。在需要复用的基础组件上沿用 `src/components/base/` 自动扫描。
- **UI 框架**。选择 Element Plus 或 Vant 时，按 [UI 框架接入指南](ui-integration.md) 增加依赖和解析器，更新锁文件。需要 Element Plus 浅色和暗色时，再按 [多主题实施指南](element-plus-themes.md) 接入。缺省使用原生标签和现有基础组件。
- **环境与后端**。`APP_ENV` 应与命令的环境模式一致。填写公开的 `API_BASE_URL` 后，继续复用 `src/env/index.ts` 和 `src/utils/`。Rsbuild 注入的变量会进入浏览器产物，服务端密钥由服务端环境管理。
- **部署与离线**。默认部署在根路径。若指定子路径，将公共配置中的 `server.base` 和 `output.assetPrefix` 设为同一部署路径，并让版本清单地址及 Worker 作用域使用同一前缀。路由从 Rsbuild 的 `import.meta.env.BASE_URL` 获取基路径。静态服务器需为前端路由提供对应 `index.html` 回退。需要离线时，在生产环境文件把 `OFFLINE_CACHE_ENABLED` 设为 `true` 并重新构建。

## 验收与交付

在目标项目运行以下命令。使用自动导入的 UI 框架时，先完成一次开发编译或生产构建以生成类型声明。

```bash
pnpm typecheck
pnpm test
pnpm build
pnpm preview
```

在开发和生产预览中打开首页和另一个路由，验证导航、直接输入深层地址及刷新。若已用真实页面替换示例，验证相同的访问路径和对应业务交互。检查 `dist/version.json` 中的应用名是否匹配新项目；只有启用离线缓存时才检查离线页和 Worker 注册。

交付时附目标目录、运行命令、已完成的页面和实际验收结果。未得到验证的环境或部署行为单独写明。需要提交、推送或部署时，按我明确给出的范围执行。
