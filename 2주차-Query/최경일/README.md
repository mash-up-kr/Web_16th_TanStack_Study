# 팩토리 패턴의 진화

## 1️⃣ 기존 패턴 - queryKey/queryFn 흩어진 코드

`useQuery`를 사용하면 `data`, `isPending`, `error` 같은 서버 상태를 클라이언트가 직접 관리할 필요 없이 TanStack Query에 위임할 수 있다. 보일러플레이트가 줄고 책임 분리도 자연스럽게 된다.

```tsx
// A 컴포넌트 — 유저 정보
const { data, isPending, error } = useQuery({
  queryKey: ["user", userId],
  queryFn: () => fetchUser(userId),
});

// B 컴포넌트 — 포스트 정보
const { data, isPending, error } = useQuery({
  queryKey: ["posts", userId],
  queryFn: () => fetchUserPosts(userId),
});

// C 컴포넌트 — 알림 정보
const { data, isPending, error } = useQuery({
  queryKey: ["notifications", userId],
  queryFn: () => fetchNotifications(userId),
});
```

문제는 `queryKey`와 `queryFn`을 매번 인라인으로 선언할 때 생긴다.

**중복 선언 → 수정 누락**

같은 쿼리를 여러 컴포넌트에서 쓴다면, 한 쪽을 수정할 때 다른 쪽도 찾아가야 한다. 파일이 늘어날수록 이 비용은 선형으로 증가한다.

```tsx
useQuery({ queryKey: ["user", userId], queryFn: () => fetchUser(userId) });
useQuery({ queryKey: ["user", userId], queryFn: () => fetchUser(userId) });
// → fetchUser 시그니처가 바뀌면 선언된 모든 곳을 직접 찾아가야 함
```

**오타/불일치 → 캐시 분기**

키를 문자열 리터럴로 직접 쓰면 오타 위험이 높다. TanStack Query는 키 기준으로 캐시를 식별하기 때문에, 오타 하나가 캐시를 통째로 분기시킨다.

```tsx
useQuery({ queryKey: ['user', userId], ... })   // A 컴포넌트
useQuery({ queryKey: ['users', userId], ... })  // B 컴포넌트 — 's' 하나 차이
// → 같은 데이터인데 캐시가 따로 생기고, 요청이 두 번 날아감
```

**키 구조 변경 → invalidation 누락**

키 구조가 바뀌면 `invalidateQueries` 호출부도 같이 업데이트해야 하는데, 이 연결고리가 코드 어디에도 명시되어 있지 않다.

```tsx
// 키 구조를 바꿨는데
queryKey: ["user", "detail", userId];

// invalidate 쪽은 예전 키 그대로 — 캐시가 안 날아감
queryClient.invalidateQueries({ queryKey: ["user", userId] });
```

인라인 선언은 쿼리가 늘어날수록 중복·오타·누락이 복합적으로 터진다.

<br/>

## 2️⃣ 쿼리 키 팩토리 등장

앞서 본 문제들의 공통 원인은 하나다 — `queryKey`가 코드 여기저기에 흩어져 있다는 것. 이걸 한 곳에서 관리하자는 아이디어가 **쿼리 키 팩토리 패턴**이다.

기본 사용법

키를 반환하는 객체를 하나 만들고, 모든 컴포넌트가 이걸 참조하게 한다.

```tsx
// queryKeys.ts
export const userKeys = {
  all: ["user"] as const,
  detail: (userId: string) => ["user", userId] as const,
  posts: (userId: string) => ["user", userId, "posts"] as const,
};
```

```tsx
// A 컴포넌트
const { data } = useQuery({
  queryKey: userKeys.detail(userId),
  queryFn: () => fetchUser(userId),
});

// B 컴포넌트
const { data } = useQuery({
  queryKey: userKeys.posts(userId),
  queryFn: () => fetchUserPosts(userId),
});

// 무효화
queryClient.invalidateQueries({ queryKey: userKeys.all });
// → 'user'로 시작하는 캐시 전부 날아감
```

키가 한 곳에 모이니 오타 위험이 줄고, 수정할 때도 `queryKeys.ts` 하나만 열면 된다.

**근데 키 팩토리도 너무 커진다면..?**

Before

```tsx
// queries/keys.ts — 모든 도메인 키가 한 파일에
export const userKeys = {
  all: ["user"] as const,
  detail: (id: string) => ["user", id] as const,
  posts: (id: string) => ["user", id, "posts"] as const,
};

export const postKeys = {
  all: ["post"] as const,
  detail: (id: string) => ["post", id] as const,
  comments: (id: string) => ["post", id, "comments"] as const,
};

export const notificationKeys = {
  all: ["notification"] as const,
  unread: () => ["notification", "unread"] as const,
  byUser: (id: string) => ["notification", id] as const,
};

// ... 도메인 늘어날수록 계속 길어짐
```

