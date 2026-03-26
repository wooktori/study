# isComposing으로 한글 입력 처리하기

채팅 기능에서 "Enter → 전송" UX는 매우 일반적이지만, **한글 IME가 개입되는 순간** 예상치 못한 버그가 생긴다.

---

## 초기 코드 및 문제 현상

```ts
consthandleKeyDown = (e: React.KeyboardEvent<HTMLTextAreaElement>) => {
  if (e.key === "Enter" && !e.shiftKey) {
    e.preventDefault();
    handleSubmit();
  }
};
```

- 입력: `안녕하세요`
- Enter 입력
- 실제 전송 결과
  ```
  안녕하세요
  요
  ```

마지막 글자가 분리되어 **두 번 전송**되는 현상. Windows에서는 재현 불가, **macOS에서만 발생**.

---

## 원인: IME와 조합(Composition) 이벤트

**IME(Input Method Editor)** 는 키 입력을 문자로 조합해주는 시스템이다.
한글은 자음 + 모음을 조합하는 과정이 있기 때문에 브라우저가 이를 "확정되지 않은 상태"로 관리한다.

```
ㅎ → 하 → 한 → 한ㄱ → 한글
```

> 주소창에 한글을 입력해보면 조합 중인 글자 아래에 언더라인이 생긴다.
> 이 상태에서 백스페이스를 누르면 조합 중인 요소 하나만 지워지고,
> 조합이 완료된 후 백스페이스를 누르면 글자 전체가 지워진다.

이를 위해 브라우저는 다음 이벤트를 제공한다.

| 이벤트              | 설명      |
| ------------------- | --------- |
| `compositionstart`  | 조합 시작 |
| `compositionupdate` | 조합 중   |
| `compositionend`    | 조합 완료 |

### macOS에서만 문제가 생기는 이유

핵심은 OS별 **keydown 발생 횟수**가 다르다는 것이다.

| 환경        | 이벤트 순서                                                                                   |
| ----------- | --------------------------------------------------------------------------------------------- |
| **Windows** | `compositionend` → `keydown(Enter, isComposing: false)`                                       |
| **macOS**   | `keydown(Enter, isComposing: true)` → `compositionend` → `keydown(Enter, isComposing: false)` |

Windows는 keydown이 **1번**, macOS는 **2번** 발생한다.

### 이중 전송 상세 흐름

`안녕하세요` 입력 중 Enter — `안녕하세` 는 확정, `요` 는 조합 중인 상태

```
Enter 입력
    ↓
keydown (isComposing: true)
    → isComposing 체크 없음 → handleSubmit() 실행
    → "안녕하세요" 전송, textarea 비워짐
    ↓
compositionend (data: "요")
    → IME가 "요"를 확정하려 함 → 비워진 textarea에 "요" 삽입
    ↓
keydown (isComposing: false)
    → handleSubmit() 실행
    → "요" 전송
```

결과:

```
안녕하세요   ← 첫번째 keydown에서 전송
요           ← 두번째 keydown에서 전송
```

---

## 해결: `nativeEvent.isComposing` 체크

`nativeEvent.isComposing`은 브라우저가 직접 관리하는 IME 상태값이다.
조합 중이면 `true`, 조합 완료 후엔 `false`로 바뀌므로 이를 체크해 차단한다.

```tsx
const handleKeyDown = (e: React.KeyboardEvent<HTMLTextAreaElement>) => {
  if (e.key !== "Enter" || e.shiftKey) return;
  if (e.nativeEvent.isComposing) return; // 조합 중이면 차단

  e.preventDefault();
  handleSubmit();
};
```

정상 흐름:

```
① keydown (isComposing: true)  → 차단 ✓
② compositionend               → "요" 삽입
③ keydown (isComposing: false) → handleSubmit() 실행 ✓
```

> `e.isComposing`이 아닌 `e.nativeEvent.isComposing`을 써야 한다.
> React의 합성 이벤트(SyntheticEvent)에는 `isComposing`이 없기 때문이다.

---

## 정리

| 시도                           | 결과      | 이유                           |
| ------------------------------ | --------- | ------------------------------ |
| `isComposing` 체크 없음        | 이중 전송 | macOS에서 keydown이 2번 발생   |
| `nativeEvent.isComposing` 체크 | 정상 동작 | 조합 중엔 차단, 완료 후엔 전송 |

> macOS에서 keydown이 2번 발생하는 것이 근본 원인이며,
> `nativeEvent.isComposing`으로 조합 상태를 확인해 첫 번째 keydown을 차단한다.

_마지막 업데이트: 2026.03.26_
