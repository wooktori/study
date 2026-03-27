# useReducer

---

## 목차

1. [useReducer 훅이란? 어떻게 사용하는지?](#1-usereducer-훅이란-어떻게-사용하는지)
2. [언제 useState 대신 useReducer를 사용해야 하는지?](#2-언제-usestate-대신-usereducer를-사용해야-하는지)

---

## 1. useReducer 훅이란? 어떻게 사용하는지?

**복잡한 상태 로직을 reducer 함수로 분리해서 관리하는 훅**이다. Redux의 패턴을 컴포넌트 수준에서 사용할 수 있게 해준다.

```jsx
const [state, dispatch] = useReducer(reducer, initialState);
```

- **reducer** — `(state, action) => newState` 형태의 순수 함수
- **dispatch** — action을 reducer에 전달하는 함수
- **action** — 어떤 변화를 일으킬지 설명하는 객체 (보통 `type` 필드를 가짐)

```jsx
// 1. 초기 상태
const initialState = { count: 0 };

// 2. reducer 함수 — 상태 변환 로직을 한 곳에 모음
function reducer(state, action) {
  switch (action.type) {
    case "increment":
      return { count: state.count + 1 };
    case "decrement":
      return { count: state.count - 1 };
    case "reset":
      return { count: 0 };
    default:
      throw new Error("알 수 없는 액션: " + action.type);
  }
}

// 3. 컴포넌트에서 사용
function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <div>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+1</button>
      <button onClick={() => dispatch({ type: "decrement" })}>-1</button>
      <button onClick={() => dispatch({ type: "reset" })}>초기화</button>
    </div>
  );
}
```

---

## 2. 언제 useState 대신 useReducer를 사용해야 하는지?

| 상황 | 권장 |
|---|---|
| 단순한 단일 값 상태 | `useState` |
| 여러 상태가 서로 독립적 | `useState` |
| 여러 상태가 함께 바뀌는 로직 | `useReducer` |
| 다음 상태가 이전 상태에 크게 의존 | `useReducer` |
| 상태 변환 로직을 테스트하고 싶을 때 | `useReducer` |
| 상태 변화 추적/디버깅이 필요할 때 | `useReducer` |

```jsx
// useState로 관리하기 복잡한 예시
const [isLoading, setIsLoading] = useState(false);
const [data, setData] = useState(null);
const [error, setError] = useState(null);
// 세 상태가 항상 같이 바뀌는데 따로 관리하면 불일치가 생길 수 있음

// ✅ useReducer로 하나로 묶기
const [state, dispatch] = useReducer(reducer, {
  isLoading: false,
  data: null,
  error: null,
});
// dispatch({ type: "fetch_start" }) 하나로 세 상태를 일관성 있게 변경
```

> **정리:** `useReducer`는 상태 변환 로직을 reducer 함수로 분리해 관리한다. 단순 상태는 `useState`, 여러 상태가 연관되거나 로직이 복잡할 때는 `useReducer`가 적합하다.

---

> 최종 수정일: 2026-03-27
