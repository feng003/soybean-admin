# 用户列表功能开发指南

## 概述

本文档详细描述了在 SoybeanAdmin 中实现用户列表功能的完整流程，包括权限控制、路由配置、API 处理等核心实现细节。

## 1. 系统架构概览

SoybeanAdmin 采用分层架构设计：
- **表现层**: Vue3 + TypeScript + NaiveUI 组件
- **状态管理层**: Pinia Store 管理
- **路由层**: Vue Router + elegant-router 自动路由生成
- **服务层**: 基于 Axios 的 HTTP 请求封装
- **权限层**: 基于角色的访问控制 (RBAC)

## 2. 权限系统实现

### 2.1 用户信息结构
位置：`src/typings/api.d.ts:62-67`

```typescript
interface UserInfo {
  userId: string;
  userName: string;
  roles: string[];      // 用户角色列表
  buttons: string[];    // 按钮权限列表
}
```

### 2.2 权限验证机制
位置：`src/store/modules/auth/index.ts`

#### 认证流程
1. **登录验证**: `login()` 方法处理用户登录
   - 调用 `fetchLogin()` 验证用户凭据
   - 存储 token 和 refreshToken 到 localStorage
   - 获取用户信息并更新 store

2. **用户信息获取**: `getUserInfo()` 方法
   - 调用 `fetchGetUserInfo()` API
   - 更新 userInfo 状态，包含角色和权限信息

3. **Token 管理**:
   - 自动添加 Authorization 头
   - Token 过期自动刷新机制
   - 登出时清理所有认证信息

### 2.3 角色权限控制
位置：`src/store/modules/route/shared.ts:12-43`

#### 路由权限过滤
```typescript
function filterAuthRouteByRoles(route: ElegantConstRoute, roles: string[]) {
  const routeRoles = (route.meta && route.meta.roles) || [];
  
  // 空角色配置表示允许所有用户访问
  const isEmptyRoles = !routeRoles.length;
  
  // 检查用户角色是否包含在路由要求的角色中
  const hasPermission = routeRoles.some(role => roles.includes(role));
  
  return hasPermission || isEmptyRoles ? [filterRoute] : [];
}
```

#### 超级管理员权限
位置：`src/store/modules/auth/index.ts:32-36`

```typescript
const isStaticSuper = computed(() => {
  const { VITE_AUTH_ROUTE_MODE, VITE_STATIC_SUPER_ROLE } = import.meta.env;
  
  return VITE_AUTH_ROUTE_MODE === 'static' && 
         userInfo.roles.includes(VITE_STATIC_SUPER_ROLE);
});
```

## 3. 路由系统实现

### 3.1 路由模式
SoybeanAdmin 支持两种路由模式：
- **静态模式 (static)**: 路由在构建时生成，适用于开发环境
- **动态模式 (dynamic)**: 路由从后端获取，适用于生产环境

### 3.2 路由初始化流程
位置：`src/store/modules/route/index.ts`

#### 常量路由初始化
```typescript
async function initConstantRoute() {
  if (isInitConstantRoute.value) return;
  
  const staticRoute = createStaticRoutes();
  
  if (authRouteMode.value === 'static') {
    addConstantRoutes(staticRoute.constantRoutes);
  } else {
    // 从后端获取常量路由
    const { data, error } = await fetchGetConstantRoutes();
    if (!error) {
      addConstantRoutes(data);
    }
  }
  
  handleConstantAndAuthRoutes();
  setIsInitConstantRoute(true);
}
```

#### 权限路由初始化
```typescript
async function initAuthRoute() {
  if (!authStore.userInfo.userId) {
    await authStore.initUserInfo();
  }
  
  if (authRouteMode.value === 'static') {
    initStaticAuthRoute();
  } else {
    await initDynamicAuthRoute();
  }
}
```

### 3.3 路由守卫机制
位置：`src/router/guard/route.ts`

#### 权限检查流程
```typescript
router.beforeEach(async (to, from, next) => {
  const authStore = useAuthStore();
  
  const isLogin = Boolean(localStg.get('token'));
  const needLogin = !to.meta.constant;
  const routeRoles = to.meta.roles || [];
  
  // 检查用户是否有访问权限
  const hasRole = authStore.userInfo.roles.some(role => 
    routeRoles.includes(role)
  );
  const hasAuth = authStore.isStaticSuper || !routeRoles.length || hasRole;
  
  // 根据权限状态进行路由跳转控制
  if (!isLogin && needLogin) {
    next({ name: 'login', query: { redirect: to.fullPath } });
  } else if (!hasAuth) {
    next({ name: '403' });
  } else {
    next();
  }
});
```

