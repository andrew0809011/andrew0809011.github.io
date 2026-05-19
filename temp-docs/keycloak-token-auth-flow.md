# Keycloak Token 认证流程文档

> 本文档描述 B2B Copilot Web 前端如何集成 Keycloak 进行 SSO 登录、access token 获取与缓存、请求自动注入 Authorization header、token 过期后的刷新与重新登录，以及登录后回到原页面的重定向恢复机制。

---

## 整体流程概览

```
用户访问页面
    │
    ▼
[normal-form.tsx] 检测已有 session + isLoggedIn
    │ 已登录 → 直接跳转 /apps
    │ 未登录 ↓
    ▼
[sso-auth.tsx] 点击 SSO 登录按钮
    │
    ▼
[sso.ts] getUserOAuth2SSOUrl() 构造 Keycloak 授权 URL
    │
    ▼
浏览器跳转到 Keycloak 授权页面完成认证
    │
    ▼
Keycloak 回调 /api/auth/sso?code=xxx
    │
    ▼
[app/api/auth/sso/route.ts] 用 code 换取 session_id，写入 httpOnly cookie
    │
    ▼
跳转 /apps，登录完成
    │
    ▼
后续所有 API 请求：
[fetch.ts] beforeRequestAuthorization → fetchKeycloakAccessToken()
    │  命中缓存（20s 内）→ 直接使用
    │  缓存过期 → 请求 /sso/oauth2/access-token（携带 session cookie）
    ▼
请求头注入 Authorization: Bearer <access_token>
    │
    ▼
后端 API 验证 token
    │ 401（Keycloak token 过期，无法刷新）↓
    ▼
[fetch.ts] fetchKeycloakAccessToken 401 handler
    │  1. 保存当前 URL 到 localStorage['oauth_authorize_pending']（3 分钟 TTL）
    │  2. 清除 has_sso_session cookie（防止自动跳过 SSO 验证死循环）
    │  3. 跳转 /signin
    ▼
用户重新 SSO 登录 → Keycloak → /api/auth/sso → /apps
    │
    ▼
[app-initializer.tsx] AppInitializer.useEffect
    │  resolvePostLoginRedirect → 读 oauth_authorize_pending
    └─ location.replace(savedUrl) → 回到原页面 ✅
```

---

## 详细流程说明

### 1. 登录入口检测 — `app/signin/normal-form.tsx`

**文件路径：** `web/app/signin/normal-form.tsx`

登录页初始化时，`init()` 函数检查是否已登录：

```typescript
// 检查浏览器 cookies 中是否有 session
const hasSession = document.cookie.includes('session=')

if (isLoggedIn && hasSession) {
  setIsRedirecting(true)
  const redirectUrl = resolvePostLoginRedirect(searchParams)
  router.replace(redirectUrl || '/apps')
  return
}
```

- `isLoggedIn`：来自 `useIsLogin()` hook，调用后端 `/console/api/setup` 检查登录状态
- `hasSession`：检查浏览器 cookie 中是否存在 `session=` 字段（自定义扩展，避免仅凭 isLoggedIn 的误判）
- 两者都满足才认为已登录，直接跳转

---

### 2. SSO 登录触发 — `app/signin/components/sso-auth.tsx`

**文件路径：** `web/app/signin/components/sso-auth.tsx`

用户点击 "Sign in with SSO" 按钮后，根据协议类型处理。B2B 使用 OAuth2 协议：

```typescript
else if (protocol === SSOProtocol.OAuth2) {
  setRedirect(searchParams.get('redirect_path'))  // 保存登录前的页面路径
  getUserOAuth2SSOUrl().then((url) => {
    window.location.href = url  // 跳转到 Keycloak 授权页
  })
}
```

- `setRedirect`：使用 `useSigninRedirect` hook，将 `redirect_path` 存入 `localStorage`，以便登录成功后回到原页面

---

### 3. 构造 Keycloak 授权 URL — `service/sso.ts`

**文件路径：** `web/service/sso.ts`