키 문자열을 직접 반복해서 쓰다 보니 `all`과 `detail`, `posts`가 연관된 키라는 게 구조에서 드러나지 않는다. 파일도 하나라 도메인이 늘어날수록 한 파일에서 수백 줄을 관리하게 된다. `invalidateQueries`로 어디까지 날아가는지도 직접 추적해봐야 알 수 있다.

After

```tsx
// queries/userKeys.ts
// all을 함수로 만드는 이유: 하위 키들이 spread로 참조하기 위해
export const userKeys = {
  all: () => ["user"] as const,
  detail: (id: string) => [...userKeys.all(), id] as const,
  posts: (id: string) => [...userKeys.detail(id), "posts"] as const,
};

// queries/postKeys.ts
export const postKeys = {
  all: () => ["post"] as const,
  detail: (id: string) => [...postKeys.all(), id] as const,
  comments: (id: string) => [...postKeys.detail(id), "comments"] as const,
};

// queries/notificationKeys.ts
export const notificationKeys = {
  all: () => ["notification"] as const,
  unread: () => [...notificationKeys.all(), "unread"] as const,
  byUser: (id: string) => [...notificationKeys.all(), id] as const,
};
```

상위 키를 spread로 참조하니 계층 관계가 코드에서 직접 보인다. `invalidateQueries`의 범위도 구조에서 바로 예측된다.

```tsx
// 특정 유저 관련 캐시 전부 무효화
queryClient.invalidateQueries({ queryKey: userKeys.detail(userId) });
// → ['user', userId], ['user', userId, 'posts'] 전부 무효화

// post 도메인만 무효화
queryClient.invalidateQueries({ queryKey: postKeys.all() });
// → ['post']로 시작하는 캐시 전부 무효화
```

도메인별로 파일을 나누니 팀 단위 작업 시 충돌 가능성도 줄고, 어느 파일에서 어떤 키를 관리하는지도 명확해진다.

<br/>

## 3️⃣ 키만 관리하는 v4의 한계

계층적 키 팩토리로 관리 포인트는 줄었지만, 한 가지 문제가 남아 있다.

```tsx
// queries/userKeys.ts
export const userKeys = {
  all: () => ["user"] as const,
  detail: (id: string) => [...userKeys.all(), id] as const,
};
```

```tsx
// UserProfile.tsx
const { data } = useQuery({
  queryKey: userKeys.detail(userId),
  queryFn: () => fetchUser(userId), // fn은 여전히 여기
  staleTime: 1000 * 60 * 5, // options도 여전히 여기
});

// UserCard.tsx — 같은 쿼리를 또 씀
const { data } = useQuery({
  queryKey: userKeys.detail(userId),
  queryFn: () => fetchUser(userId), // 또 선언
  staleTime: 1000 * 60 * 5, // 또 선언
});
```

키는 한 곳에 모았는데, `queryFn`과 `staleTime` 같은 옵션은 여전히 각 컴포넌트에 흩어져 있다. 결국 **key만 팩토리화된 것**이고, fn과 options는 기존 안티패턴과 다를 게 없다.

`fetchUser` 시그니처가 바뀌거나 `staleTime`을 조정해야 할 때, 이 쿼리를 쓰는 모든 컴포넌트를 다시 찾아가야 한다. 키 팩토리가 해결한 문제와 똑같은 문제가 fn/options 레벨에서 반복되는 셈이다.

진짜로 필요한 건 key뿐 아니라 **fn과 options까지 한 단위로 묶는 것**이다.

<br/>

## 4️⃣ 팩토리 진화 — `queryOptions()`

먼저, queryOptions 없을 때 TS가 불편했던 부분

`queryOptions()` 도입 전에는 타입을 개발자가 직접 보장해야 했다.

**getQueryData가 unknown 반환**

```tsx
// key를 직접 넘기면
const user = queryClient.getQueryData(["user", userId]);
//    user의 타입: unknown

// 제네릭을 수동으로 붙여야 함
const user = queryClient.getQueryData<User>(["user", userId]);
//    user의 타입: User | undefined
// → 근데 이건 TypeScript가 검증하는 게 아니라 개발자가 "맞아요" 라고 선언하는 것
// → fetchUser가 AdminUser를 반환하도록 바뀌어도 여기선 에러가 안 남
```

**prefetch와 useQuery가 연결되지 않음**

```tsx
useQuery({
  queryKey: ["user", userId],
  queryFn: () => fetchUser(userId), // 반환: User
});

await queryClient.prefetchQuery({
  queryKey: ["user", userId],
  queryFn: () => fetchUser(userId), // 또 선언
  // 실수로 다른 fn을 써도 TypeScript는 두 선언이 연결됐다는 걸 모름 → 에러 없음
});
```

**커스텀 훅 우회의 한계**

