# Element Plus 多主题实施指南

本方案将 `admin-ddm-v2` 的浅色、暗色方案整理为可在独立 SPA 中接入的步骤。先完成 [Element Plus 按需导入与 Sass 接入](ui-integration.md#element-plus)。本指南中的文件在项目选择接入主题时创建。示例配色可换成项目自己的色板。

## 参考实现与分工

参考工程为 `D:/hrhg-project/group-admin/admin-ddm-v2`。其中 `index.html` 在样式加载前恢复主题，`src/composables/use-theme.ts` 负责共享状态和切换动画，`src/assets/styles/modules/theme.scss` 维护应用语义变量，`src/assets/element-plus/modules/dark.scss` 把这些变量映射到 Element Plus。

| 文件 | 职责 |
| --- | --- |
| `build/rsbuild.base.config.ts` | 配置 Sass 按需加载并向 HTML 传入存储键 |
| `index.html` | 首次绘制前恢复 `html[data-theme]` |
| `src/constants/modules/storage.ts` | 定义当前应用的主题偏好存储键 |
| `src/composables/use-theme.ts` | 维护主题状态、保存偏好、运行切换动画 |
| `src/assets/styles/modules/tokens.scss` | 声明稳定的色板、间距和字体等基础值 |
| `src/assets/styles/modules/theme.scss` | 为浅色和暗色提供页面、文字、边框、交互等语义变量 |
| `src/assets/element-plus/modules/var.scss` | 编译期品牌色、字体等 Element Plus Sass 变量 |
| `src/assets/element-plus/modules/dark.scss` | 暗色下将 `--el-*` 桥接到应用变量 |
| `src/assets/styles/modules/component-overrides.scss` | 按组件公开类名组织全局覆盖，按需创建 |
| `src/assets/styles/modules/theme-transition.scss` | 仅在手动切换期间管理新旧快照层 |
| `src/assets/styles/index.scss` | 由应用入口一次性引入的全局样式 |

## 1. 在首屏绘制前恢复主题

定义一个带应用命名空间的键，并通过 Rsbuild 传给 HTML。复制脚手架搭建新项目时，为新项目选用自己的命名空间。

```ts
// src/constants/modules/storage.ts
export const STORAGE_LY_RSBUILD_SPA_UI_THEME = 'STORAGE_LY_RSBUILD_SPA_UI_THEME';
```

在 `build/rsbuild.base.config.ts` 导入这个常量，并在已有的 `html` 配置中增加下面的属性。

```ts
templateParameters: {
  themeStorageKey: JSON.stringify(STORAGE_LY_RSBUILD_SPA_UI_THEME),
},
```

保留模板根节点的 `data-theme="light"`。把恢复脚本放在 `index.html` 的 `<head>` 内，位于样式和入口脚本之前。这里读取的是 JSON 字符串，与项目的 `setLocalStorage` 使用同一种格式。

```html
<script>
  try {
    const preference = JSON.parse(localStorage.getItem(<%= themeStorageKey %>) ?? 'null');
    if (preference === 'dark') document.documentElement.dataset.theme = 'dark';
  } catch {
    // 存储不可用或数据损坏时使用根节点的默认浅色。
  }
</script>
```

刷新恢复主题只设置属性。部署使用 CSP 时，把这段首屏脚本纳入现有的 nonce 或 hash 策略。

## 2. 定义色板与语义变量

固定值集中在色板。页面使用 `--color-bg-page`、`--color-text-primary` 等变量，换主题时只在根节点重新定义差异。参考项目同时定义卡片、表格、控件和反馈的语义变量；示例先覆盖基础页面和常用 Element Plus 控件。

```scss
// src/assets/styles/modules/tokens.scss
:root {
  --palette-white: #fff;
  --palette-slate-50: #f5f7fa;
  --palette-slate-200: #dce1e8;
  --palette-slate-500: #667085;
  --palette-slate-700: #414b5b;
  --palette-slate-850: #242a35;
  --palette-slate-900: #1c212a;
  --palette-slate-950: #141820;
  --palette-blue-300: #89b4ff;
  --palette-blue-600: #2f6feb;
  --font-family-base: system-ui, sans-serif;
}
```

```scss
// src/assets/styles/modules/theme.scss
:root {
  color-scheme: light;
  --color-primary: var(--palette-blue-600);
  --color-bg-page: var(--palette-slate-50);
  --fill-color: var(--palette-white);
  --overlay-fill-color: var(--palette-white);
  --color-text-primary: var(--palette-slate-950);
  --color-text-secondary: var(--palette-slate-500);
  --border-color: var(--palette-slate-200);
  --border-color-light: var(--palette-slate-200);
  --control-hover-fill-color: var(--palette-slate-50);
  --control-disabled-fill-color: var(--palette-slate-50);
  --control-disabled-text-color: var(--palette-slate-500);
  --control-primary-fill-color: var(--palette-blue-600);
  --modal-backdrop-color: rgb(20 24 32 / 45%);
}

html[data-theme='dark'] {
  color-scheme: dark;
  --color-primary: var(--palette-blue-300);
  --color-bg-page: var(--palette-slate-950);
  --fill-color: var(--palette-slate-900);
  --overlay-fill-color: var(--palette-slate-850);
  --color-text-primary: var(--palette-slate-50);
  --color-text-secondary: var(--palette-slate-200);
  --border-color: var(--palette-slate-700);
  --border-color-light: var(--palette-slate-850);
  --control-hover-fill-color: var(--palette-slate-850);
  --control-disabled-fill-color: var(--palette-slate-850);
  --control-disabled-text-color: #a0a9b7;
  --modal-backdrop-color: rgb(0 0 0 / 65%);
}
```

在全局入口一次加载公共样式。项目接入表格密度、状态提示等覆盖时，可以在最后追加对应模块。

```scss
// src/assets/styles/index.scss
@use './modules/tokens.scss';
@use './modules/theme.scss';
@use './modules/theme-transition.scss';
@use '../element-plus/modules/dark.scss';

body {
  margin: 0;
  font-family: var(--font-family-base);
  background: var(--color-bg-page);
  color: var(--color-text-primary);
}
```

## 3. 将主题映射到 Element Plus

`ElementPlusResolver({ importStyle: 'sass' })` 配合 Sass 的 `additionalData` 加载按需样式。把品牌色写在现有的 `var.scss` 的 `@forward … with` 中，示例与色板都采用 `#2f6feb`。如果需要统一字体，在相同的 `with` 中加入 `$font-family: ('': var(--font-family-base))`。Sass 变量在构建时生效，运行时切换由 CSS 变量完成。

参考后台还通过注入入口加载了消息、抽屉、弹窗和导航的公共覆盖。独立脚手架可以让注入入口只加载 Sass 变量，将会输出 CSS 的覆盖从全局入口加载一次。主题映射挂在 `html[data-theme='dark']` 上，同样覆盖挂载到 `body` 的 Dialog、Select 下拉层和消息提示。

```scss
// src/assets/element-plus/modules/dark.scss
html[data-theme='dark'] {
  --el-bg-color: var(--fill-color);
  --el-bg-color-page: var(--color-bg-page);
  --el-bg-color-overlay: var(--overlay-fill-color);
  --el-text-color-primary: var(--color-text-primary);
  --el-text-color-regular: var(--color-text-secondary);
  --el-text-color-placeholder: var(--control-disabled-text-color);
  --el-text-color-disabled: var(--control-disabled-text-color);
  --el-border-color: var(--border-color);
  --el-border-color-light: var(--border-color-light);
  --el-fill-color: var(--control-hover-fill-color);
  --el-fill-color-light: var(--control-hover-fill-color);
  --el-fill-color-blank: var(--fill-color);
  --el-disabled-bg-color: var(--control-disabled-fill-color);
  --el-disabled-border-color: var(--border-color);
  --el-disabled-text-color: var(--control-disabled-text-color);
  --el-overlay-color-lighter: var(--modal-backdrop-color);
  --el-color-primary: var(--color-primary);

  @each $level in (3, 5, 7, 8, 9) {
    --el-color-primary-light-#{$level}: color-mix(in srgb, var(--fill-color) #{$level * 10%}, var(--color-primary));
  }

  --el-color-primary-dark-2: color-mix(in srgb, var(--color-primary) 80%, var(--palette-white));

  .el-button--primary:not(.is-plain, .is-link, .is-text, .is-dashed) {
    --el-button-bg-color: var(--control-primary-fill-color);
    --el-button-border-color: var(--control-primary-fill-color);
    --el-button-hover-bg-color: color-mix(in srgb, var(--control-primary-fill-color) 86%, var(--palette-white));
    --el-button-hover-border-color: var(--el-button-hover-bg-color);
    --el-button-active-bg-color: color-mix(in srgb, var(--control-primary-fill-color) 82%, var(--color-bg-page));
    --el-button-active-border-color: var(--el-button-active-bg-color);
  }
}
```

实心按钮使用较深的背景承载白字，普通链接继续使用暗色下较浅的强调色。扩展时按实际组件补齐 success、warning、danger 等状态色阶、蒙层和阴影。参考项目还通过暗色 Sass 变量适配 `el-empty`，并针对 Switch、Checkbox、选中日期及分页补充对比度。

业务页面只需要调整某个浮层时，给它设置 `popper-class`、`modal-class` 或对应组件提供的公开属性，再在全局覆盖文件按业务类名写规则。根节点使用 `data-theme` 即可运行本方案；Element Plus 官方暗色 CSS 使用 `html.dark`，采用官方方案时需要同步选择器和样式加载方式。[Element Plus 暗色主题](https://element-plus.org/zh-CN/guide/dark-mode.html)

## 4. 共享主题状态并绑定切换入口

把下面的实现放进 `src/composables/use-theme.ts`。多个入口读取同一份只读状态和切换锁。点击用指针坐标，键盘触发用按钮中心，无触发控件时从视口中心展开。浏览器不支持 View Transition、用户减少动效或动画失败时，主题仍会更新并保存。

```ts
// src/composables/use-theme.ts
import { nextTick, readonly, ref } from 'vue';
import { STORAGE_LY_RSBUILD_SPA_UI_THEME } from '@/constants/modules/storage';
import { getLocalStorage, setLocalStorage } from '@/utils/modules/storage';

type ThemeName = 'light' | 'dark';
const themeName = ref<ThemeName>('light');
const isThemeTransitioning = ref(false);

const applyTheme = (theme: ThemeName): void => {
  document.documentElement.dataset.theme = theme;
  themeName.value = theme;
};

/** 挂载前同步 Vue 状态和首屏主题。 */
export const initializeTheme = (): void => {
  const saved = getLocalStorage<unknown>(STORAGE_LY_RSBUILD_SPA_UI_THEME);
  applyTheme(saved === 'dark' ? 'dark' : 'light');
};

/** 为多个入口提供共享主题状态和切换方法。 */
export const useTheme = () => {
  const toggleTheme = async (event?: Event): Promise<void> => {
    if (isThemeTransitioning.value) return;
    const next = themeName.value === 'dark' ? 'light' : 'dark';
    const changeTheme = (): void => {
      applyTheme(next);
      try {
        setLocalStorage(STORAGE_LY_RSBUILD_SPA_UI_THEME, next);
      } catch {
        // 存储受限时仍能切换当前页面。
      }
    };

    if (!document.startViewTransition || window.matchMedia('(prefers-reduced-motion: reduce)').matches) {
      changeTheme();
      return;
    }

    const bounds =
      event?.currentTarget instanceof HTMLElement ? event.currentTarget.getBoundingClientRect() : undefined;
    const pointer = event instanceof MouseEvent && event.detail > 0;
    const x = pointer ? event.clientX : bounds ? bounds.left + bounds.width / 2 : window.innerWidth / 2;
    const y = pointer ? event.clientY : bounds ? bounds.top + bounds.height / 2 : window.innerHeight / 2;
    const radius = Math.hypot(Math.max(x, window.innerWidth - x), Math.max(y, window.innerHeight - y));

    const root = document.documentElement;
    let transition: ViewTransition | undefined;
    isThemeTransitioning.value = true;
    root.dataset.themeTransition = 'active';

    try {
      transition = document.startViewTransition(async () => {
        changeTheme();
        await nextTick();
      });
      await transition.ready;
      await root.animate(
        { clipPath: [`circle(0px at ${x}px ${y}px)`, `circle(${radius}px at ${x}px ${y}px)`] },
        { duration: 500, easing: 'ease-in-out', pseudoElement: '::view-transition-new(root)' },
      ).finished;
      await transition.finished;
    } catch {
      transition?.skipTransition();
      await transition?.updateCallbackDone.catch(() => undefined);
      if (themeName.value !== next) changeTheme();
    } finally {
      delete root.dataset.themeTransition;
      isThemeTransitioning.value = false;
    }
  };

  return { themeName: readonly(themeName), isThemeTransitioning: readonly(isThemeTransitioning), toggleTheme };
};
```

`src/index.ts` 引入 `@/assets/styles/index.scss` 和 `initializeTheme`，在挂载应用前调用 `initializeTheme()`。浅色、暗色切换共用同一入口。比如在需要切换的组件中调用 `const { themeName, isThemeTransitioning, toggleTheme } = useTheme();`，组件可以使用下面的按钮。

```vue
<el-button :disabled="isThemeTransitioning" @click="toggleTheme">
  {{ themeName === 'dark' ? '切换到浅色' : '切换到暗色' }}
</el-button>
```

CSS 只在手动主题切换期间覆盖原生快照动画，正常页面切换仍使用各自的动画配置。

```scss
// src/assets/styles/modules/theme-transition.scss
html[data-theme-transition='active'] {
  &::view-transition-old(root),
  &::view-transition-new(root) {
    animation: none;
    mix-blend-mode: normal;
  }

  &::view-transition-old(root) {
    z-index: 1;
  }

  &::view-transition-new(root) {
    z-index: 2;
  }
}
```

图表、Canvas 等由脚本绘制的内容可以在主题变化后读取 `getComputedStyle(document.documentElement)` 中的语义色，再更新图表选项。请求结果和图表数据可以继续复用；跨站 iframe 的内部主题由其服务控制。[View Transition 实现指南](https://developer.chrome.com/docs/web-platform/view-transitions/same-document)

## 5. 验收清单

| 场景 | 预期 |
| --- | --- |
| 清空主题键后打开网站 | 首屏使用浅色 |
| 已保存暗色后直接打开深层路由或刷新 | HTML 在样式渲染前恢复暗色，Vue 挂载后状态一致 |
| 点击切换、按 Enter 切换 | 分别从鼠标位置、按钮中心展开；切换完成后按钮可再次操作 |
| 减少动效、浏览器不支持 View Transition | 主题直接切换，偏好仍可恢复 |
| 存储不可用或键值损坏 | 页面正常显示，无法读取偏好时使用浅色 |
| 打开 Dialog、Select 下拉层、消息提示 | Teleport 到 `body` 的内容也使用暗色，文字、边框和背景匹配 |
| 检查禁用、激活、悬浮等状态 | 可读且可区分，重点核对实心按钮、表单和下拉列表 |
| 在两个路由间跳转或确认版本更新 | 主题偏好继续保留；刷新恢复时无切换动画 |

用 `pnpm build` 和 `pnpm preview` 复验生产样式、HTML 模板和主题恢复。
