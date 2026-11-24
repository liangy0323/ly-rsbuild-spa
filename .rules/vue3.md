# Vue 3 编码规范

本文档用于约束 Vue 3 单文件组件的写法。目录结构和命名分别见 [目录组织规范](directory.md) 和 [命名规范](naming.md)。

---

## 一、组件结构

使用 `<script setup lang="ts">`，按导入、Props / Emits / Model、状态、计算、方法、副作用组织；不存在的分组不写。注释遵循 `comment-style.md`，不强制无信息量的导入和结构分组注释。

```vue
<template>
  <button type="button" :disabled="props.disabled" @click="handleSubmit">提交</button>
</template>

<script setup lang="ts">
interface Props {
  /**
   * 是否禁用提交
   */
  disabled?: boolean;
}

interface Emits {
  submit: [];
}

const props = withDefaults(defineProps<Props>(), { disabled: false });
const emits = defineEmits<Emits>();
const handleSubmit = (): void => emits('submit');
</script>
```

Vue、Vue Router 和 Pinia 的 API 按 [公共构建配置](../build/rsbuild.base.config.ts) 自动导入，类型和业务模块显式导入。第三方 UI 库按各自的接入方式使用。

---

## 二、文件与组件引用

### 组件文件策略

- 多文件组件：使用 `kebab-case` 目录 + `index.vue` 入口。

```text
base-selector/
├── index.vue
├── base-selector.scss
└── components/
    └── option-panel/
        └── index.vue
```

- 单文件组件：简单组件直接使用 `kebab-case.vue`，无需建目录。
- 页面内私有子组件按 `directory.md` 的 SPA 规则组织，避免目录嵌套过深。
- `<script setup>` 中导入组件使用 PascalCase。
- `<template>` 中引用组件统一使用 kebab-case。

```vue
<!-- 正确 -->
<base-tab v-model="activeValue" />
<el-skeleton v-for="item in list" :key="item.id" />
<infinite-list :loading="loading" @load="handleLoad" />

<!-- 错误 -->
<BaseTab v-model="activeValue" />
<ElSkeleton v-for="item in list" :key="item.id" />
```

---

## 三、Props / Emits / Model

### v-model

项目基于 Vue 3.5+。标准 `v-model` 契约优先使用 `defineModel`，避免手写 `modelValue` prop 和 `update:modelValue` emit。

```ts
/**
 * 定义 v-model
 */
const modelValue = defineModel<string>({ default: '' });

/**
 * 定义 visible v-model
 */
const visible = defineModel<boolean>('visible', { default: false });
```

使用边界：

1. 真实双向绑定契约使用 `defineModel`。
2. 普通输入属性仍放在 `Props` 中，不为了少写代码滥用 `defineModel`。
3. `defineModel` 只负责 `v-model`，额外事件仍使用 `Emits interface + defineEmits<Emits>()`。
4. 多个 `v-model` 可以使用多个 `defineModel`，命名必须体现语义。

### Props

组件 props 默认使用 `Props interface + withDefaults + props`。这种写法默认值集中、字段注释清晰、模板和逻辑中来源明确，适合团队协作。

```ts
/**
 * 定义 props 的类型声明
 */
interface Props {
  /**
   * 标题
   */
  title: string;
  /**
   * 是否禁用
   */
  disabled?: boolean;
  /**
   * 尺寸
   */
  size?: 'small' | 'medium' | 'large';
}

/**
 * 定义 props
 */
const props = withDefaults(defineProps<Props>(), {
  disabled: false,
  size: 'medium',
});
```

规则：

- Props interface 固定命名为 `Props`。
- 每个 props 字段必须写 JSDoc 注释，说明语义。
- props 实例统一命名为 `props`。
- 模板和脚本中统一使用 `props.xxx`，让数据来源明确。
- 默认值统一写在 `withDefaults` 中。
- 标准 `v-model` 字段不放在 Props 中，优先使用 `defineModel`。
- Vue 3.5 的 Reactive Props Destructure 允许使用，但不作为团队默认风格；只有在极简单组件中，经评审确认不会降低统一性时才可使用。