```tsx
function useUserQuery(userId: string) {
  return useQuery({
    queryKey: ["user", userId],
    queryFn: () => fetchUser(userId),
  });
}
// → useQuery 밖(prefetch, getQueryData)에서는 재사용 불가
// → 훅은 컴포넌트 안에서만 쓸 수 있으니까
```

`queryOptions()`로 해결

v5에서 공식 도입된 `queryOptions()`는 key, fn, options를 하나의 단위로 묶는다.

```tsx
// queries/userQueries.ts
import { queryOptions } from "@tanstack/react-query";

export const userQueries = {
  detail: (id: string) =>
    queryOptions({
      queryKey: ["user", id],
      queryFn: () => fetchUser(id),
      staleTime: 1000 * 60 * 5,
    }),

  posts: (id: string) =>
    queryOptions({
      queryKey: ["user", id, "posts"],
      queryFn: () => fetchUserPosts(id),
    }),
};

export const postQueries = {
  detail: (id: string) =>
    queryOptions({
      queryKey: ["post", id],
      queryFn: () => fetchPost(id),
    }),

  comments: (id: string) =>
    queryOptions({
      queryKey: ["post", id, "comments"],
      queryFn: () => fetchPostComments(id),
    }),
};
```

```tsx
// 어디서든 같은 단위로 가져다 씀
const { data } = useQuery(userQueries.detail(userId));

// prefetch도 같은 단위로
await queryClient.prefetchQuery(userQueries.detail(userId));

// getQueryData도 타입 안전하게
const cached = queryClient.getQueryData(userQueries.detail(userId).queryKey);
//    cached의 타입: User | undefined  ← 자동 추론
```

fn과 options가 key와 함께 한 곳에 정의되니, `fetchUser` 시그니처가 바뀌거나 `staleTime`을 조정할 때 `userQueries.ts` 하나만 열면 된다.

타입 추론이 되는 이유 — 실제 내부 동작

런타임에 `queryOptions()`는 받은 객체를 그대로 반환한다. 그런데 TypeScript와 함께 쓸 때 큰 이점이 있다. 한 곳에서 쿼리의 모든 옵션을 정의하면 타입 추론과 타입 안전성이 전부 따라온다.

어떻게 가능한 건지, 3단계로 ㄱㄱ

<br/>

**1단계 — 문제: queryKey는 런타임에 그냥 배열이다**

`queryKey: ['user', id]`는 런타임에 `string[]`일 뿐이다. 캐시에 저장될 때 `JSON.stringify`로 해시화되면서 타입 정보는 완전히 사라진다.

그래서 `getQueryData`에 키만 넘기면 TypeScript가 "이 키로 꺼내면 `User` 나온다"는 걸 알 방법이 없다.

```tsx
queryClient.getQueryData(["user", userId]);
//                        ↑ 그냥 string[] — 어떤 타입인지 TypeScript 모름
//  반환 타입: unknown
```

<br/>

**2단계 — 해결책: queryKey에 타입 정보를 몰래 붙인다**

`queryOptions()`의 타입 시그니처를 보면 반환 타입에 이런 게 있다:

```tsx
// @tanstack/query-core/src/types.ts
export const dataTagSymbol = Symbol("dataTagSymbol");

// queryKey의 타입을 이렇게 확장함
type DataTag<TType, TValue> = TType & {
  [dataTagSymbol]: TValue; // 런타임엔 없음, 타입 레벨에만 존재
};
```

`&`(intersection)으로 기존 배열 타입을 건드리지 않으면서, 타입 레벨에만 정보를 덧붙이는 방식이다. 런타임에는 아무 변화가 없다.

`queryOptions()`를 통과하면 `queryKey`의 타입이 이렇게 바뀐다:

```tsx
// 통과 전
['user', string]

// 통과 후 — 타입 레벨에서만
['user', string] & { [dataTagSymbol]: User }
//                   ↑ "이 키로 꺼내면 User 나온다"는 정보가 붙음
```

<br/>

**3단계 — 꺼낼 때: 붙여놓은 정보를 다시 읽는다**

```tsx
// @tanstack/query-core/src/queryClient.ts
getQueryData<TTaggedQueryKey extends QueryKey>(
  queryKey: TTaggedQueryKey
): InferDataFromTag<TTaggedQueryKey> | undefined
```

`InferDataFromTag`는 `[dataTagSymbol]`에 붙어있는 타입을 꺼내서 반환 타입으로 쓴다. 제네릭 없이도 추론이 되는 이유가 이거다.

<br/>

**한 줄 요약**

`queryOptions()`는 런타임엔 아무것도 안 하고, TypeScript에게만 `queryKey`에 `"이 키 → 이 타입"` 태그를 붙여준다. `getQueryData`는 그 태그를 읽어서 반환 타입을 결정한다.

<br/>

### **🤔 최종 내 느낌**

- 팩토리 짱!
