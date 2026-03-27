# useState

---

## 목차

1. [useState 훅이란?](#1-usestate-훅이란)
2. [updater 함수를 사용하는 것이 항상 좋은지?](#2-updater-함수를-사용하는-것이-항상-좋은지)

---

## 1. useState 훅이란?

**함수형 컴포넌트에 상태(state)를 추가하는 훅**이다. 상태 값과 그것을 변경하는 setter 함수를 배열로 반환한다.

```jsx
const [state, setState] = useState(initialValue);
```

```jsx
function Counter() {
  const [count, setCount] = useState(0); // 초기값 0

  return (
    <div>
      <p>현재: {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
      <button onClick={() => setCount(0)}>초기화</button>
    </div>
  );
}
```

초기값으로 계산 비용이 큰 값을 전달할 때는 **초기화 함수**를 넘긴다. 이 함수는 첫 렌더링에만 실행된다.

```jsx
// ❌ 매 렌더링마다 createInitialState() 실행
const [state, setState] = useState(createInitialState());

// ✅ 첫 렌더링에만 실행
const [state, setState] = useState(createInitialState);
```

---

## 2. updater 함수를 사용하는 것이 항상 좋은지?

**항상 좋은 것은 아니다.** updater 함수(`setState(prev => ...)`)와 직접 값 전달(`setState(value)`) 방식의 차이를 알고 상황에 맞게 써야 한다.

### updater 함수를 써야 하는 경우 — 이전 상태에 의존할 때

```jsx
// ❌ 이전 상태에 의존하는데 직접 값 전달 — 비동기 환경에서 버그 발생 가능
setCount(count + 1);
setCount(count + 1); // count가 같은 값이라 결국 1만 증가

// ✅ updater 함수 — 항상 최신 상태를 참조
setCount(prev => prev + 1);
setCount(prev => prev + 1); // 정확히 2 증가
```

### 직접 값 전달로 충분한 경우 — 이전 상태와 무관할 때

```jsx
// ✅ 이전 상태와 관계없는 값 설정은 직접 전달이 더 단순
setIsOpen(true);
setUser(newUser);
setError(null);
```

| 방식 | 사용 시점 |
|---|---|
| `setState(value)` | 새 값이 이전 상태와 무관할 때 |
| `setState(prev => ...)` | 새 값이 이전 상태에 의존할 때, 비동기 상황, 일괄 처리 환경 |

> **정리:** `useState`는 함수형 컴포넌트에 상태를 추가한다. updater 함수는 이전 상태를 안전하게 참조해야 할 때 사용하고, 단순 값 설정은 직접 전달이 더 명확하다.

---

> 최종 수정일: 2026-03-27