### Emits

Emits 类型统一使用 `interface Emits`，内部使用 Vue 3.3+ 元组语法，实例统一命名为 `emits`。

```ts
/**
 * 定义 emits 的类型声明
 */
interface Emits {
  submit: [];
  change: [value: string];
  select: [id: string, index: number];
}

/**
 * 定义 emits
 */
const emits = defineEmits<Emits>();
```

规则：

- key 是事件名。
- value 是参数元组，`[]` 表示无参数。
- 元组中的参数名用于表达语义，不影响调用方式。
- 禁止使用 call signature 写法。
- 禁止把复杂 emits 类型内联到 `defineEmits` 中。

---

## 四、组件边界与数据流

### 组件边界

非简单功能在编码前先确定组件边界：

1. 每个组件用一句话说明单一职责。
2. 明确子组件的 props / emits 契约。
3. 页面级组件默认只做布局、数据装配和功能组合。
4. 独立 UI 区块下沉到子组件。
5. 状态、请求、副作用和复杂交互优先抽到 hook/helper。

出现以下情况时，应检查职责边界并按复杂度决定是否拆分：

- 同一个组件同时承担数据编排和多个大块展示。
- 模板中出现 3 个以上独立 UI 区块。
- 列表项、卡片、筛选区、操作栏等结构可独立命名。
- `<script setup>` 中业务逻辑超过 20 行（仅作为检查提醒）。

### 数据流

组件通信默认遵循 Props Down / Events Up：

1. props 是只读输入，子组件不得直接修改 props。
2. 子组件需要修改父级状态时，通过 `emit` 暴露事件。
3. `v-model` 只用于真实双向绑定契约，不用于普通状态同步。
4. `provide/inject` 只用于跨多层级共享上下文，不替代清晰的 props / emits。
5. 父组件不要依赖子组件 ref 读取内部状态；确需命令式调用时，统一按本文「Template Ref」章节执行。

---

## 五、状态、计算与副作用

### 响应式数据

```ts
/**
 * 列表数据
 */
const itemList = shallowRef<ListItem[]>([]);

/**
 * 当前选中值
 */
const activeValue = ref('');
```

判断标准：

- 大列表数据、表格行、消息流等使用 `shallowRef`。
- primitive、简单对象、小型状态使用 `ref`。
- 更新 `shallowRef` 列表时整体替换引用。

### 计算属性

派生值优先使用 `computed`，不要用 `watch` / `watchEffect` 维护另一个派生 `ref`。

```ts
/**
 * 计算属性
 * 作用：启用状态的列表
 */
const activeItemList = computed(() => itemList.value.filter((item) => item.active));
```

`computed` getter 必须保持纯净，只做计算，不做请求、存储、emit、日志、埋点或修改其他响应式状态。

### 方法

组件内函数统一使用箭头函数，便于保持写法一致。

```ts
/**
 * 处理提交
 */
const handleSubmit = (): void => {
  emits('submit');
};
```

规则：

- 事件处理函数统一使用 `handle` 前缀。
- 工具函数优先抽到 helper/utils。
- 请求、状态编排和复杂交互优先抽到 hook。

### Watch

`watch` / `watchEffect` 只用于副作用，例如请求、订阅、存储同步、埋点、DOM 相关操作。

```ts
watch(
  keyword,
  async (value, _oldValue, onCleanup) => {
    const controller = new AbortController();

    onCleanup(() => {
      controller.abort();
    });

    await search(value, { signal: controller.signal });
  },
  { immediate: true },
);
```

规则：

1. 首次加载和后续变化逻辑一致时，使用 `{ immediate: true }`。
2. `watch` 内发起异步请求时，必须处理竞态。
3. 监听 reactive 对象字段时，使用 getter：`watch(() => state.id, callback)`。
4. 不要用 deep watch 兜底业务设计；确需使用时必须说明原因。

### Store

状态用 `storeToRefs`，方法直接解构。