```typescript
export const getUserOAuth2SSOUrl = async (): Promise<string> => {
  return new Promise((resolve) => {
    const redirectUriOrigin = process.env.NEXT_PUBLIC_WEB_PREFIX as string;
    const params = {
      client_id: process.env.KEYCLOAK_CLIENT_ID,
      response_type: "code",
      redirect_uri: `${redirectUriOrigin}/api/auth/sso`,  // 回调地址
      scope: "openid",
    };
    const url = new URL(
      `${process.env.KEYCLOAK_BASE_URL}/auth/realms/${process.env.KEYCLOAK_REALM}/protocol/openid-connect/auth`
    );
    Object.entries(params).forEach(([key, value]) => {
      url.searchParams.append(key, value as string);
    });
    resolve(url.toString());
  });
};
```

**关键环境变量：**

| 变量 | 说明 |
|---|---|
| `KEYCLOAK_BASE_URL` | Keycloak 服务器地址 |
| `KEYCLOAK_REALM` | Keycloak Realm 名称 |
| `KEYCLOAK_CLIENT_ID` | Keycloak 客户端 ID |
| `NEXT_PUBLIC_WEB_PREFIX` | 前端部署域名，用于构造回调地址 |

---

### 4. OAuth2 回调处理 — `app/api/auth/sso/route.ts`

**文件路径：** `web/app/api/auth/sso/route.ts`

Keycloak 认证成功后，携带 `code` 参数回调到 `/api/auth/sso`：

```typescript
export async function GET(request: NextApiRequest) {
  const authorizationCode = new URL(request.url!).searchParams.get("code");

  // 1. 用 code 换取 session_id（调用后端 API）
  const serverResponse = await fetch(
    `${getLoginApiPrefix()}/console/api/sso/oauth2/callback?${params.toString()}`
  );

  const responseData = await serverResponse.json();
  const sessionId = responseData.data?.session_id;

  // 2. 将 session_id 写入 httpOnly cookie
  response.cookies.set("session_id", sessionId, {
    httpOnly: true,
    sameSite: "lax",
    secure: isProd,
    maxAge: 24 * 60 * 60,  // 24 小时
    path: "/",
  });

  // 3. 写入一个非 httpOnly 的标记 cookie，供前端 JS 检测是否已有 session
  response.cookies.set("has_sso_session", "1", {
    httpOnly: false,
    sameSite: "lax",
    ...
  });

  // 4. 跳转到 /apps
  return NextResponse.redirect(new URL(`${redirectUriOrigin}/apps`));
}
```

**Cookie 设计说明：**
- `session_id`：httpOnly，不能被 JS 读取，安全性高，由后端验证
- `has_sso_session`：非 httpOnly，仅作为"已有 session"的标志位，供前端 `normal-form.tsx` 检测用，不含敏感信息
- **不设置 domain**：避免 INT/PROD 多环境共享 cookie 互相覆盖

---

### 5. Access Token 获取与缓存 — `service/fetch.ts`

**文件路径：** `web/service/fetch.ts`

这是 Keycloak 认证流程的核心模块。每次发 API 请求前，自动获取短时效的 Keycloak access token 并注入 Authorization header。

#### 5.1 缓存机制 — `createKeycloakAccessTokenFetcher`

```typescript
function createKeycloakAccessTokenFetcher() {
  const cacheDuration = 20000  // 20 秒缓存
  let cachedAccessToken: undefined | string = undefined
  let tokenTimestamp: undefined | number = undefined
  let isFetching = false

  return async function getAccessToken() {
    const now = Date.now()

    // 命中缓存，直接返回
    if (cachedAccessToken && tokenTimestamp && now - tokenTimestamp < cacheDuration) {
      return cachedAccessToken
    }

    // 正在获取中，等待完成（防并发重复请求）
    if (isFetching) {
      while (isFetching) {
        await new Promise(resolve => setTimeout(resolve, 100))
      }
      return cachedAccessToken!
    }

    isFetching = true
    try {
      // 调用后端接口，携带 session cookie，后端通过 session 换取 Keycloak access token
      const response = await fetch(`${API_PREFIX}/sso/oauth2/access-token`, {
        credentials: 'include',
      })
      const data = await response.json()
      cachedAccessToken = data.access_token
      tokenTimestamp = now
    } catch (error) {
      console.error(error)
      // 获取失败不抛出，由后端统一处理 401
    } finally {
      isFetching = false
    }
    return cachedAccessToken
  }
}

export const fetchKeycloakAccessToken = createKeycloakAccessTokenFetcher()
```

