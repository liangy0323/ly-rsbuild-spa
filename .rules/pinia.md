# Pinia 状态管理规范

## Store 写法

统一使用 Composition 风格：

```ts
/**
 * 布局偏好的持久化键与其他应用缓存隔离
 */
import { PINIA_DEMO_LAYOUT } from '@/constants';

export const useLayoutStore = defineStore(
  'layout',
  () => {
    const isSidebarCollapsed = ref(false);

    const setSidebarCollapsed = (value: boolean): void => {
      isSidebarCollapsed.value = value;
    };

    return { isSidebarCollapsed, setSidebarCollapsed };
  },
  {
    persist: {
      key: PINIA_DEMO_LAYOUT,
      pick: ['isSidebarCollapsed'],
    },
  },
);
```

---

## 持久化策略

- 需要持久化时使用 `pinia-plugin-persistedstate`，仅保存必要状态
- 持久化 key 统一使用 `PINIA_<APP>_<STORE>` 格式的常量。Store id 与存储 key 分开管理，详见 [storage.md](storage.md)。
- 使用 `pick` 选择性保存，不持久化全部状态
- 大量数据不放 Pinia 持久化；确有离线存储需求时再选择合适的存储方案

---

## 组织方式

```
stores/
├── index.ts              # Pinia 实例 + 持久化插件注册
└── modules/
    ├── chat.ts           # 聊天 Store
    ├── conversation.ts   # 会话 Store
    ├── user.ts           # 用户 Store
    └── layout.ts         # 布局 Store
```

---

## 使用约定

- 组件顶层调用 `useXxxStore()`，不在函数内部创建
- Store 内部的 `ref`/`computed` 不需要额外类型标注（能推断的不标注）
- Store 之间可以互相引用，但避免循环依赖

## 组件外使用与权限就绪

- 在 bootstrap 中先创建 Pinia 并 `app.use(pinia)`，组件外需要 Store 时显式调用 `useXxxStore(pinia)`，不要在模块导入顶层依赖隐式活动实例。
- 指令注册时注入读取权限状态的函数，函数内部读取 Store 的响应式字段；指令本身不调用 Store，也不访问 localStorage。
- 权限加载异步性独立于 Pinia 初始化。用非持久化的 `ready` 表示当前会话权限已被后端确认；权限缓存恢复不能使其自动变为 true。
- 更新权限、重新登录或切换账号时先重置 ready，再请求当前会话权限；过期会话的响应不得写入新会话。
