# useRef

---

## 목차

1. [useRef란?](#1-useref란)
2. [ref 콘텐츠가 재생성되는 것을 어떻게 막는지?](#2-ref-콘텐츠가-재생성되는-것을-어떻게-막는지)
3. [렌더링 메서드에서 ref에 접근하는 것이 가능한지?](#3-렌더링-메서드에서-ref에-접근하는-것이-가능한지)

---

## 1. useRef란?

**렌더링 사이에 값을 유지하지만, 값이 변경돼도 리렌더링을 유발하지 않는 훅**이다. 주로 DOM 요소에 직접 접근하거나, 렌더링과 무관한 값을 저장할 때 사용한다.

```jsx
const ref = useRef(initialValue);
// ref.current로 값에 접근/변경
```

```jsx
function FocusInput() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus(); // DOM 직접 접근
  }

  return (
    <>
      <input ref={inputRef} />
      <button onClick={handleClick}>포커스</button>
    </>
  );
}
```

---

## 2. ref 콘텐츠가 재생성되는 것을 어떻게 막는지?

`useRef`의 초기값은 첫 렌더링에만 사용되지만, 비용이 큰 객체를 인자로 직접 전달하면 매 렌더링마다 생성 후 버려진다.

```jsx
// ❌ 매 렌더링마다 new VideoPlayer()가 호출되고 버려짐
const playerRef = useRef(new VideoPlayer());

// ✅ null 체크로 최초 한 번만 생성
const playerRef = useRef(null);

if (playerRef.current === null) {
  playerRef.current = new VideoPlayer(); // 처음 렌더링 때만 생성
}
```

이 패턴은 첫 렌더링에서만 초기화가 필요한 값에 사용한다.

---

## 3. 렌더링 메서드에서 ref에 접근하는 것이 가능한지?

**렌더링 중(JSX를 반환하는 도중)에는 `ref.current`를 읽거나 쓰면 안 된다.** 리액트는 렌더링을 순수하게 유지하기를 요구하며, ref 접근은 사이드이펙트다.

```jsx
// ❌ 렌더링 중 ref 읽기 — 예측 불가능한 동작
function Component() {
  const ref = useRef(0);
  ref.current = ref.current + 1; // 렌더링 중 변경 ❌
  return <div>{ref.current}</div>;
}
```

ref는 **이벤트 핸들러나 `useEffect` 안**에서 접근해야 한다.

```jsx
function Component() {
  const countRef = useRef(0);

  function handleClick() {
    countRef.current += 1; // ✅ 이벤트 핸들러에서 접근
    console.log(countRef.current);
  }

  return <button onClick={handleClick}>클릭</button>;
}
```

단, `ref` 어트리뷰트에 ref 객체를 연결하는 것은 렌더링 중에 해도 된다. 리액트가 DOM을 커밋한 후 `ref.current`를 자동으로 설정한다.

> **정리:** `useRef`는 렌더링 없이 값을 유지한다. 비용이 큰 초기값은 null 체크로 최초 1회만 생성하고, ref 값은 렌더링 중이 아닌 이벤트 핸들러나 useEffect에서 접근한다.

---

> 최종 수정일: 2026-03-27
