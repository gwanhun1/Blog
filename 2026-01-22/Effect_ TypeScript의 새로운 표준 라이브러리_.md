# TypeScript의 미래, Effect 라이브러리 찍먹하기 (feat. 에러 핸들링)

2026년 현재, 프론트엔드와 백엔드를 막론하고 TypeScript 진영에서 가장 뜨거운 논쟁의 중심에는 **Effect**가 있습니다.

누군가는 "러닝 커브가 너무 높은 함수형 프로그래밍 놀이터"라고 말하지만, 이미 많은 선도적인 팀들은 "이것이야말로 TypeScript에 빠져 있던 **진짜 표준 라이브러리(Standard Library)**"라며 프로덕션 코드를 Effect로 다시 짜고 있습니다.

도대체 Effect가 뭐길래 `lodash`나 `axios`를 넘어, 아예 코드를 짜는 방식 자체를 바꾸려 하는 걸까요? 오늘은 Effect가 주목받는 가장 큰 이유인 **'타입 안전한 에러 핸들링'**을 중심으로 그 매력을 알아보겠습니다.

---

## 1. 우리는 여전히 "불안한" TypeScript를 쓰고 있다

TypeScript가 정적 타이핑으로 많은 버그를 잡아주지만, 여전히 해결하지 못한 난제가 하나 있습니다. 바로 **런타임 에러(Runtime Error)**와 **예외 처리(Exception Handling)**입니다.

일반적인 TypeScript 코드를 봅시다.

```ts
// 이 함수는 에러를 던질까요? 타입 시그니처만 봐서는 알 수 없습니다.
async function getUser(id: string): Promise<User> {
  if (!id) throw new Error("ID Required"); // 1. 숨겨진 에러
  const res = await fetch(\`/api/users/\${id}\`);
  if (!res.ok) throw new Error("Fetch Failed"); // 2. 또 다른 숨겨진 에러
  return res.json();
}

async function main() {
  try {
    const user = await getUser("123");
  } catch (e) {
    // 3. 여기서 'e'는 'unknown' 또는 'any'입니다.
    // 어떤 에러가 터졌는지 타입 시스템은 전혀 모릅니다.
    console.error(e); 
  }
}
```

위 코드의 문제점은 명확합니다.

1.  **숨겨진 에러:** 함수의 `return` 타입(`Promise<User>`)만으로는 실패 가능성을 알 수 없습니다.
2.  **타입 소실:** `catch (e)` 블록에서 개발자는 `e`가 무엇인지 추측해서 코드를 짜야 합니다.
3.  **안전장치 부재:** 실수로 `try-catch`를 빼먹으면 앱이 멈춥니다.

---

## 2. Effect: 성공과 실패를 모두 타입에 담다

Effect는 이 문제를 **"성공과 실패, 그리고 의존성까지 모두 타입에 명시하자"**는 철학으로 해결합니다.

Effect의 기본 타입은 다음과 같습니다.
`Effect<Success, Error, Requirements>`

* **Success:** 성공했을 때 반환되는 값 (예: `User`)
* **Error:** 실패했을 때 반환되는 **구체적인 에러 타입** (예: `NetworkError | IndexError`)
* **Requirements:** 실행에 필요한 의존성 (예: `DatabaseService`)

### Effect로 다시 쓴 코드

이제 똑같은 로직을 Effect로 작성해 보겠습니다.

```typescript
import { Effect, Console } from "effect";

// 커스텀 에러 클래스 정의
class FetchError {
  readonly _tag = "FetchError";
}

class JsonParseError {
  readonly _tag = "JsonParseError";
}

// 1. 에러 타입이 명확히 드러납니다.
// 타입: Effect.Effect<User, FetchError | JsonParseError>
const getUser = (id: string) =>
  Effect.tryPromise({
    try: () => fetch(`/api/users/${id}`).then(res => res.json()),
    catch: () => new FetchError() // 에러를 명시적으로 매핑
  });

const program = Effect.gen(function* (_) {
  // 2. 제너레이터 문법으로 동기 코드처럼 작성합니다 (async/await 대체)
  const user = yield* _(getUser("123"));
  
  yield* _(Console.log(`User found: ${user.name}`));
});

// 3. 실행 시점에 모든 에러를 처리해야만 컴파일이 됩니다.
const runnable = program.pipe(
  Effect.catchTags({
    FetchError: () => Console.error("네트워크 에러가 발생했습니다."),
    JsonParseError: () => Console.error("데이터 형식이 잘못되었습니다."),
  })
);

Effect.runPromise(runnable);
```

**무엇이 바뀌었나요?**

* **타입 안전성:** `getUser` 함수를 호출하는 쪽에서는 반드시 `FetchError`와 `JsonParseError`를 처리해야 한다는 것을 타입스크립트가 알려줍니다. (처리하지 않으면 빨간 줄이 그어집니다!)
* **파이프라인:** `pipe`와 `Effect.gen`을 통해 데이터 흐름과 에러 처리를 레고 블록 조립하듯 연결할 수 있습니다.
* **가독성:** `try-catch` 지옥 없이, 성공 흐름과 실패 흐름이 깔끔하게 분리됩니다.

---

## 3. 단순 에러 핸들링 그 이상

Effect는 단순히 에러만 잡는 도구가 아닙니다. Node.js나 브라우저 환경에서 필요한 모든 유틸리티를 제공하는 거대한 생태계입니다.

* **동시성 제어 (Concurrency):** `Promise.all`보다 훨씬 강력한 동시성 제어 기능을 제공합니다. API 요청 10개 중 3개만 먼저 실행하고, 하나라도 실패하면 나머지를 자동 취소하는 등의 로직을 단 몇 줄로 구현할 수 있습니다.
* **의존성 주입 (Dependency Injection):** React의 Context API나 NestJS의 DI 시스템처럼, 컴포넌트나 함수에 필요한 서비스(DB, API 클라이언트 등)를 주입하고 테스트하기 쉽게 만들어줍니다.
* **재시도 로직 (Retry):** `Effect.retry(task, Schedule.exponential(1000))` 한 줄이면, API 요청 실패 시 지수 백오프로 재시도하는 로직이 완성됩니다.

---

## 4. 마치며: 러닝 커브, 넘을 가치가 있을까?

솔직히 말씀드리면, Effect는 어렵습니다. 함수형 프로그래밍의 용어들(Option, Either, Pipe 등)에 익숙해져야 하고, 사고방식을 바꿔야 합니다. 간단한 토이 프로젝트나 랜딩 페이지에 도입하기에는 과한 도구일 수 있습니다.

하지만 **"유지보수 가능한, 절대 죽지 않는, 규모 있는 애플리케이션"**을 만들어야 하는 2026년의 실무 환경에서 Effect는 대체 불가능한 강력함을 제공합니다.

`try-catch` 안에서 `any` 타입의 에러와 씨름하는 데 지치셨나요? 그렇다면 이제 Effect를 찍먹해 볼 시간입니다.

---

**References**
* Effect 공식 문서
* Effect Discord Community

#typescript