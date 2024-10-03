Vue.js 애플리케이션에서 전역으로 `accessToken`이 없을 경우 특정 페이지로 리다이렉트하는 방법을 구현할 수 있습니다. 이를 위해 Vue Router의 **Navigation Guards**를 사용할 수 있습니다. 다음은 이를 설정하는 기본적인 방법입니다.

### 1. Vue Router 설정

먼저, `Vue Router`에서 `beforeEach` 네비게이션 가드를 사용하여 모든 경로 이동 전에 `accessToken`의 존재 여부를 확인합니다. `accessToken`이 없으면 `/main`으로 리다이렉트하고, 있으면 원래 요청한 페이지로 이동시킵니다.

```javascript
// src/router/index.js
import { createRouter, createWebHistory } from 'vue-router'
import Home from '../views/Home.vue'
import Main from '../views/Main.vue'
import { useAuthStore } from '@/stores/auth' // Pinia의 예시

const routes = [
  {
    path: '/',
    name: 'Home',
    component: Home,
  },
  {
    path: '/main',
    name: 'Main',
    component: Main,
  },
  // 다른 경로들...
]

const router = createRouter({
  history: createWebHistory(process.env.BASE_URL),
  routes,
})

router.beforeEach((to, from, next) => {
  const authStore = useAuthStore()
  const token = authStore.accessToken // Pinia를 사용하는 경우, IndexedDB에서 token을 가져올 수 있음

  if (!token && to.path !== '/main') {
    // accessToken이 없고, /main이 아닌 다른 경로로 접근할 때
    next({ path: '/main' }) // /main으로 리다이렉트
  } else {
    next() // 다른 경우는 정상적으로 요청 경로로 이동
  }
})

export default router
```

### 2. Pinia에서 accessToken 관리 (Optional)

Pinia를 사용하여 `accessToken`을 관리하는 예시입니다. IndexedDB에서 가져오거나 다른 방법으로 `accessToken`을 저장할 수 있습니다.

```javascript
// src/stores/auth.js
import { defineStore } from 'pinia'

export const useAuthStore = defineStore('auth', {
  state: () => ({
    accessToken: null, // 초기에는 null로 설정
  }),
  actions: {
    setToken(token) {
      this.accessToken = token
      // 여기서 IndexedDB 등에 저장할 수 있음
    },
    removeToken() {
      this.accessToken = null
      // 여기서 IndexedDB 등에서 삭제할 수 있음
    },
  },
})
```

### 3. 컴포넌트에서 AccessToken 설정

로그인 후 `accessToken`을 가져왔다면, 이를 `Pinia`를 통해 설정할 수 있습니다.

```javascript
// 로그인 성공 시 accessToken 설정
const authStore = useAuthStore()
authStore.setToken('받은AccessToken')
```

이렇게 하면 `accessToken`이 없는 경우 `/main`으로 리다이렉트되며, 토큰이 있는 경우에만 다른 페이지로 정상적으로 이동하게 됩니다.