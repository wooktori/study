# JavaScript this

> 전역 객체, this 바인딩 규칙, 화살표 함수, 클래스, 콜백에서의 this 핵심 Q&A 정리

---

## 목차

1. [브라우저와 Node.js의 전역 객체가 다른 이유](#1-브라우저와-nodejs의-전역-객체가-다른-이유)
2. [this가 컨텍스트마다 달라지도록 설계한 이유와 화살표 함수의 this가 렉시컬 스코프로 고정된 이유](#2-this가-컨텍스트마다-달라지도록-설계한-이유와-화살표-함수의-this가-렉시컬-스코프로-고정된-이유)
3. [클래스와 객체에서 this의 차이점](#3-클래스와-객체에서-this의-차이점)
4. [특정 클래스에 다른 클래스의 메서드가 사용될 때 this 바인딩](#4-특정-클래스에-다른-클래스의-메서드가-사용될-때-this-바인딩)
5. [bind 없이 this를 고정하는 방법](#5-bind-없이-this를-고정하는-방법)
6. [bind 후 call/apply로 this를 변경할 수 없는 이유](#6-bind-후-callapply로-this를-변경할-수-없는-이유)
7. [콜백과 이벤트 핸들러에서 this 동작과 주의점](#7-콜백과-이벤트-핸들러에서-this-동작과-주의점)

---

## 1. 브라우저와 Node.js의 전역 객체가 다른 이유

### 전역 객체란?

자바스크립트 코드가 실행될 때 가장 바깥에 존재하는 객체다. 어디서든 접근할 수 있고, 전역 변수와 함수가 이 객체의 프로퍼티로 등록된다.

```js
// 브라우저
console.log(this); // window
console.log(globalThis); // window

// Node.js
console.log(this); // {} (모듈 스코프에서는 빈 객체)
console.log(globalThis); // global
```

### 왜 서로 다른가?

자바스크립트는 **언어 명세(ECMAScript)** 와 **실행 환경(런타임)** 이 분리되어 있다. ECMAScript는 "전역 객체가 있어야 한다"고만 정의하고, 그 구체적인 구현은 각 환경에 맡긴다.

| 환경       | 전역 객체 | 제공하는 것                       |
| ---------- | --------- | --------------------------------- |
| 브라우저   | `window`  | DOM, BOM, `document`, `location`  |
| Node.js    | `global`  | `process`, `__dirname`, `require` |
| Web Worker | `self`    | 브라우저 API 일부 (DOM 접근 불가) |

### 환경마다 필요한 것이 다르다

```
브라우저: 화면을 그려야 함 → window.document, window.alert 필요
Node.js:  파일/네트워크를 다뤄야 함 → global.process, global.Buffer 필요
```

이 차이 때문에 하나의 전역 객체로 통일할 수 없다. 각 환경이 자신에게 필요한 API를 전역 객체에 붙여 제공한다.

### globalThis로 통일

ES2020부터 **어느 환경에서든 전역 객체를 가리키는** `globalThis`가 추가됐다.

```js
// 환경에 상관없이 동작
console.log(globalThis === window); // 브라우저: true
console.log(globalThis === global); // Node.js: true
```

> **정리:** 전역 객체가 다른 이유는 자바스크립트 명세가 구현을 환경에 위임하기 때문이다. 브라우저와 Node.js는 목적이 다르므로 각자 필요한 API를 전역 객체로 제공한다.

---

## 2. this가 컨텍스트마다 달라지도록 설계한 이유와 화살표 함수의 this가 렉시컬 스코프로 고정된 이유

### 왜 this가 컨텍스트마다 달라지도록 설계했나?

자바스크립트는 **하나의 함수를 여러 객체에서 재사용**할 수 있도록 설계됐다. `this`가 고정되면 함수를 다른 객체에 붙여 쓸 수 없다.

```js
function introduce() {
  console.log(`나는 ${this.name}야`);
}

const user1 = { name: "철수", introduce };
const user2 = { name: "영희", introduce };

user1.introduce(); // "나는 철수야"
user2.introduce(); // "나는 영희야"
// 같은 함수를 다른 객체에서 재사용 가능
```

`this`가 호출 시점에 결정되기 때문에 가능한 패턴이다. 만약 `this`가 선언 시점에 고정됐다면 각 객체마다 함수를 새로 만들어야 한다.

### this 바인딩 규칙 요약

| 호출 방식                        | this                 |
| -------------------------------- | -------------------- |
| `obj.method()`                   | `obj`                |
| `fn()`                           | 전역 (`window`)      |
| `new Fn()`                       | 새로 생성된 인스턴스 |
| `fn.call(ctx)` / `fn.apply(ctx)` | `ctx`                |
| `fn.bind(ctx)()`                 | `ctx` (고정)         |
| 이벤트 핸들러 `function`         | 이벤트 발생 요소     |
| 화살표 함수                      | 상위 스코프의 `this` |

### 그런데 왜 화살표 함수는 렉시컬 스코프로 고정했나?

동적 `this`는 재사용성을 높이지만, **콜백 내부에서 외부 객체를 참조하려 할 때 불편**하다는 문제가 있다.

```js
// ES5 시절의 문제
function Timer() {
  this.count = 0;
  setInterval(function () {
    this.count++; // ❌ this가 window → count는 Timer 인스턴스의 것이 아님
  }, 1000);
}

// 당시 해결책 — this를 변수에 저장
function Timer() {
  const self = this; // this를 캡처
  this.count = 0;
  setInterval(function () {
    self.count++; // ✅ 클로저로 외부 this 참조
  }, 1000);
}
```

ES6에서 화살표 함수가 도입되면서 이 패턴을 언어 차원에서 해결했다.

```js
// 화살표 함수로 해결
function Timer() {
  this.count = 0;
  setInterval(() => {
    this.count++; // ✅ 선언된 위치(Timer 함수)의 this를 사용
  }, 1000);
}
```

### 두 설계의 역할 분담

| 함수 종류   | this 결정 시점 | 주 용도                               |
| ----------- | -------------- | ------------------------------------- |
| 일반 함수   | 호출할 때      | 메서드, 생성자 — 객체를 주체로 동작   |
| 화살표 함수 | 선언할 때      | 콜백 — 외부 컨텍스트를 유지해야 할 때 |

> **정리:** 동적 `this`는 함수 재사용성을 위해 설계됐고, 화살표 함수의 렉시컬 `this`는 콜백에서 외부 컨텍스트를 잃는 문제를 해결하기 위해 도입됐다. 두 방식은 서로 보완 관계다.

---

## 3. 클래스와 객체에서 this의 차이점

### 객체 리터럴에서의 this

```js
const person = {
  name: "철수",
  greet() {
    console.log(this.name); // 호출한 객체(person)
  },
};

person.greet(); // "철수"

const fn = person.greet;
fn(); // undefined — 호출 주체가 없으면 전역
```

객체 리터럴의 메서드에서 `this`는 **호출 방식**에 따라 결정된다. 객체에 묶여있지 않다.

### 클래스에서의 this

```js
class Person {
  constructor(name) {
    this.name = name;
  }
  greet() {
    console.log(this.name);
  }
}

const p = new Person("영희");
p.greet(); // "영희"

const fn = p.greet;
fn(); // ❌ TypeError (strict mode) — 클래스는 항상 strict mode
```

클래스 내부는 **자동으로 strict mode**가 적용된다. 그래서 호출 주체 없이 호출하면 `this`가 전역이 아닌 `undefined`가 된다.

### 핵심 차이

| 구분           | 객체 리터럴             | 클래스                    |
| -------------- | ----------------------- | ------------------------- |
| strict mode    | ❌ 기본 아님            | ✅ 자동 적용              |
| 주체 없는 호출 | `this = window`         | `this = undefined` (에러) |
| 인스턴스 생성  | 직접 작성               | `new` 키워드 사용         |
| 상속           | 수동 (Object.create 등) | `extends`로 명확하게      |

> **정리:** 클래스는 strict mode가 자동 적용되어 `this`가 더 엄격하게 동작한다. 메서드를 분리해서 호출하면 에러가 발생하므로 `bind`나 화살표 함수를 사용해야 한다.

---

## 4. 특정 클래스에 다른 클래스의 메서드가 사용될 때 this 바인딩

### 다른 클래스의 메서드를 빌려 쓸 때

메서드를 빌려와 호출하면, `this`는 **실제 호출 시점에 앞에 있는 객체**로 결정된다.

```js
class Animal {
  constructor(name) {
    this.name = name;
  }
  speak() {
    console.log(`${this.name}이 소리를 냅니다`);
  }
}

class Robot {
  constructor(name) {
    this.name = name;
  }
}

const dog = new Animal("강아지");
const robot = new Robot("로봇");

// call로 this를 robot으로 지정
dog.speak.call(robot); // "로봇이 소리를 냅니다" — this = robot
```

### 메서드를 프로퍼티로 직접 할당할 때

```js
class Logger {
  constructor(prefix) {
    this.prefix = prefix;
  }
  log(message) {
    console.log(`[${this.prefix}] ${message}`);
  }
}

class App {
  constructor() {
    this.prefix = "APP";
    const logger = new Logger("LOGGER");
    this.log = logger.log; // 메서드를 그대로 할당
  }
}

const app = new App();
app.log("시작"); // "[APP] 시작" — this = app (호출 주체)
```

메서드가 어느 클래스에서 왔는지는 관계없다. **누가 점(.)으로 호출하느냐**가 `this`를 결정한다.

### call / apply / bind로 명시적 지정

```js
class Validator {
  constructor(min, max) {
    this.min = min;
    this.max = max;
  }
  check(value) {
    return value >= this.min && value <= this.max;
  }
}

class Form {
  constructor() {
    this.min = 1;
    this.max = 100;
  }
}

const validator = new Validator(0, 50);
const form = new Form();

// form의 min/max를 기준으로 validator의 check 실행
validator.check.call(form, 30); // true (1 <= 30 <= 50)
validator.check.call(form, 80); // false (80 > 50이므로 false)
```

### 정리

| 사용 방식                         | this       |
| --------------------------------- | ---------- |
| `a.method()`                      | `a`        |
| `b.method = a.method; b.method()` | `b`        |
| `a.method.call(b)`                | `b`        |
| `a.method.bind(b)()`              | `b` (고정) |

> **정리:** 메서드는 어느 클래스에서 정의됐는지와 무관하게, 호출할 때 앞에 오는 객체가 `this`가 된다. `call`, `apply`, `bind`로 명시적으로 지정할 수도 있다.

---

## 5. bind 없이 this를 고정하는 방법

### 방법 1 — 화살표 함수로 메서드 정의

클래스 필드 문법으로 화살표 함수를 메서드로 정의하면, 인스턴스 생성 시 `this`가 고정된다.

```js
class Button {
  constructor(label) {
    this.label = label;
  }

  // 화살표 함수로 정의 — this가 인스턴스로 고정
  handleClick = () => {
    console.log(this.label);
  };
}

const btn = new Button("확인");
document.addEventListener("click", btn.handleClick); // ✅ "확인"
```

일반 메서드와의 차이:

```js
class Button {
  handleClick() {
    /* 일반 메서드 — prototype에 저장 */
  }
  handleClick = () => {
    /* 화살표 — 인스턴스마다 별도 함수 생성 */
  };
}
```

### 방법 2 — 생성자에서 클로저로 고정

```js
class Counter {
  constructor() {
    const self = this;
    this.count = 0;

    this.increment = function () {
      self.count++; // 클로저로 self 캡처
      console.log(self.count);
    };
  }
}

const counter = new Counter();
const fn = counter.increment;
fn(); // 1 — this가 아닌 self(클로저)를 사용
```

### 방법 3 — 화살표 함수로 감싸기

```js
class Timer {
  constructor() {
    this.count = 0;
  }
  start() {
    setInterval(() => this.tick(), 1000); // 화살표로 감싸서 호출
  }
  tick() {
    this.count++;
    console.log(this.count);
  }
}
```

### 방법 비교

| 방법                 | 특징                          | 단점                            |
| -------------------- | ----------------------------- | ------------------------------- |
| 화살표 함수 필드     | 선언과 동시에 고정, 가장 간결 | 인스턴스마다 함수 생성 (메모리) |
| 생성자에서 클로저    | 명시적, ES5 호환              | 코드 장황                       |
| 화살표 함수로 감싸기 | 기존 메서드 유지, 유연        | 호출마다 래퍼 함수 생성         |

> **원칙:** 이벤트 핸들러나 콜백으로 넘길 메서드는 **클래스 필드 화살표 함수**로 정의하는 것이 가장 간결하다.

---

## 6. bind 후 call/apply로 this를 변경할 수 없는 이유

### 현상

```js
function greet() {
  console.log(this.name);
}

const user = { name: "철수" };
const admin = { name: "관리자" };

const boundGreet = greet.bind(user); // this = user로 고정

boundGreet(); // "철수"
boundGreet.call(admin); // "철수" — call로 바꿔도 여전히 철수
boundGreet.apply(admin); // "철수" — apply도 마찬가지
```

### 이유: bind는 새 함수를 만들면서 this를 내부적으로 잠근다

`bind`는 `this`를 영구 고정한 **새 함수**를 반환한다. 내부적으로 아래처럼 동작한다.

```js
// bind의 동작을 단순화하면
Function.prototype.myBind = function (ctx) {
  const fn = this;
  return function (...args) {
    return fn.call(ctx, ...args); // ctx가 클로저로 캡처되어 항상 사용됨
  };
};
```

반환된 함수 안에서 `fn.call(ctx, ...)`를 호출하기 때문에, 외부에서 `call`/`apply`로 `this`를 바꿔봐도 **이미 내부에서 고정된 `ctx`로 실행**된다.

```
boundGreet.call(admin)
  → 내부에서 greet.call(user) 실행  ← user가 클로저로 고정
  → admin은 무시됨
```

### call / apply / bind 비교

```js
greet.call(user, arg1, arg2); // 즉시 호출, 인수 나열
greet.apply(user, [arg1, arg2]); // 즉시 호출, 인수 배열
greet.bind(user)(arg1, arg2); // 새 함수 반환, 나중에 호출
```

| 메서드                   | 즉시 실행      | this 변경 가능 여부 |
| ------------------------ | -------------- | ------------------- |
| `call`                   | ✅             | ✅ (bind 전에만)    |
| `apply`                  | ✅             | ✅ (bind 전에만)    |
| `bind`                   | ❌ (함수 반환) | ✅ (한 번만)        |
| bind된 함수에 call/apply | ✅             | ❌ 무시됨           |

> **정리:** `bind`는 `this`를 클로저로 캡처한 새 함수를 만든다. 외부에서 `call`/`apply`로 `this`를 지정해도 내부에서 이미 고정된 값을 사용하므로 변경되지 않는다.

---

## 7. 콜백과 이벤트 핸들러에서 this 동작과 주의점

### 콜백 함수에서 this

콜백은 함수를 **다른 함수에 전달해 나중에 호출**하는 방식이다. 호출하는 쪽이 `this`를 결정한다.

```js
const user = {
  name: "철수",
  greet() {
    console.log(this.name);
  },
};

// ❌ 메서드를 꺼내서 전달 — user와 연결 끊김
setTimeout(user.greet, 1000); // undefined

// setTimeout 내부: callback() 으로 그냥 호출 → this = 전역
```

`forEach`, `map`, `setTimeout` 등 대부분의 콜백을 받는 함수는 콜백을 **주체 없이 그냥 호출**한다.

```js
// forEach 내부가 대략 이런 구조
function forEach(callback) {
  for (let i = 0; i < this.length; i++) {
    callback(this[i]); // 그냥 호출 → this = 전역
  }
}
```

### 콜백에서 this를 유지하는 방법

```js
const user = {
  name: "철수",
  greet() {
    console.log(this.name);
  },
};

// 방법 1 — 화살표 함수로 감싸기
setTimeout(() => user.greet(), 1000); // ✅

// 방법 2 — bind
setTimeout(user.greet.bind(user), 1000); // ✅
```

### 이벤트 핸들러에서의 this

일반 함수를 이벤트 핸들러로 등록하면 `this`는 **이벤트가 발생한 DOM 요소**다.

```js
const button = document.querySelector("button");

button.addEventListener("click", function () {
  console.log(this); // button 요소
  console.log(this === button); // true
});
```

화살표 함수를 쓰면 `this`는 **상위 스코프**가 된다.

```js
button.addEventListener("click", () => {
  console.log(this); // window (전역) — 화살표 함수는 this가 없음
  console.log(this === button); // false
});
```

### 클래스 메서드를 이벤트 핸들러로 넘길 때 주의점

```js
class Form {
  constructor() {
    this.value = "입력값";

    // ❌ this가 끊김
    button.addEventListener("click", this.handleSubmit);

    // ✅ bind로 고정
    button.addEventListener("click", this.handleSubmit.bind(this));

    // ✅ 화살표 함수로 감싸기
    button.addEventListener("click", () => this.handleSubmit());
  }

  handleSubmit() {
    console.log(this.value); // this가 Form 인스턴스여야 함
  }
}
```

### 상황별 this 정리

| 상황                                            | this                 |
| ----------------------------------------------- | -------------------- |
| `obj.method()` 직접 호출                        | `obj`                |
| 콜백으로 전달 후 호출 (`forEach`, `setTimeout`) | 전역 (`window`)      |
| `addEventListener` + 일반 함수                  | 이벤트 발생 요소     |
| `addEventListener` + 화살표 함수                | 상위 스코프의 `this` |
| `addEventListener` + bind된 함수                | bind된 객체          |

### DOM 요소 vs 인스턴스 선택 기준

```js
// DOM 요소가 필요할 때 — 일반 함수
button.addEventListener("click", function () {
  this.classList.toggle("active"); // this = button
});

// 클래스 인스턴스가 필요할 때 — 화살표 함수 또는 bind
button.addEventListener("click", () => {
  this.handleClick(); // this = 클래스 인스턴스
});
```

> **원칙:** 이벤트 핸들러에서 DOM 요소를 `this`로 쓰려면 일반 함수, 클래스 인스턴스를 유지하려면 화살표 함수나 `bind`를 사용한다.

---

_마지막 업데이트: 2026.03.14_
