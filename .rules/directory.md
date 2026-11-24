# 目录组织规范

## 项目边界

本仓库提供 Vue 3 + TypeScript + Rsbuild 单页应用脚手架。应用入口为 `src/index.ts`，根组件为 `src/App.vue`。新增目录按实际功能创建，以下结构包含接入相应能力后的推荐位置。

## 目标结构

```text
.
├── .rules/                          # 编码与项目规则
├── build/                           # 公共、开发和生产构建配置
│   ├── rsbuild.base.config.ts
│   ├── rsbuild.dev.config.ts
│   ├── rsbuild.prod.config.ts
│   └── runtime/                     # 环境、版本检测和离线缓存
├── docs/                            # 接入说明和项目文档
├── public/                          # 保持原名、不经源码构建处理的资源
├── tests/
│   ├── unit/                        # Rstest 单元测试
│   └── e2e/                         # 接入端到端测试后创建
├── src/
│   ├── index.ts                     # 创建和挂载应用
│   ├── App.vue                      # 根组件
│   ├── bootstrap/                   # 应用启动装配和全局副作用
│   ├── api/                         # API 地址、请求函数与边界适配
│   ├── assets/                      # 图片、字体和样式
│   ├── components/
│   │   ├── base/                    # 基础组件，可自动扫描导入
│   │   └── business/                # 已有跨页面复用的业务组件
│   ├── composables/                 # 跨页面复用的组合式能力
│   ├── constants/                   # 按业务域划分的常量
│   ├── directive/                   # 自定义指令
│   ├── feedback/                    # 消息、确认等 UI 反馈适配
│   ├── layouts/                     # 应用布局
│   ├── router/                      # 路由入口、守卫及业务模块
│   ├── services/                    # 按后端服务划分请求实例
│   ├── stores/                      # Pinia 实例及 Store 模块
│   ├── types/                       # 公共类型
│   ├── utils/                       # 通用工具
│   └── views/                       # 按业务域组织页面
├── AGENTS.md
├── env.d.ts                         # 环境与全局声明
├── index.html
├── package.json
├── pnpm-lock.yaml
├── pnpm-workspace.yaml
└── tsconfig.json
```

`auto-imports.d.ts` 和 `components.d.ts` 由构建插件生成。

Rsbuild 配置统一维护在 `build/`：`rsbuild.base.config.ts` 提供公共配置，开发与生产配置分别合并公共配置。启动、构建和预览命令通过 `--config` 指定对应文件，根目录不设置 `rsbuild.config.ts`。

## 页面私有结构

```text
src/views/live/manage/
├── index.vue
├── index.scss                      # 样式达到拆分条件时创建
├── components/
│   └── live-form-dialog/
│       ├── index.vue
│       └── live-form-dialog.scss
└── composables/
    ├── use-live-list.ts
    └── use-live-actions.ts
```

- 页面目录和普通代码文件使用 kebab-case；页面入口使用 `index.vue`。
- 多文件组件采用 kebab-case 目录加 `index.vue`；简单组件直接使用 `kebab-case.vue`。
- 只服务一个页面的弹窗、列表、选择器和组合式逻辑保留在页面附近。
- 页面入口保留初始化顺序和跨 composable 的联动，不再为简单组合创建无明确职责的总 hook。
- 跨页面稳定复用后再提升组件或逻辑，避免提前公共化。
- 类型按业务域集中到 `src/types/modules/`，仅组件私有 Props、Emits 留在组件内部。
- 页面私有的筛选项、表格列和专用解析留在业务目录，文件名体现具体职责，例如 `training-list-config.ts`、`parse-drawing-xml.ts`。多个页面共用的业务表格及其列配置可以随组件维护，页面通过具名请求函数提供数据。
- 跨业务域共享的选择器应通过明确参数和事件复用，不从一个页面导入另一个页面的私有组件。

## API 双文件与业务分域

```text
src/api/modules/live/
├── manage/
│   ├── api.ts                      # URL 常量
│   └── interface.ts                # 请求函数及边界适配
└── statistics/
    ├── api.ts
    └── interface.ts
```