```ts
const moduleStore = useModuleStore();
const { currentValue, itemList } = storeToRefs(moduleStore);
const { setCurrentValue, clearItemList } = moduleStore;
```

### Template Ref

Vue 3.5+ 使用 `useTemplateRef`，替代旧的 `ref<Type | null>(null)` 模式。

#### DOM ref

```vue
<template>
  <div ref="containerRef">
    <input ref="inputRef" />
  </div>
</template>

<script setup lang="ts">
const containerRef = useTemplateRef<HTMLDivElement>('containerRef');
const inputRef = useTemplateRef<HTMLInputElement>('inputRef');
</script>
```

#### 项目内部组件 ref

项目内部 Vue 组件的模板引用，统一使用 `InstanceType<typeof Xxx> | null` 推导组件实例类型。组件通过 `defineExpose` 暴露的方法，可以直接从 `ref.value` 上获得类型提示。

```vue
<template>
  <detail-modal ref="detailModalRef" />
</template>

<script setup lang="ts">
/**
 * 导入组件
 */
import DetailModal from './components/detail-modal/index.vue';

/**
 * 定义响应式数据
 */
const detailModalRef = useTemplateRef<InstanceType<typeof DetailModal> | null>('detailModalRef');

/**
 * 打开详情弹窗
 */
const handleDetailOpen = (): void => {
  detailModalRef.value?.open();
};
</script>
```

#### 第三方组件 ref

Element Plus、Vant 等第三方组件通常会导出组件实例类型。此类模板引用优先使用官方导出的类型声明，不使用 `InstanceType<typeof Xxx>` 自行推导。

```vue
<template>
  <el-scrollbar ref="scrollbarRef" />
</template>

<script setup lang="ts">
/**
 * 导入类型声明
 */
import type { ScrollbarInstance } from 'element-plus';

/**
 * 定义响应式数据
 */
const scrollbarRef = useTemplateRef<ScrollbarInstance | null>('scrollbarRef');
</script>
```

规则：

- DOM 节点 ref 使用对应 DOM 类型，如 `HTMLDivElement`、`HTMLInputElement`。
- 项目内部组件 ref 使用 `InstanceType<typeof ComponentName> | null`。
- 第三方组件 ref 优先使用官方导出的实例类型，如 Element Plus 的 `ScrollbarInstance`。
- 组件需要暴露给父组件调用的方法，必须在子组件中用 `defineExpose` 明确暴露最小 API。
- 模板中的 `ref` 名称与 `useTemplateRef` 字符串必须完全一致。

---

## 六、Composable

编写 hook/composable 时，参数接收推荐使用 `MaybeRefOrGetter<T>` + `toValue`。

```ts
const useSearch = (keyword: MaybeRefOrGetter<string>) => {
  /**
   * 处理搜索
   */
  const handleSearch = (): void => {
    const value = toValue(keyword);
    // ...
  };

  return { handleSearch };
};
```

规则：

1. Composable 只承载可复用的有状态逻辑；纯格式化、计算、转换函数放到 `utils`。
2. 多个可选参数使用 options object，避免位置参数难以阅读。
3. 返回 plain object，便于调用方解构。
4. 内部状态不允许外部随意修改时，返回 `readonly(state)` + 显式 action。
5. 事件监听、定时器、订阅、第三方实例必须在 composable 内完成清理。

### 命名与目录约定

按复用范围分层放置，文件名一律 `use-<职责>.ts`，导出 `use<职责>`：

| 复用范围       | 存放位置                                   | barrel                                    |
| -------------- | ------------------------------------------ | ----------------------------------------- |
| 项目内多页复用 | `src/composables/`              | 不 barrel，`@/composables/use-*` 显式导入 |
| 单页逻辑分块   | `src/views/<page>/composables/` | 不 barrel，相对路径显式导入               |
| 单组件私有     | 组件目录内                                 | 不 barrel                                 |

命名按「职责」而非「合集」拆分，禁止三类反模式：

