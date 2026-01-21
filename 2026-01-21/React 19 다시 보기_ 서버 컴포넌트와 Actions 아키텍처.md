# React 19 다시 보기: 서버 컴포넌트와 Actions 아키텍처

2026년 1월, 현재 프론트엔드 생태계에서 React 19는 더 이상 '새로운 기술'이 아닙니다. Next.js, React Router(구 Remix), TanStack Start 등 주요 메타 프레임워크들이 모두 React 19의 스펙을 기반으로 안정화되었고, 우리 개발자들 또한 서버 컴포넌트(RSC)를 활용한 개발 방식에 꽤나 익숙해졌습니다.

도입 초기에는 "왜 굳이 컴포넌트를 서버와 클라이언트로 나눠야 하는가?"에 대한 논쟁도 있었지만, 이제는 이 아키텍처가 가져다주는 이점(성능, 보안, 개발 생산성)이 명확해졌습니다.

이번 글에서는 이제는 실무의 표준(Standard)이 된 **React 19의 핵심 아키텍처, 서버 컴포넌트와 Actions**를 다시 한번 정리해 봅니다.

---

## 1. 데이터 흐름의 변화: Fetching on Server

React 19 이전, 우리는 데이터를 가져오기 위해 클라이언트에서 `useEffect`를 사용하거나, React Query 같은 라이브러리에 전적으로 의존해야 했습니다. 하지만 RSC(React Server Components)의 도입으로 데이터 페칭의 위치가 **'브라우저'에서 '서버'로** 완전히 이동했습니다.

[Image of React Server Components architecture diagram showing server vs client boundary]

### Server Component의 아키텍처적 이점

* **Zero-Bundle Size:** 서버 컴포넌트 내부의 로직과 라이브러리(예: 무거운 날짜 처리 라이브러리, 마크다운 파서 등)는 클라이언트로 전송되지 않습니다. 결과 HTML(혹은 직렬화된 데이터)만 전송되므로 초기 로딩 속도가 비약적으로 상승했습니다.
* **Direct Backend Access:** 별도의 API Endpoint를 거치지 않고, 컴포넌트가 직접 DB나 내부 마이크로서비스에 접근할 수 있습니다.

### Code Pattern: 2026년의 표준

이제 `async/await`를 컴포넌트 레벨에서 사용하는 것이 자연스럽습니다.

```tsx
// app/dashboard/page.tsx
import db from '@/lib/db';
import { StatCard } from '@/components/stat-card';

// Server Component (Default)
export default async function DashboardPage() {
  // 별도의 API 호출 없이 서버 자원에 직접 접근
  const stats = await db.analytics.getDailyStats();

  return (
    <main>
      <h1>오늘의 지표</h1>
      <div className="grid gap-4">
        {stats.map((stat) => (
          <StatCard key={stat.id} title={stat.label} value={stat.value} />
        ))}
      </div>
    </main>
  );
}

## 2. 데이터 변이의 변화: Server Actions

데이터를 읽는 것(Read)이 서버 컴포넌트라면, 데이터를 쓰는 것(Write/Mutation)은 **Server Actions**가 담당합니다. 이는 과거의 `<form>` 태그 동작 방식을 현대적으로 재해석한 것입니다.

### 'use server'와 RPC

함수 상단에 `'use server'`를 선언하면, React는 자동으로 해당 함수를 서버 API 엔드포인트처럼 처리합니다. 클라이언트에서 이 함수를 호출하면, 마치 **RPC(Remote Procedure Call)**처럼 서버의 함수가 실행됩니다.

이는 `API Route 생성 -> fetch 요청 -> 응답 처리`로 이어지던 보일러플레이트 코드를 획기적으로 줄여주었습니다.

```tsx
// actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import db from '@/lib/db';

export async function createTodo(formData: FormData) {
  const title = formData.get('title');
  
  // 서버 사이드 로직 (DB 접근)
  await db.todo.create({
    data: { title: title as string, completed: false }
  });

  // 데이터 변경 후 해당 경로의 캐시를 갱신 (Next.js 예시)
  revalidatePath('/todos');
}
```

---

## 3. UX를 완성하는 Hooks: useActionState & useOptimistic

React 19는 서버 기반의 로직이 클라이언트의 UX를 해치지 않도록 강력한 훅들을 제공합니다. 이제는 필수가 된 두 가지 훅을 살펴봅시다.

### useActionState (구 useFormState)

Server Action의 결과 상태(로딩 중, 성공, 에러 메시지 등)를 관리하기 위해 `useState`를 덕지덕지 바를 필요가 없어졌습니다. 폼의 라이프사이클을 우아하게 관리할 수 있습니다.

```tsx
'use client';

import { useActionState } from 'react';
import { createTodo } from './actions';

export function TodoForm() {
  // state: 액션의 반환값, action: 폼 제출 함수, isPending: 로딩 상태
  const [state, action, isPending] = useActionState(createTodo, null);

  return (
    <form action={action}>
      <input name="title" required placeholder="할 일을 입력하세요" />
      <button type="submit" disabled={isPending}>
        {isPending ? '추가 중...' : '할 일 추가'}
      </button>
      
      {state?.error && <p className="text-red-500">{state.error}</p>}
    </form>
  );
}
```

### useOptimistic

서버의 응답을 기다리지 않고, UI를 먼저 업데이트하여 사용자에게 즉각적인 반응성을 제공합니다. (예: 좋아요 버튼, 댓글 작성, 채팅 등)

사용자는 네트워크 속도와 관계없이 앱이 "빠르다"고 느끼게 됩니다.

```tsx
'use client';

import { useOptimistic } from 'react';

export function LikeButton({ likes }: { likes: number }) {
  // optimisticLikes: 화면에 보여질 값
  // addOptimisticLike: 낙관적 업데이트를 트리거하는 함수
  const [optimisticLikes, addOptimisticLike] = useOptimistic(
    likes,
    (state, newLike) => state + newLike
  );

  return (
    <button 
      onClick={async () => {
        addOptimisticLike(1); // 1. UI 즉시 업데이트 (사용자는 기다리지 않음)
        await incrementLikeAction(); // 2. 백그라운드에서 실제 서버 요청
      }}
    >
      Likes: {optimisticLikes}
    </button>
  );
}
```

---

## 4. 마치며: '경계(Boundary)'를 이해하는 것이 핵심

React 19 이후의 개발은 **"Server와 Client의 경계(Boundary)를 어디에 둘 것인가?"**를 결정하는 과정이 되었습니다.

1.  **Server Component:** 데이터가 필요하고, SEO가 중요하며, 토큰/API 키 등 보안이 필요한 로직이 있는가?
2.  **Client Component:** `onClick`, `onChange` 같은 인터랙션이 필요하고, `window` 객체 등 브라우저 API를 쓰는가?

이 명확한 역할 분담은 애플리케이션의 유지보수성을 높이고, 성능 최적화를 자연스럽게 유도합니다. 2026년의 React는 단순히 UI 라이브러리를 넘어, 프론트엔드와 백엔드를 아우르는 거대한 **웹 애플리케이션 아키텍처**로 완성되었습니다.

아직 레거시 패턴(단순 데이터 페칭을 위한 `useEffect` 등)이 남아있는 프로젝트가 있다면, 점진적으로 RSC 패턴으로 전환해 보시기를 적극 권장합니다. 코드의 양이 줄어들고 구조가 단순해지는 경험을 하실 수 있을 것입니다.

---

**References**
* React Documentation (Server Components)
* React 19 Release Note
* Next.js App Router Documentation

#react