## 4. API 服务层实现

### 4.1 请求拦截器配置
位置：`src/service/request/index.ts`

#### 认证头添加
```typescript
async onRequest(config) {
  const Authorization = getAuthorization();
  Object.assign(config.headers, { Authorization });
  return config;
}
```

#### 响应拦截处理
```typescript
async onBackendFail(response, instance) {
  const authStore = useAuthStore();
  const responseCode = String(response.data.code);
  
  // 登出码处理
  const logoutCodes = import.meta.env.VITE_SERVICE_LOGOUT_CODES?.split(',') || [];
  if (logoutCodes.includes(responseCode)) {
    authStore.resetStore();
    return null;
  }
  
  // Token 过期处理
  const expiredTokenCodes = import.meta.env.VITE_SERVICE_EXPIRED_TOKEN_CODES?.split(',') || [];
  if (expiredTokenCodes.includes(responseCode)) {
    const success = await handleExpiredRequest(request.state);
    if (success) {
      return instance.request(response.config);
    }
  }
}
```

### 4.2 用户相关 API
位置：`src/service/api/auth.ts`

```typescript
// 登录接口
export function fetchLogin(userName: string, password: string) {
  return request<Api.Auth.LoginToken>({
    url: '/auth/login',
    method: 'post',
    data: { userName, password }
  });
}

// 获取用户信息
export function fetchGetUserInfo() {
  return request<Api.Auth.UserInfo>({ 
    url: '/auth/getUserInfo' 
  });
}

// 刷新令牌
export function fetchRefreshToken(refreshToken: string) {
  return request<Api.Auth.LoginToken>({
    url: '/auth/refreshToken',
    method: 'post',
    data: { refreshToken }
  });
}
```

### 4.3 路由相关 API
位置：`src/service/api/route.ts`

```typescript
// 获取用户路由
export function fetchGetUserRoutes() {
  return request<Api.Route.UserRoute>({ 
    url: '/route/getUserRoutes' 
  });
}

// 检查路由是否存在
export function fetchIsRouteExist(routeName: string) {
  return request<boolean>({ 
    url: '/route/isRouteExist', 
    params: { routeName } 
  });
}
```

## 5. 用户列表功能实现步骤

### 5.1 创建用户管理页面路由

#### Step 1: 创建页面文件
```bash
# 创建用户管理页面目录
mkdir -p src/views/manage/user

# 创建页面组件文件
touch src/views/manage/user/index.vue
```

#### Step 2: 定义路由元信息
在页面组件中添加路由元信息：

```typescript
defineOptions({
  name: 'manage_user',
});
```

#### Step 3: 配置路由权限
路由将自动生成，需要在路由 meta 中配置权限：

```typescript
// 在路由配置中设置角色权限
meta: {
  title: '用户管理',
  i18nKey: 'route.manage_user',
  roles: ['admin', 'user-manager'], // 只有管理员和用户管理员可以访问
  icon: 'ic:round-manage-accounts',
  order: 1
}
```

### 5.2 实现用户列表组件

#### Step 1: 定义用户数据类型
```typescript
// src/typings/api.d.ts
declare namespace Api {
  namespace User {
    interface UserItem extends Common.CommonRecord {
      userId: string;
      userName: string;
      email: string;
      phone: string;
      roles: string[];
      status: Common.EnableStatus;
    }

    interface UserSearchParams extends Common.CommonSearchParams {
      userName?: string;
      email?: string;
      status?: Common.EnableStatus;
    }

    type UserList = Common.PaginatingQueryRecord<UserItem>;
  }
}
```

#### Step 2: 创建用户 API 服务
```typescript
// src/service/api/user.ts
export function fetchGetUserList(params?: Api.User.UserSearchParams) {
  return request<Api.User.UserList>({
    url: '/user/list',
    method: 'get',
    params
  });
}

export function fetchAddUser(data: Omit<Api.User.UserItem, 'id'>) {
  return request({
    url: '/user/add',
    method: 'post',
    data
  });
}

export function fetchUpdateUser(data: Api.User.UserItem) {
  return request({
    url: '/user/update',
    method: 'put',
    data
  });
}

export function fetchDeleteUser(id: number) {
  return request({
    url: `/user/delete/${id}`,
    method: 'delete'
  });
}
```