1. **合集式命名**：`use-<page>.ts`（如 `use-watchlist-detail` 早期把列表分页 + 详情构建 + 码表联动全塞一个 composable）。按职责拆成 `use-watchlist-list`（分页）/ `use-watchlist-detail`（详情）。
2. **职责模糊命名**：`use-data` / `use-logic` / `use-helper` 这类无信息量的名字。
3. **编排式 composable**：不要为「把多个 composable 串起来」再造一个无明确职责的 `use-<page>`，页面级编排（初始化顺序、跨 composable 联动）留在对应页面入口。

单页 `composables/` 目录因此**不设 `index.ts`**：页面私有 composable 无对外复用诉求，barrel 只会诱导“合集式”聚合。应用内 `composables/` 一律不走 barrel；页面私有的 `utils`、`types`、`constants` 仍按 [directory.md](directory.md) 走 barrel。

---

## 七、模板规则

模板表达式保持声明式，复杂派生逻辑放到 `<script setup>` 中。

```vue
<script setup lang="ts">
const activeUserList = computed(() => userList.value.filter((user) => user.active));
</script>

<template>
  <li v-for="user in activeUserList" :key="user.id">
    {{ user.name }}
  </li>
</template>
```

规则：

1. `v-for` 必须提供稳定的 primitive key，优先使用业务 id。
2. 不要在同一个元素上同时使用 `v-if` 和 `v-for`，列表过滤放到 `computed`。
3. 频繁切换显示状态用 `v-show`；低频且初始渲染成本高的内容用 `v-if`。
4. `v-html` 只能用于可信 HTML 或已净化 HTML。

### v-html 处理标准

凡需用 `v-html` 渲染后端返回的富文本，一律按以下标准处理：

1. 净化必须经过项目内统一的白名单净化工具。
2. 净化结果用 `computed` 缓存，不在模板里直接调用 `sanitizeHtml(...)`。
3. `v-html` 那一行加局部 disable 注释，不要全局关闭 `vue/no-v-html`。
4. 如果内容本身是纯文本，使用 `{{ }}`，不要用 `v-html`。

```vue
<script setup lang="ts">
/**
 * 导入工具类
 */
import { sanitizeHtml } from '@/utils';

/**
 * 净化后的富文本内容
 */
const safeContent = computed(() => sanitizeHtml(props.data.content));
</script>

<template>
  <!-- 内容已由 sanitizeHtml 净化，可以安全渲染富文本 -->
  <!-- eslint-disable-next-line vue/no-v-html -->
  <span v-html="safeContent"></span>
</template>
```

---

## 八、性能规则

### 渲染优化

```vue
<!-- 静态内容只渲染一次 -->
<div v-once>
  <h1>{{ appTitle }}</h1>
  <p>版本 {{ version }}</p>
</div>

<!-- 依赖不变时跳过更新 -->
<div v-for="item in list" :key="item.id" v-memo="[item.id, item.status]">
  <base-list-item :data="item" />
</div>
```

规则：

- 大列表优先考虑虚拟滚动。
- 长列表渲染前先确认是否需要分页、懒加载或虚拟列表。
- `v-once` 只用于纯静态内容。
- `v-memo` 只用于能明确判断依赖稳定的列表项。

---

## 九、样式策略

- 简单组件样式可以内联在 `<style lang="scss" scoped>` 中。
- 样式超过 50 行，或需要拆分模块时，移动到外部 SCSS。
- 应用全局样式在 `src/index.ts` 中引入，组件局部样式就近维护。
- 组件内不要硬编码可主题化色值，优先使用已有 CSS 变量。

---

## 十、关键约定

- 只用 Composition API，不使用 Options API。
- 使用 `<script setup lang="ts">`，不使用普通 `<script>`。
- 模板中组件引用统一使用 kebab-case。
- 组件内函数统一使用箭头函数。
- 事件处理函数统一使用 `handle` 前缀。
- 状态、请求、副作用和复杂交互优先抽 hook/helper。
- 组件逻辑超过 20 行时检查职责，按复杂度决定是否拆分。
- 完成组件后检查内存泄漏和容错处理。

---
