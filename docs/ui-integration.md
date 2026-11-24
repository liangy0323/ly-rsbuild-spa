# UI 框架接入

基础脚手架默认无 UI 框架。按项目需要选择 Element Plus（桌面和后台页面）或 Vant（移动端页面），两套示例可以独立使用。组件解析都加在现有的 `build/rsbuild.base.config.ts` 中，开发、生产和 Rstest 复用这份公共配置。

## Element Plus

参考 `admin-ddm-v2` 的 `build/rsbuild.base.config.ts`、`src/assets/element-plus/` 和 `src/feedback/modules/message.ts`。参考项目用 Rspack 自动导入组件和命令式 API，并在 Sass 编译时统一设置品牌变量。

### 安装与目录

```bash
pnpm add -E element-plus@2.14.8
```

```text
src/
├── assets/
│   ├── element-plus/
│   │   ├── index.scss             # 供 Sass 加载器注入的主题入口
│   │   └── modules/
│   │       └── var.scss           # Element Plus 编译期变量
│   └── styles/
│       └── index.scss             # 需要全局生效的应用样式，按需创建
├── components/base/              # 应用自己的基础组件
├── feedback/                     # 消息、确认等 UI 适配，按需创建
└── index.ts                      # 统一引入应用全局样式
```

### 增加解析器

在 `build/rsbuild.base.config.ts` 的导入区加入 `import { ElementPlusResolver } from 'unplugin-vue-components/resolvers';`。为已有插件补充 `resolvers`，沿用基础组件扫描目录和声明文件生成路径。

```ts
AutoImport({
  imports: ['vue', 'vue-router', 'pinia'],
  resolvers: [ElementPlusResolver({ importStyle: 'sass' })],
  dts: resolve(process.cwd(), './auto-imports.d.ts'),
}),
Components({
  dirs: [resolve(process.cwd(), './src/components/base')],
  resolvers: [ElementPlusResolver({ importStyle: 'sass', directives: false })],
  dts: resolve(process.cwd(), './components.d.ts'),
}),
```

若需要 `v-loading` 等 Element Plus 指令，将组件解析器的 `directives` 改为 `true`。按需导入时保持现有 Vue 启动入口，由解析器加载使用到的组件、API 和样式。

### 接入 Sass 主题

把公共配置中已有的 `pluginSass()` 替换为以下调用。

```ts
pluginSass({
  sassLoaderOptions: {
    additionalData: '@use "@/assets/element-plus/index.scss" as *;',
  },
}),
```

`src/assets/element-plus/index.scss` 转发主题配置入口。

```scss
@use './modules/var.scss';
```

`src/assets/element-plus/modules/var.scss` 在组件主题编译前配置品牌色。

```scss
@forward 'element-plus/theme-chalk/src/common/var.scss' with (
  $colors: (
    'primary': (
      'base': #2f6feb,
    ),
  )
);
```

注入入口用于 Sass 配置。会输出 CSS 的全局覆盖、应用语义变量及暗色 `--el-*` 覆盖统一放在 `src/assets/styles/`，由 `src/index.ts` 引入一次。需要深浅色切换时，可沿用参考项目在 `html[data-theme='dark']` 上覆盖变量的做法，让挂载到 `body` 的弹层也继承主题。只使用预编译默认主题时，两个解析器的 `importStyle` 可改为 `'css'`，沿用默认 `pluginSass()`。

### 最小验证页

把下面内容放进 `src/views/components/index.vue` 或其他临时路由页，确认按钮显示主题色，点击时弹出消息。

```vue
<script setup lang="ts">
const showReady = (): void => {
  ElMessage.success('Element Plus 已接入');
};
</script>

<template>
  <el-button type="primary" @click="showReady">显示消息</el-button>
</template>
```

[Element Plus 按需导入](https://element-plus.org/zh-CN/guide/quickstart.html) · [Element Plus 主题定制](https://element-plus.org/zh-CN/guide/theming.html)

接入浅色和暗色主题时，继续阅读 [Element Plus 多主题实施指南](element-plus-themes.md)。

## Vant

参考 `app-client-web` 的 `packages/build-config/src/config/modules/base.ts` 和 `packages/ui/src/styles/`。参考项目用同一个 Vant 解析器处理命令式 API 和模板组件，并使用 `--van-*` 变量配置主题。独立 SPA 可以直接在本项目的公共配置中采用同样的做法。

### 安装与目录

```bash
pnpm add -E vant@4.10.2
pnpm add -DE @vant/auto-import-resolver@1.3.0
```

```text
src/
├── assets/styles/
│   ├── index.scss                 # 全局样式入口
│   └── modules/
│       └── theme.scss             # 应用语义变量及 Vant 覆盖
├── components/base/              # 页面共用的基础组件
├── feedback/                     # Toast、Dialog 等 UI 适配，按需创建
└── index.ts                      # 引入全局样式
```

### 增加解析器和主题

在 `build/rsbuild.base.config.ts` 的导入区加入 `import { VantResolver } from '@vant/auto-import-resolver';`，为现有两个插件增加解析器。

```ts
AutoImport({
  imports: ['vue', 'vue-router', 'pinia'],
  resolvers: [VantResolver()],
  dts: resolve(process.cwd(), './auto-imports.d.ts'),
}),
Components({
  dirs: [resolve(process.cwd(), './src/components/base')],
  resolvers: [VantResolver()],
  dts: resolve(process.cwd(), './components.d.ts'),
}),
```

解析器会加载模板中用到的组件和 `showToast` 等 API 对应的样式。全局主题可以按参考项目集中维护。让 `src/assets/styles/index.scss` 执行 `@use './modules/theme.scss';`，再在 `src/index.ts` 引入 `@/assets/styles/index.scss`。

```scss
// src/assets/styles/modules/theme.scss
html[data-theme] {
  --color-primary: #2f6feb;
  --van-primary-color: var(--color-primary);
}
```

当前 HTML 已有 `data-theme="light"`。需要暗色时可以在 `html[data-theme='dark']` 中覆盖语义变量。只需调整局部区域主题时，使用 `<van-config-provider :theme-vars="themeVars">`。参考项目的 rem/PostCSS 配置属于移动项目的布局方案，可按实际设计稿另行接入。

### 最小验证页

把下面内容放进 `src/views/components/index.vue` 或其他临时路由页，在移动端视口下检查按钮和 Toast。

```vue
<script setup lang="ts">
const showReady = (): void => {
  showToast('Vant 已接入');
};
</script>

<template>
  <van-button type="primary" @click="showReady">显示提示</van-button>
</template>
```

[Vant 快速上手](https://vant-ui.github.io/vant/#/zh-CN/quickstart) · [Vant ConfigProvider](https://vant-ui.github.io/vant/#/zh-CN/config-provider)

## 验证接入结果

1. 用 `pnpm dev` 打开示例页，检查组件、消息样式和主题。开发编译会生成自动导入声明。
2. 运行 `pnpm build` 和 `pnpm typecheck`。确认组件和命令式 API 已写入对应的自动导入声明，声明文件由插件生成。
3. 用 `pnpm preview` 再验证一次生产产物。项目接入暗色、弹层或移动端布局后，在相应的主题和视口下验证。