- 业务域较小时可直接在 `live/` 下放双文件，不强制再拆二级目录。
- 按业务能力组织接口，平台专属接口可在对应业务域内保留适配子模块。
- `src/api/index.ts` 对外导出请求函数，不向业务页面暴露 URL 常量、服务实例或内部适配细节。
- 各模块导出使用明确且不冲突的命名，不把多个域的 `getList` 无差别聚合。
- API 显式导入 `@/services/service-api` 等具体服务实例。
- 请求、响应和分页原始类型维护在 `src/types/modules/`，按接口实际契约定义。
- 不因前端业务目录重组而改写后端 URL、字段或权限码。

## Barrel 规则

- `api/`、`constants/`、`types/`、`utils/`、`stores/` 等普通模块可采用 `index.ts + modules/`。
- `composables/` 不设置 barrel，调用方从具体 `use-*.ts` 文件导入。
- `services/` 不设置总 barrel，具体后端依赖保持可见。
- Vue 组件不为形式增加 `index.ts`；业务组件显式导入具体入口。
- 单文件能力不为形式创建 `modules/`；`env.d.ts`、配置文件和生成声明保持约定名称。
- 默认使用命名导出；框架、构建工具要求的配置默认导出以及根组件不受普通业务模块限制。

## 公共能力边界

- `utils/` 放纯格式化、转换、校验及不依赖框架的浏览器工具，例如通用脚本、样式资源加载；不放消息提示、导航行为或业务流程编排。SDK 专属资源地址、工厂检查和实例生命周期由对应组件或 composable 管理。
- `feedback/` 收敛消息提示、确认等 UI 反馈适配。
- `bootstrap/` 负责跨层装配；`index.ts` 保留创建和挂载应用的主流程。
- `stores/` 只维护需要跨页面共享的状态，不把所有列表、弹窗表单及请求结果放入 Store。
- `services/` 的服务实例按实际对接的后端增加。
- SDK、图表、上传等能力在接入时按职责确定模块位置。
- 命令式预览或覆盖层在有真实复用时再增加 `components/overlay/`，关闭后卸载与释放资源。
- 静态资源按需放入 `src/assets/` 或 `public/`，主题和组件覆盖使用单一入口，避免重复注入。

## 旧代码迁移参考

| 原有组织方式           | 目标组织方式                                 | 迁移边界                                     |
| ---------------------- | -------------------------------------------- | -------------------------------------------- |
| 入口中集中的全局副作用 | `src/bootstrap/`                             | 按职责迁移启动逻辑，入口保留创建和挂载应用   |
| `src/hooks/`           | `src/composables/` 或页面私有 `composables/` | 分页、表单与业务流程按复用范围拆分           |
| `src/api/<module>/`    | `src/api/modules/<domain>/`                  | 保留 URL 和后端参数契约，集中类型            |
| 全目录组件扫描         | 仅扫描 `components/base/`                    | 迁移页面时补齐业务、私有及布局组件显式导入   |
| 全局与页面散落的类型   | `src/types/modules/`                         | 区分 HTTP 原始契约和前端统一视图模型         |
| `assets/scss/`         | `assets/styles/`                             | 迁移 SCSS 引用，不把样式重命名等同于视觉重做 |
| 服务直接调用 UI 或跳转 | `services/` 协议与 `bootstrap/` 装配         | 保留登录与错误识别能力，调整依赖方向         |

## 测试布局

- `tests/unit/` 按服务、导航、分页及业务模块组织 Rstest 测试。
- 接入端到端测试后，在 `tests/e2e/` 按关键用户流程组织用例。
- 测试配置与 setup 按测试框架分离，匹配规则互不重叠。
- 测试数据与 mock 归属对应测试范围，不能把测试替身放进生产 API 模块。
- 配置引用真实实现路径，不为满足目录树创建空测试文件。

## 自定义指令边界

- 注册入口为 `src/directive/index.ts`，指令目录使用 kebab-case，例如 `src/directive/has-permissions/index.ts`。
- 使用 Vue 3 的 `ObjectDirective` 类型与生命周期；监听、定时器和响应式 effect 必须在卸载时清理。
- 权限来源由注册阶段注入，禁止读取 Pinia 持久化键或在模块顶层隐式创建 Store。
- 权限未就绪时默认隐藏，不移除 Vue 管理的 DOM；异步授权、撤权与绑定值变化都必须重新判断。
- 组件已提供公开事件时优先使用，避免通过私有 DOM 选择器监听交互。
