# Axios 服务层规范

## 架构

接入 Axios 服务层后，可使用 `AxiosFactory` 封装。每个后端服务使用独立实例和拦截器。

```
services/
├── core/index.ts                    # AxiosFactory 基类
├── service-api.ts                   # 主 API 服务实例
├── service-api-interceptor.ts       # 主 API 服务拦截器
├── service-market.ts                # 按实际需要增加其他服务
└── service-market-interceptor.ts    # 对应拦截器
```

---

## 为什么每个服务独立

- 不同服务的 base URL 不同
- 不同服务的 token 策略可能不同
- 不同服务的错误处理逻辑不同
- 互不干扰，便于独立调试

---

## 拦截器职责

拦截器文件 (`service-xxx-interceptor.ts`) 负责：

- **请求拦截**：token 注入、请求头设置
- **响应拦截**：统一错误提示、401 跳转登录
- **网络错误**：中文错误提示

拦截器负责判断何时需要认证或反馈。UI、路由和状态管理通过
`configureServiceAuth`、`configureServiceFeedback` 等注入协议，由应用 `bootstrap/` 在启动阶段装配：

```ts
configureServiceAuth({
  getToken: () => authStore.token,
  onAuthenticationFailure: () => {
    authStore.logout();
    void router.replace({ name: 'Login' });
  },
});
```

依赖方向必须保持为 `bootstrap/store -> services`。禁止出现 `services -> store/router/feedback` 的反向依赖。

---

## 请求函数规范

请求函数放在 `api/modules/<模块>/interface.ts`，不在组件中直接调用 Axios：

```ts
/**
 * 用户登录
 */
export const login = (params: LoginParams) => {
  return serviceApi.post<LoginResponse, LoginParams>(API.API_LOGIN, params);
};
```

组件/hook 中只调用 API 层函数，不直接使用 service 实例。

API 模块必须显式导入具体服务实例，例如 `@/services/service-api`；`services/` 不建立聚合
`index.ts`，避免隐藏当前接口实际请求哪个后端服务。

---

## 类型扩展规范

Axios 源码大量使用 `any`，扩展类型时遵循以下原则：

### 框架继承的 `any`

接入 ESLint 且需要保留上游签名时，在对应类型适配文件的顶部声明例外。

```ts
/* eslint-disable @typescript-eslint/no-explicit-any */
```

不要在每个 interface 前后各写一对 enable/disable，噪音太大。

### 自己定义的参数：用 `unknown` 替代 `any`

拦截器 error 参数用 `unknown`，配合 `isAxiosError` 类型守卫：

```ts
export interface InterceptorHooks {
  requestInterceptor?: (config: InternalAxiosRequestConfig) => InternalAxiosRequestConfig;
  requestInterceptorCatch?: (error: unknown) => unknown;
  responseInterceptor?: (response: AxiosResponse) => AxiosResponse | Promise<AxiosResponse>;
  responseInterceptorCatch?: (error: unknown) => unknown;
}
```

```ts
import { isAxiosError } from 'axios';

responseInterceptorCatch: (error: unknown) => {
  if (isAxiosError(error)) {
    // error.response、error.status 等全部有类型
  }
  return Promise.reject(error);
};
```

### 重复字段抽 interface

多个扩展接口共享的字段抽成独立 interface，避免重复定义：

```ts
/* eslint-disable @typescript-eslint/no-explicit-any */

/**
 * 导入类型声明
 */
import type { AxiosRequestConfig, AxiosResponse, InternalAxiosRequestConfig } from 'axios';

/**
 * 请求扩展选项
 */
export interface RequestOptions {
  /**
   * 是否全局展示错误信息
   */
  globalErrorMessage?: boolean;
  /**
   * 是否全局展示成功信息
   */
  globalSuccessMessage?: boolean;
}

/**
 * 扩展请求配置
 */
export interface ExpandAxiosRequestConfig<D = any> extends AxiosRequestConfig<D> {
  interceptorHooks?: InterceptorHooks;
  requestOptions?: RequestOptions;
}

/**
 * 扩展内部请求配置
 */
export interface ExpandInternalAxiosRequestConfig<D = any> extends InternalAxiosRequestConfig<D> {
  interceptorHooks?: InterceptorHooks;
  requestOptions?: RequestOptions;
}

/**
 * 扩展响应配置
 */
export interface ExpandAxiosResponse<T = any, D = any> extends AxiosResponse<T, D> {
  config: ExpandInternalAxiosRequestConfig<D>;
}
```

## 接口契约

请求头、响应字段、成功码和登录失效码以对应后端契约为准。各接口保留实际的请求及响应字段，分页模型由对应 API 显式适配。业务失败转换为可识别的错误，认证和反馈通过启动装配注入，避免重复提示。
