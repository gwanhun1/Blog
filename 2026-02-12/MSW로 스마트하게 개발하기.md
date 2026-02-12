## 1. 프론트엔드의 독립 선언, MSW란 무엇인가?

현대적인 웹 개발 프로세스에서 프론트엔드와 백엔드는 대개 병렬적으로 작업이 진행됩니다. 하지만 현실적으로 프론트엔드는 백엔드 API 없이는 화면에 데이터를 뿌려줄 수 없는 의존적인 구조를 갖게 됩니다. 여기서 발생하는 '대기 시간'은 프로젝트의 가장 큰 병목 구간입니다.

**MSW(Mock Service Worker)**는 이 고질적인 문제를 해결하기 위해 등장했습니다. MSW는 브라우저의 **Service Worker API**를 사용하여 실제 네트워크 요청을 가로채고, 가짜 응답(Mock Response)을 보내주는 라이브러리입니다. 단순히 코드 안에서 `fetch` 함수를 가로채는(Monkey Patching) 기존의 모킹 방식들과는 궤를 달리합니다.

## 2. 왜 MSW는 다른 모킹 방식보다 우월한가?

기존에는 주로 두 가지 방식을 사용했습니다.

* **Mock 데이터 하드코딩:** `const data = { ... }` 식으로 코드에 직접 박아넣는 방식입니다. 개발이 끝나면 일일이 지워야 하고, 실제 네트워크 환경을 테스트할 수 없습니다.
* **JSON Server 사용:** 별도의 로컬 서버를 띄워야 합니다. 엔드포인트가 실제 API와 다를 수 있고, 인증(Auth) 로직 등을 구현하기 번거롭습니다.

**반면, MSW는 다음과 같은 독보적인 장점을 제공합니다.**

### ① 코드의 순수성 (Zero Intrusion)

MSW는 네트워크 레벨에서 작동하므로, 비즈니스 로직에 가짜 데이터를 위한 코드를 단 한 줄도 섞을 필요가 없습니다. `if (dev) return mockData` 같은 조건문이 사라집니다.

### ② 브라우저 개발자 도구 활용

실제 네트워크 요청이 발생하기 때문에, 크롬 개발자 도구의 **Network 탭**에서 요청과 응답을 그대로 확인할 수 있습니다. API의 응답 속도 지연(Delay)이나 HTTP 상태 코드(404, 500 등)를 시뮬레이션하기에 최적입니다.

### ③ 테스트 코드와의 높은 호환성

브라우저 환경뿐만 아니라 Node.js 환경에서도 동작합니다. 즉, 개발 환경에서 사용한 모킹 핸들러를 그대로 Jest나 Vitest 같은 단위/통합 테스트 코드에서 재사용할 수 있어 효율적입니다.

---

## 3. MSW의 핵심 동작 원리

MSW의 마법은 **Service Worker**에서 일어납니다. 서비스 워커는 브라우저와 네트워크 사이에서 '프록시 서버' 역할을 수행하는 스크립트입니다.

1. **등록(Registration):** 애플리케이션 실행 시 MSW 서비스 워커가 브라우저에 등록됩니다.
2. **가로채기(Interception):** 애플리케이션이 `fetch`나 `axios`로 요청을 보내면, 서비스 워커가 이를 서버로 보내기 전 중간에서 가로챕니다.
3. **매칭(Matching):** 미리 정의된 핸들러 리스트를 조회하여 요청받은 URL과 매칭되는 것이 있는지 확인합니다.
4. **응답(Mocking):** 매칭된 핸들러가 있다면 정의된 Mock 응답을 브라우저에 돌려주고, 매칭되지 않는다면 실제 네트워크로 요청을 흘려보냅니다(Passthrough).

---

## 4. 실전 가이드: MSW 도입 및 활용

### Step 1: 핸들러 작성

가장 먼저 어떤 요청을 가로챌지 정의해야 합니다.

```javascript
// src/mocks/handlers.ts
import { http, HttpResponse, delay } from 'msw'

export const handlers = [
  http.get('/api/posts', async () => {
    // 실제 서버처럼 응답 지연 시뮬레이션
    await delay(500)
    
    return HttpResponse.json([
      { id: 1, title: 'MSW 도입기', author: 'Dev' },
      { id: 2, title: '서비스 워커의 이해', author: 'Admin' },
    ])
  }),

  http.post('/api/login', async ({ request }) => {
    const info = await request.json()
    if (info.id === 'admin') {
      return HttpResponse.json({ name: '관리자' }, { status: 200 })
    }
    return new HttpResponse(null, { status: 401 })
  }),
]

```

### Step 2: 서비스 워커 설정 및 실행

MSW는 전용 CLI를 통해 서비스 워커 스크립트를 생성합니다.

```bash
npx msw init public/ --save

```

이후 브라우저 환경에서 MSW를 활성화하는 로직을 엔트리 포인트(`main.tsx` 등)에 추가합니다.

```javascript
// src/main.tsx
async function enableMocking() {
  if (process.env.NODE_ENV !== 'development') {
    return
  }
  const { worker } = await import('./mocks/browser')
  return worker.start()
}

enableMocking().then(() => {
  renderApp()
})

```

---

## 5. 결론: MSW가 바꾸는 협업 문화.

MSW를 도입하면 단순히 기술적인 편의성만 좋아지는 것이 아닙니다. **"API 명세"가 확정되는 순간 프론트엔드와 백엔드가 각자 자기 할 일에 집중할 수 있는 환경**이 만들어집니다.

프론트엔드 개발자는 백엔드 API가 완성될 때까지 더 이상 기다릴 필요가 없습니다. 기획서상의 예외 케이스(데이터가 없을 때, 서버 에러가 날 때 등)를 핸들러에서 자유자재로 구성하며 미리 대응할 수 있습니다. 이는 결과적으로 프로젝트 전체의 개발 속도를 비약적으로 상승시킵니다.

아직도 API 완료 소식만 기다리며 더미 데이터를 만지고 계신가요? 이제 MSW로 진정한 개발의 자유를 누려보시길 바랍니다.

#MSW