**设计要点：**
- **20 秒缓存**：Keycloak access token 短时效（默认 5 分钟），前端每 20 秒最多刷新一次，避免过多请求
- **防并发锁**：`isFetching` 标志位确保多个请求同时触发时只发一次 HTTP 请求
- **后端托管**：access token 不由前端直接持有，而是通过 `session_id` cookie 让后端 `/sso/oauth2/access-token` 来返回当前有效的 token（后端负责 token 刷新）

#### 5.2 请求前自动注入 — `beforeRequestAuthorization`

```typescript
const beforeRequestAuthorization: BeforeRequestHook = async (request) => {
  const keycloakAccessToken = await fetchKeycloakAccessToken()

  try {
    const requestUrl = new URL(request.url)
    const apiUrl = new URL(process.env.NEXT_PUBLIC_API_PREFIX as string || ...)
    // 只对同域 API 请求注入 Authorization header
    if (requestUrl.hostname === apiUrl.hostname) {
      request.headers.set('Authorization', `Bearer ${keycloakAccessToken}`)
    }
  } catch (error) {
    console.error(...)
  }
}
```

此 hook 注册在 `ky` HTTP 客户端的 `beforeRequest` 钩子中，所有通过 `base()` 函数发出的请求均自动执行，无需每个接口单独处理。

---

### 6. Token 过期刷新与重新登录 — `service/refresh-token.ts`

**文件路径：** `web/service/refresh-token.ts`

当 API 请求返回 401 时，触发 token 刷新逻辑。

#### 6.1 刷新流程

```typescript
async function getNewAccessToken(timeout: number): Promise<void> {
  // 检查其他标签页是否正在刷新（跨标签页防重复）
  const isRefreshingSign = globalThis.localStorage.getItem(LOCAL_STORAGE_KEY)
  if ((isRefreshingSign === '1' && isRefreshingSignAvailable(timeout)) || isRefreshing) {
    await waitUntilTokenRefreshed()  // 等待其他标签页刷新完成
    return
  }

  // 加锁
  isRefreshing = true
  globalThis.localStorage.setItem(LOCAL_STORAGE_KEY, '1')
  globalThis.localStorage.setItem('last_refresh_time', Date.now().toString())

  // 调用刷新接口（基于 cookie 中的 refresh token，无需 body）
  const [error, ret] = await fetchWithRetry(globalThis.fetch(`${API_PREFIX}/refresh-token`, {
    method: 'POST',
    credentials: 'include',
    headers: { 'Content-Type': 'application/json;utf-8' },
  }))

  if (error || ret.status === 401) {
    return Promise.reject(error || ret)
  }
}
```

**跨标签页防重复设计：**
- 使用 `localStorage` 的 `is_other_tab_refreshing` 键作为跨标签页锁
- 如果检测到其他标签页正在刷新，当前标签页等待（轮询 `localStorage`），避免多个标签页同时发起刷新请求

#### 6.2 超时控制

```typescript
export async function refreshAccessTokenOrRelogin(timeout: number) {
  return Promise.race([
    // 超时取消
    new Promise<void>((_, reject) => setTimeout(() => {
      releaseRefreshLock()
      reject(new Error('request timeout'))
    }, timeout)),
    // 实际刷新
    getNewAccessToken(timeout),
  ])
}
```

`timeout` 传入 `TIME_OUT = 100000`（100 秒），超时后释放锁并报错。

---

### 7. 401 处理与重新登录 — `service/base.ts`

**文件路径：** `web/service/base.ts`

#### 7.1 普通请求 (`request` 函数)

```typescript
if (errResp.status === 401) {
  // ...特殊 code 处理（not_setup, not_init_validated 等）

  // 尝试刷新 token
  const [refreshErr] = await asyncRunSafe(refreshAccessTokenOrRelogin(TIME_OUT))
  if (refreshErr === null)
    return baseFetch<T>(url, options, otherOptionsForBaseFetch)  // 刷新成功，重试原请求

  // 刷新失败，跳转登录页
  if (location.pathname !== `${basePath}/signin` || !IS_CE_EDITION) {
    jumpTo(loginUrl)
    return Promise.reject(err)
  }
}
```

#### 7.2 SSE 流式请求 (`ssePost` 函数)