#### Step 3: 实现页面组件
```vue
<!-- src/views/manage/user/index.vue -->
<script setup lang="ts">
import { ref, onMounted } from 'vue';
import type { Ref } from 'vue';
import { NButton, NPopconfirm, NSpace, NTag } from 'naive-ui';
import { fetchGetUserList, fetchDeleteUser } from '@/service/api';
import { useAppStore } from '@/store/modules/app';
import { useTable, useTableOperate } from '@/hooks/common/table';
import { $t } from '@/locales';

defineOptions({
  name: 'manage_user'
});

const appStore = useAppStore();

// 搜索参数
const searchParams = ref<Api.User.UserSearchParams>({
  current: 1,
  size: 10,
  userName: '',
  email: '',
  status: null
});

// 表格配置
const {
  columns,
  columnChecks,
  data,
  getData,
  getDataByPage,
  loading,
  mobilePagination,
  searchQuery,
  resetSearchQuery
} = useTable({
  apiFn: fetchGetUserList,
  searchParams,
  columns: () => [
    {
      type: 'selection',
      align: 'center',
      width: 48
    },
    {
      key: 'index',
      title: $t('common.index'),
      align: 'center',
      width: 64
    },
    {
      key: 'userName',
      title: '用户名',
      align: 'center',
      minWidth: 100
    },
    {
      key: 'email',
      title: '邮箱',
      align: 'center',
      minWidth: 200
    },
    {
      key: 'roles',
      title: '角色',
      align: 'center',
      width: 200,
      render: row => {
        if (row.roles?.length) {
          return h(
            NSpace,
            { size: 'small', wrap: false },
            {
              default: () =>
                row.roles.map(role =>
                  h(
                    NTag,
                    { type: 'primary', size: 'small' },
                    { default: () => role }
                  )
                )
            }
          );
        }
        return '';
      }
    },
    {
      key: 'status',
      title: $t('common.status'),
      align: 'center',
      width: 100,
      render: row => {
        if (row.status === null) {
          return '';
        }

        const status = row.status === '1' ? 'success' : 'error';
        const label = row.status === '1' ? $t('common.enable') : $t('common.disable');

        return h(NTag, { type: status, size: 'small' }, { default: () => label });
      }
    },
    {
      key: 'createTime',
      title: $t('common.createTime'),
      align: 'center',
      width: 180
    },
    {
      key: 'operate',
      title: $t('common.operate'),
      align: 'center',
      width: 130,
      render: row => (
        <div class="flex-center gap-8px">
          <NButton
            type="primary"
            ghost
            size="small"
            onClick={() => handleEdit(row.id)}
          >
            {$t('common.edit')}
          </NButton>
          <NPopconfirm onPositiveClick={() => handleDelete(row.id)}>
            {{
              default: () => $t('common.confirmDelete'),
              trigger: () => (
                <NButton type="error" ghost size="small">
                  {$t('common.delete')}
                </NButton>
              )
            }}
          </NPopconfirm>
        </div>
      )
    }
  ]
});

// 表格操作
const {
  drawerVisible,
  operateType,
  editingData,
  handleAdd,
  handleEdit,
  checkedRowKeys,
  onBatchDeleted
} = useTableOperate(data, getData);

// 删除用户
async function handleDelete(id: number) {
  const { error } = await fetchDeleteUser(id);
  if (!error) {
    await getData();
    window.$message?.success($t('common.deleteSuccess'));
  }
}

onMounted(() => {
  getData();
});
</script>

<template>
  <div class="min-h-500px flex-col-stretch gap-16px overflow-hidden lt-sm:overflow-auto">
    <!-- 搜索区域 -->
    <UserSearch v-model:model="searchParams" @reset="resetSearchQuery" @search="searchQuery" />
    
    <!-- 表格区域 -->
    <NCard
      :title="$t('page.manage.user.title')"
      :bordered="false"
      size="small"
      class="sm:flex-1-hidden card-wrapper"
    >
      <template #header-extra>
        <TableHeaderOperation
          v-model:columns="columnChecks"
          :disabled-delete="checkedRowKeys.length === 0"
          :loading="loading"
          @add="handleAdd"
          @delete="onBatchDeleted"
          @refresh="getData"
        />
      </template>
      
      <NDataTable
        v-model:checked-row-keys="checkedRowKeys"
        :columns="columns"
        :data="data"
        size="small"
        :flex-height="!appStore.isMobile"
        :scroll-x="962"
        :loading="loading"
        remote
        :row-key="row => row.id"
        :pagination="mobilePagination"
        class="sm:h-full"
      />
    </NCard>
    
    <!-- 用户操作抽屉 -->
    <UserOperateDrawer
      v-model:visible="drawerVisible"
      :operate-type="operateType"
      :row-data="editingData"
      @submitted="getData"
    />
  </div>
</template>
```

### 5.3 权限控制实现

