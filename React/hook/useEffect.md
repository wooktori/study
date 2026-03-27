# useEffect와 useLayoutEffect

---

## 목차

1. [useEffect 훅이란?](#1-useeffect-훅이란)
2. [반응형 의존성이 로직에 어떤 영향을 미치는지?](#2-반응형-의존성이-로직에-어떤-영향을-미치는지)
3. [설정 및 정리 함수는 얼마나 자주 호출되는지?](#3-설정-및-정리-함수는-얼마나-자주-호출되는지)
4. [객체 또는 함수를 의존성에서 제거해야 하는 시점은 언제인지?](#4-객체-또는-함수를-의존성에서-제거해야-하는-시점은-언제인지)
5. [useLayoutEffect란? 어떻게 동작하는지?](#5-uselayouteffect란-어떻게-동작하는지)

---

## 1. useEffect 훅이란?

**컴포넌트가 렌더링된 후 사이드이펙트를 실행하는 훅**이다. 데이터 패칭, 구독, DOM 조작, 타이머 설정 등에 사용한다.

```jsx
useEffect(() => {
  // 설정(setup) 함수 — 이펙트 로직
  const subscription = subscribe(userId);

  return () => {
    // 정리(cleanup) 함수 — 이펙트 해제
    subscription.unsubscribe();
  };
}, [userId]); // 의존성 배열
```

---

## 2. 반응형 의존성이 로직에 어떤 영향을 미치는지?

의존성 배열에 포함된 값이 변경될 때마다 이펙트가 재실행된다. 의존성 배열의 구성에 따라 동작이 달라진다.

```jsx
// 1. 의존성 없음 — 마운트/언마운트 시에만 실행
useEffect(() => {
  console.log("마운트");
  return () => console.log("언마운트");
}, []);

// 2. 의존성 있음 — userId가 바뀔 때마다 재실행
useEffect(() => {
  fetchUser(userId);
}, [userId]);

// 3. 의존성 배열 생략 — 매 렌더링마다 실행 (주의: 대부분 불필요)
useEffect(() => {
  console.log("렌더링마다 실행");
});
```

**반응형 값이란?** 컴포넌트 내부에서 선언된 props, state, 그리고 그것들로부터 계산된 변수는 모두 반응형 값이다. 이펙트 내에서 사용하는 반응형 값은 의존성 배열에 포함해야 한다. `eslint-plugin-react-hooks`의 `exhaustive-deps` 규칙이 이를 자동으로 감지해준다.

---

## 3. 설정 및 정리 함수는 얼마나 자주 호출되는지?

| 시점 | 설정 함수 | 정리 함수 |
|---|---|---|
| 컴포넌트 마운트 | ✅ 실행 | ❌ |
| 의존성 변경 전 | ❌ | ✅ 실행 (이전 이펙트 정리) |
| 의존성 변경 후 | ✅ 실행 | ❌ |
| 컴포넌트 언마운트 | ❌ | ✅ 실행 |

```
마운트         → setup 실행
userId 변경    → cleanup 실행 → setup 재실행
userId 변경    → cleanup 실행 → setup 재실행
언마운트       → cleanup 실행
```

**개발 환경(Strict Mode)** 에서는 리액트가 마운트 → 언마운트 → 마운트를 한 번 더 수행해 정리 함수가 제대로 작동하는지 검증한다. 이 때문에 개발 중에는 이펙트가 두 번 실행되는 것처럼 보인다.

---

## 4. 객체 또는 함수를 의존성에서 제거해야 하는 시점은 언제인지?

객체와 함수는 렌더링마다 새로 생성되므로, 의존성에 포함하면 이펙트가 매 렌더링마다 재실행된다.

```jsx
// ❌ 문제 — options 객체가 렌더링마다 새로 생성됨
function Component({ userId }) {
  const options = { id: userId, type: "user" }; // 매 렌더링마다 새 객체

  useEffect(() => {
    fetchData(options);
  }, [options]); // 매 렌더링마다 실행
}
```

### 해결 방법 1: 객체를 이펙트 안으로 이동

```jsx
useEffect(() => {
  const options = { id: userId, type: "user" }; // 이펙트 내부에서 생성
  fetchData(options);
}, [userId]); // 원시 값만 의존성으로
```

### 해결 방법 2: useMemo로 객체 메모이제이션

```jsx
const options = useMemo(() => ({ id: userId, type: "user" }), [userId]);

useEffect(() => {
  fetchData(options);
}, [options]); // options가 실제로 바뀔 때만 재실행
```

### 해결 방법 3: 함수는 useCallback으로 메모이제이션

```jsx
const fetchUser = useCallback(() => {
  fetch(`/api/users/${userId}`);
}, [userId]);

useEffect(() => {
  fetchUser();
}, [fetchUser]);
```

> **정리:** `useEffect`는 렌더링 후 사이드이펙트를 실행한다. 의존성이 바뀔 때마다 정리 후 재실행된다. 객체와 함수는 이펙트 내부로 이동하거나 `useMemo`/`useCallback`으로 메모이제이션해 불필요한 재실행을 막는다.

---

## 5. useLayoutEffect란? 어떻게 동작하는지?

**DOM 업데이트 직후, 브라우저가 화면을 그리기 전에 동기적으로 실행되는 훅**이다. API는 `useEffect`와 동일하다.

```jsx
useLayoutEffect(() => {
  // DOM이 업데이트된 직후, 페인트 전에 실행
  const rect = element.current.getBoundingClientRect();
  setPosition(rect);
}, []);
```

### useEffect vs useLayoutEffect

```
렌더링 → DOM 업데이트 → useLayoutEffect 실행 → 브라우저 페인트 → useEffect 실행
```

| 구분 | useEffect | useLayoutEffect |
|---|---|---|
| 실행 시점 | 브라우저 페인트 후 (비동기) | 브라우저 페인트 전 (동기) |
| 성능 | 페인트를 블로킹하지 않음 | 페인트를 블로킹할 수 있음 |
| 용도 | 대부분의 사이드이펙트 | DOM 측정, 레이아웃 기반 계산 |

**useLayoutEffect가 필요한 경우** — DOM 크기/위치를 측정해 상태를 업데이트해야 할 때. `useEffect`를 사용하면 페인트가 먼저 일어나 화면이 잠깐 깜빡일 수 있다.

```jsx
function Tooltip({ text, targetRef }) {
  const tooltipRef = useRef(null);
  const [position, setPosition] = useState({ top: 0, left: 0 });

  useLayoutEffect(() => {
    // 페인트 전에 위치를 계산해야 깜빡임이 없음
    const rect = targetRef.current.getBoundingClientRect();
    setPosition({ top: rect.bottom, left: rect.left });
  }, []);

  return (
    <div ref={tooltipRef} style={position}>
      {text}
    </div>
  );
}
```

> **정리:** `useLayoutEffect`는 DOM 업데이트 후 페인트 전에 동기로 실행된다. DOM 측정처럼 페인트 전에 처리해야 하는 작업에만 사용하고, 그 외에는 `useEffect`를 쓴다.

---

> 최종 수정일: 2026-03-27