```typescript
if (res.status === 401) {
  if (!isPublicAPI) {
    refreshAccessTokenOrRelogin(TIME_OUT).then(() => {
      ssePost(url, fetchOptions, otherOptions)  // 刷新成功，重试 SSE 请求
    }).catch((err) => {
      console.error(err)
      // 刷新失败时不主动跳转，依赖用户下次操作触发
    })
  }
}
```

---

### 8. 登录重定向恢复 — `app/signin/utils/post-login-redirect.ts`

**文件路径：** `web/app/signin/utils/post-login-redirect.ts`

登录成功后，恢复用户登录前的页面：

```typescript
export const resolvePostLoginRedirect = (searchParams: ReadonlyURLSearchParams) => {
  // 优先取 URL 参数中的 redirect_url
  const redirectUrl = searchParams.get(REDIRECT_URL_KEY)
  if (redirectUrl) {
    return decodeURIComponent(redirectUrl)
  }

  // 其次从 localStorage 中取（OAuth authorize 流程使用）
  return getItemWithExpiry(OAUTH_AUTHORIZE_PENDING_KEY)
}
```

配合 `useSigninRedirect` hook：

```typescript
// hooks/use-signin-redirect.ts
const setRedirect = (path?: string | null) => {
  if (typeof path === 'string') {
    localStorage.setItem('redirect_path', path)
  }
}
```

---

### 9. Token 401 时的登录重定向恢复 — `service/fetch.ts` + `app/components/app-initializer.tsx`

当 `/sso/oauth2/access-token` 返回 401（无法刷新 Keycloak token），除了跳转 signin，还需要在登录后回到用户原来的页面。

#### 9.1 写入重定向目标（`fetch.ts`）

```typescript
if (response.status === 401) {
  cachedAccessToken = undefined
  tokenTimestamp = undefined
  if (globalThis.location?.pathname !== `${basePath}/signin`) {
    // 1. 记录当前页面路径，登录后恢复（不设过期时间）
    const currentUrl = globalThis.location.pathname + globalThis.location.search
    globalThis.localStorage?.setItem(
      'oauth_authorize_pending',
      JSON.stringify({ value: currentUrl }),
    )
    // 2. 清除非 httpOnly 的 session 标记 cookie
    //    防止 normal-form.tsx 因 Dify session 仍有效而自动跳过 SSO 重新验证形成死循环
    globalThis.document && (globalThis.document.cookie = 'has_sso_session=; Max-Age=0; path=/')
    // 3. 跳转登录页
    globalThis.location.href = `${globalThis.location.origin}${basePath}/signin`
  }
}
```

**为什么需要清除 `has_sso_session`：**
- `normal-form.tsx` 通过 `document.cookie.includes('session=')` 判断是否有 session
- `has_sso_session=1` 这个字符串包含子串 `session=`，所以能匹配
- 若 Dify session（24h）仍有效但 Keycloak token 过期，不清这个 cookie 会导致：`normal-form.tsx → isLoggedIn && hasSession=true → 直接跳回 /apps → token 依然 401 → 死循环`

#### 9.2 读取并恢复（`AppInitializer`）

`AppInitializer` 包裹所有主要页面（`(commonLayout)/layout.tsx`），SSO 登录完成后用户到达 `/apps` 时触发：

```typescript
// app/components/app-initializer.tsx
useEffect(() => {
  (async () => {
    const isFinished = await isSetupFinished()
    if (!isFinished) { router.replace('/install'); return }

    const redirectUrl = resolvePostLoginRedirect(searchParams)
    if (redirectUrl) {
      location.replace(redirectUrl)   // ← 回到原页面
      return
      // setInit(true) 不会执行，children 不渲染（避免闪屏）
    }
    setInit(true)  // 正常展示页面
  })()
}, [pathname, searchParams, ...])
```

`resolvePostLoginRedirect` 的优先级：
1. URL 参数 `?oauth_redirect_url=...`（无）
2. `localStorage['oauth_authorize_pending']`（读取后自动删除，防止脏数据残留）

#### 9.3 完整链路