#### 按钮级权限控制
```vue
<template>
  <NButton
    v-if="hasButtonPermission('user:add')"
    type="primary"
    @click="handleAdd"
  >
    添加用户
  </NButton>
</template>

<script setup lang="ts">
import { useAuthStore } from '@/store/modules/auth';

const authStore = useAuthStore();

// 检查按钮权限
function hasButtonPermission(permission: string) {
  return authStore.userInfo.buttons.includes(permission);
}
</script>
```

#### 数据权限控制
```typescript
// 在 API 请求中，后端根据用户角色返回相应的数据
export function fetchGetUserList(params?: Api.User.UserSearchParams) {
  return request<Api.User.UserList>({
    url: '/user/list',
    method: 'get',
    params
    // 后端会根据 JWT token 中的用户信息自动过滤数据
  });
}
```

### 5.4 菜单权限配置

#### 静态路由模式
在静态路由模式下，权限在前端配置：

```typescript
// src/router/elegant/routes.ts (自动生成)
{
  name: 'manage_user',
  path: '/manage/user',
  component: () => import('@/views/manage/user/index.vue'),
  meta: {
    title: '用户管理',
    i18nKey: 'route.manage_user',
    roles: ['admin', 'user-manager'], // 权限角色
    icon: 'ic:round-manage-accounts',
    order: 1
  }
}
```

#### 动态路由模式
在动态路由模式下，权限由后端控制：

```typescript
// 后端返回的路由数据格式
{
  "routes": [
    {
      "name": "manage_user",
      "path": "/manage/user",
      "meta": {
        "title": "用户管理",
        "roles": ["admin", "user-manager"],
        "icon": "ic:round-manage-accounts"
      }
    }
  ],
  "home": "home"
}
```

## 6. 环境配置

### 6.1 环境变量配置
```bash
# .env 文件
# 路由模式：static | dynamic
VITE_AUTH_ROUTE_MODE=dynamic

# 超级管理员角色
VITE_STATIC_SUPER_ROLE=super

# 服务响应码配置
VITE_SERVICE_SUCCESS_CODE=0000
VITE_SERVICE_LOGOUT_CODES=10000,10001
VITE_SERVICE_MODAL_LOGOUT_CODES=10002
VITE_SERVICE_EXPIRED_TOKEN_CODES=10003
```

### 6.2 代理配置
```typescript
// vite.config.ts
proxy: {
  '/api': {
    target: 'http://localhost:8080',
    changeOrigin: true,
    rewrite: path => path.replace(/^\/api/, '')
  }
}
```

## 7. 最佳实践

### 7.1 权限设计原则
1. **最小权限原则**: 用户只拥有完成工作所需的最小权限
2. **角色分离**: 通过角色管理权限，避免直接给用户分配权限
3. **前后端双重验证**: 前端控制 UI 显示，后端控制数据访问

### 7.2 错误处理
1. **统一错误处理**: 在请求拦截器中统一处理权限错误
2. **用户友好提示**: 权限不足时给出明确的提示信息
3. **优雅降级**: 无权限时隐藏相关功能而不是报错

### 7.3 性能优化
1. **路由懒加载**: 使用动态导入减少初始包大小
2. **权限缓存**: 缓存用户权限信息避免重复请求
3. **按需加载**: 根据用户权限只加载可访问的组件

### 7.4 安全考虑
1. **Token 安全**: 使用 HTTPS 传输，设置合理的过期时间
2. **XSS 防护**: 对用户输入进行转义处理
3. **CSRF 防护**: 使用 CSRF Token 或双重 Cookie 验证

## 8. 测试建议

### 8.1 权限测试
1. **正向测试**: 验证有权限的用户能正常访问
2. **负向测试**: 验证无权限的用户被正确拒绝
3. **边界测试**: 测试权限边界情况

### 8.2 路由测试
1. **直接访问测试**: 测试直接通过 URL 访问受保护路由
2. **刷新测试**: 测试页面刷新后权限是否正常
3. **Token 过期测试**: 测试 Token 过期后的处理逻辑

## 9. 故障排查

### 9.1 常见问题
1. **权限不生效**: 检查角色配置和路由守卫逻辑
2. **页面空白**: 检查路由配置和组件导入路径
3. **API 403 错误**: 检查 Token 是否有效和权限配置

### 9.2 调试技巧
1. **Vue DevTools**: 查看 Store 状态和路由信息
2. **Network 面板**: 检查 API 请求和响应
3. **Console 日志**: 添加适当的日志输出

通过以上详细的实现指南，您可以完整地在 SoybeanAdmin 中实现一个功能完善的用户列表管理系统，包含完整的权限控制、路由管理和 API 处理逻辑。