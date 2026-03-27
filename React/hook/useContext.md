# useContext

---

## 목차

1. [컨텍스트란?](#1-컨텍스트란)
2. [컴포넌트 트리의 특정 부분에 대한 컨텍스트를 재정의하려면?](#2-컴포넌트-트리의-특정-부분에-대한-컨텍스트를-재정의하려면)
3. [일치하는 프로바이더가 없으면 컨텍스트 값은 어떻게 되는지?](#3-일치하는-프로바이더가-없으면-컨텍스트-값은-어떻게-되는지)

---

## 1. 컨텍스트란?

**props를 매번 전달하지 않고도 컴포넌트 트리 전체에 데이터를 공유하는 방법**이다. 테마, 로그인 정보, 언어 설정 같은 전역 데이터에 적합하다.

```jsx
// 1. 컨텍스트 생성
const ThemeContext = createContext("light"); // 기본값

// 2. 프로바이더로 값 제공
function App() {
  return (
    <ThemeContext.Provider value="dark">
      <Header />
      <Main />
    </ThemeContext.Provider>
  );
}

// 3. useContext로 값 소비
function Header() {
  const theme = useContext(ThemeContext); // "dark"
  return <header className={theme}>헤더</header>;
}
```

---

## 2. 컴포넌트 트리의 특정 부분에 대한 컨텍스트를 재정의하려면?

**중첩 프로바이더**를 사용한다. 안쪽 프로바이더의 값이 바깥 프로바이더의 값을 덮어쓴다.

```jsx
function App() {
  return (
    <ThemeContext.Provider value="light">
      {/* 대부분의 UI는 light 테마 */}
      <Header />
      <Main />

      <ThemeContext.Provider value="dark">
        {/* 이 부분만 dark 테마로 재정의 */}
        <SpecialSection />
      </ThemeContext.Provider>
    </ThemeContext.Provider>
  );
}

function SpecialSection() {
  const theme = useContext(ThemeContext); // "dark" — 가장 가까운 프로바이더 값
  return <section className={theme}>특별 섹션</section>;
}
```

컨텍스트는 **가장 가까운 상위 프로바이더**의 값을 사용한다는 원칙이 핵심이다.

---

## 3. 일치하는 프로바이더가 없으면 컨텍스트 값은 어떻게 되는지?

`createContext(defaultValue)`에 전달한 **기본값(defaultValue)** 이 사용된다.

```jsx
const UserContext = createContext({ name: "게스트", role: "visitor" });

// 프로바이더 없이 useContext를 사용하면 기본값이 반환됨
function Avatar() {
  const user = useContext(UserContext);
  return <img alt={user.name} />; // "게스트"
}
```

기본값은 테스트나 컴포넌트를 독립적으로 렌더링할 때 유용하다. 단, `Provider value={undefined}`를 넘기는 것은 프로바이더가 없는 것과 다르다 — 이 경우 `undefined`가 값으로 사용된다.

> **정리:** `useContext`는 props 전달 없이 트리 전체에 데이터를 공유한다. 중첩 프로바이더로 특정 부분만 컨텍스트 값을 재정의할 수 있고, 프로바이더가 없으면 `createContext`의 기본값이 사용된다.

---

> 최종 수정일: 2026-03-27