```
用户在 /apps/abc123/workflow
    ↓ Keycloak token 过期
fetchKeycloakAccessToken → 401
    ├─ localStorage['oauth_authorize_pending'] = { value: '/apps/abc123/workflow' }
    ├─ has_sso_session cookie 清除
    └─ location.href = '/signin'
    ↓
normal-form.tsx: isLoggedIn && hasSession=false → 不自动跳转 → 展示 SSO 按钮
    ↓
用户 SSO 登录 → Keycloak → /api/auth/sso → set session cookies → redirect /apps
    ↓
AppInitializer.useEffect → resolvePostLoginRedirect
    → getItemWithExpiry('oauth_authorize_pending')
    → { value: '/apps/abc123/workflow' }（不过期）
    → localStorage.removeItem('oauth_authorize_pending')（读取后自动清理）
    → location.replace('/apps/abc123/workflow') ✅
```

---

## 相关文件汇总

| 文件 | 说明 |
|---|---|
| `web/service/fetch.ts` | 核心：`fetchKeycloakAccessToken`（缓存 + 401 → 保存路径 → 跳转登录）、`beforeRequestAuthorization`（自动注入 header） |
| `web/service/refresh-token.ts` | token 刷新逻辑，含跨标签页防重复机制 |
| `web/service/base.ts` | 401 响应处理，调用刷新或跳转登录 |
| `web/service/sso.ts` | 构造 Keycloak 授权 URL |
| `web/app/api/auth/sso/route.ts` | OAuth2 回调处理，code 换 session_id，写入 cookie |
| `web/app/signin/normal-form.tsx` | 登录页初始化，检测已有 session 直接跳转 |
| `web/app/signin/components/sso-auth.tsx` | SSO 登录按钮，触发跳转 Keycloak |
| `web/app/components/app-initializer.tsx` | 登录后重定向恢复，读取 `oauth_authorize_pending` |
| `web/app/signin/utils/post-login-redirect.ts` | 登录成功后重定向目标解析（支持 URL 参数和 localStorage） |
| `web/app/account/oauth/authorize/constants.ts` | `oauth_authorize_pending`、`oauth_redirect_url` 常量定义 |
| `web/hooks/use-signin-redirect.ts` | 保存/恢复登录前页面路径（redirect_path key） |
| `web/app/components/support/nfs/service.ts` | NFS 服务，手动调用 `fetchKeycloakAccessToken` |
| `web/service/apps.ts` | 部分 raw fetch 调用手动注入 token（ZIP 导出等） |
| `web/service/langgraph-chat.ts` | LangGraph 聊天服务，动态 import 并使用 token |

---

## 环境变量依赖

| 变量名 | 用途 |
|---|---|
| `KEYCLOAK_BASE_URL` | Keycloak 服务器根地址 |
| `KEYCLOAK_REALM` | Keycloak Realm |
| `KEYCLOAK_CLIENT_ID` | OAuth2 客户端 ID |
| `NEXT_PUBLIC_WEB_PREFIX` | 前端部署域名（用于构造 redirect_uri 和 cookie domain） |
| `NEXT_PUBLIC_API_PREFIX` | 后端 API 前缀（用于判断是否注入 Authorization header） |
| `NEXT_PUBLIC_DEPLOY_ENV` | 部署环境标识，`B2BEDIPROD` 时启用 secure cookie |

---

## 关键设计决策

1. **token 不存 localStorage**：access token 通过 session cookie → 后端 `/sso/oauth2/access-token` 接口按需获取，避免 XSS 读取风险
2. **20 秒前端缓存**：减少频繁调用 token 接口，同时确保 token 接近过期时能自动刷新
3. **不设置 cookie domain**：防止 INT/PROD 多环境的 session cookie 互相覆盖
4. **跨标签页刷新锁**：`localStorage` 标志位确保同一浏览器多标签页不重复刷新
5. **`beforeRequestAuthorization` hook**：通过 `ky` 的 beforeRequest 统一注入，避免每个接口单独处理
6. **hostname 匹配过滤**：只对与 `NEXT_PUBLIC_API_PREFIX` 同域的请求注入 Authorization，防止 token 泄露到第三方域
7. **Token 401 保存路径 + 清除 has_sso_session**：Keycloak token 彻底失效时，先保存当前路径到 `oauth_authorize_pending`（不设过期时间），再清除 `has_sso_session` cookie 强制走 SSO 重新验证，避免 Dify session 仍有效时绕过 SSO 形成死循环；SSO 完成后由 `AppInitializer` 读取路径并恢复
