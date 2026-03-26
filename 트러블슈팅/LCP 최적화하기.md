# LCP 최적화 트러블슈팅

## LCP란?

**LCP(Largest Contentful Paint)** 는 뷰포트 내에서 가장 큰 콘텐츠 요소가 화면에 렌더링되는 시점을 측정하는 Core Web Vitals 지표다.

> 단순히 "큰 요소"가 아니라, **사용자가 페이지의 핵심 콘텐츠를 인식하는 시점**을 나타낸다.

| 점수      | 기준          |
| --------- | ------------- |
| 좋음      | 2.5초 이하    |
| 개선 필요 | 2.5초 ~ 4.0초 |
| 나쁨      | 4.0초 초과    |

LCP로 측정되는 요소는 주로 다음과 같다.

- `<img>` 태그
- `<image>` 태그 (SVG 내부)
- CSS `background-image`가 적용된 요소
- 텍스트 블록 (`<p>`, `<h1>` 등)

---

## 문제 현상

Lighthouse 측정 결과 LCP **3.7초** — "개선 필요" 구간.

LCP 요소를 확인해보니 **무한스크롤 완료 indicator** (_모든 팔로잉을 불러왔습니다._)가 LCP로 인식되고 있었다.
<img width="600" height="400" alt="image" src="https://github.com/user-attachments/assets/bcb54980-22bc-400c-b4ed-b4c71fe7e712" />

### 왜 indicator가 LCP로 잡혔나

해당 페이지는 클라이언트 컴포넌트에서 데이터를 패칭하는 구조였다.

```
초기 HTML 전달 (서버)
    → 의미 있는 콘텐츠 없음 (빈 껍데기)
    → 클라이언트에서 API 호출
    → 데이터 수신 후 렌더링
    → 무한스크롤 전체 완료 시 indicator 노출
    → 브라우저: "이게 제일 큰 요소네" → LCP 측정
```

브라우저는 페이지에서 **가장 늦게, 가장 크게** 렌더링된 요소를 LCP로 선택했고, 그게 바로 indicator 텍스트였다.

---

## 시도 1: indicator 제거 → LCP 개선 (잘못된 접근)

indicator를 제거하자 LCP가 프로필 이미지로 변경되며 **0.6초**로 개선됐다.

하지만 이는 **수치만을 위한 잘못된 접근**이다.

무한스크롤 완료 시 사용자에게 "더 이상 콘텐츠가 없다"는 indicator는 필요한 UX 요소다. 제거가 아니라 **indicator가 LCP로 잡히는 구조 자체를 개선**해야 했다.

<img width="600" height="400" alt="image" src="https://github.com/user-attachments/assets/023c3120-cc52-4551-84f7-7b2866da6abd" />

---

## 시도 2: indicator 텍스트 크기 축소 → LCP 개선 (잘못된 접근)

텍스트 크기를 줄이면 브라우저가 더 이상 indicator를 "가장 큰 요소"로 인식하지 않아 LCP가 이미지로 이동했다.

수치는 개선됐지만 이 역시 **근본 원인을 해결하지 않은 우회책**이다.

핵심 문제는 indicator의 크기가 아니라, **초기 렌더링 시점에 의미 있는 콘텐츠가 없다는 것**이었다.
<img width="600" height="400" alt="image" src="https://github.com/user-attachments/assets/81d7386a-2099-4400-8c51-5d89e64e3be8" />

---

## 근본 원인 분석

LCP가 늦은 이유는 indicator 자체가 아니라 **렌더링 구조** 때문이었다.

```
문제: 클라이언트에서 데이터 패칭
    → 서버에서 보내는 HTML에 콘텐츠 없음
    → 브라우저가 API 응답을 기다린 후 렌더링
    → 핵심 콘텐츠(팔로우 유저 목록)가 늦게 노출
    → 가장 늦게 나타난 indicator가 LCP로 인식
```

해당 페이지에서 진짜 핵심 콘텐츠는 **팔로우 유저 목록**이다.
핵심 콘텐츠가 초기 렌더링 단계에서 바로 노출되도록 구조를 바꿔야 했다.

---

## 최종 해결: React Query prefetch + hydration

서버 컴포넌트에서 React Query prefetch를 적용해 **핵심 콘텐츠가 초기 HTML에 포함**되도록 개선했다.
<img width="600" height="400" alt="image" src="https://github.com/user-attachments/assets/dcc75ca6-5143-4d49-80eb-7a04ed90e8df" />

```
개선 후: 서버에서 데이터 prefetch
    → 초기 HTML에 팔로우 유저 목록 포함
    → 브라우저가 HTML을 파싱하자마자 콘텐츠 렌더링
    → 팔로우 유저 목록이 LCP로 인식
    → indicator는 여전히 존재하지만 LCP에 영향 없음
```

```tsx
// 서버 컴포넌트
export default async function Page() {
  const queryClient = new QueryClient();

  await queryClient.prefetchInfiniteQuery({
    queryKey: ["followings"],
    queryFn: fetchFollowings,
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <FollowingList />
    </HydrationBoundary>
  );
}
```

---

## 결과

| 구분           | LCP   | 비고                              |
| -------------- | ----- | --------------------------------- |
| 개선 전        | 3.7초 | indicator가 LCP                   |
| indicator 제거 | 0.6초 | UX 훼손                           |
| prefetch 적용  | 1.4초 | 핵심 콘텐츠가 LCP, indicator 유지 |

**3.7초 → 1.4초, 약 62% 개선**

수치는 같지만 의미가 다르다. indicator를 제거한 게 아니라 **핵심 콘텐츠를 더 빨리 보여주는 구조로 개선**한 결과다.

---

## 부록: Next.js Image preload

LCP 개선 과정에서 `<Image preload>` 옵션도 검토했다.

```tsx
<Image src={profileImage} priority /> // preload와 동일한 효과
```

| 상황                               | 판단                             |
| ---------------------------------- | -------------------------------- |
| 대표 이미지 1개 (히어로 이미지 등) | preload 적합                     |
| 리스트 형태 (유저 목록 등)         | preload 사용 시 오히려 성능 저하 |

리스트의 모든 이미지에 preload를 적용하면 불필요한 네트워크 요청이 폭증해 초기 로딩이 느려진다. 리스트에서는 적용하지 않는 것이 맞다.

---

## 교훈

1. **LCP 수치만 보지 말고 LCP 요소를 확인하자** — 수치가 나빠도 원인이 다르면 해결책도 다르다.
2. **우회책과 근본 해결을 구분하라** — 요소 제거나 크기 축소는 지표를 속이는 것 - 절대 하면 안되는 일시적인 해결책.
3. **LCP 최적화의 핵심은 핵심 콘텐츠를 빨리 보여주는 것** — 서버 렌더링, prefetch, 캐싱 전략이 실질적인 해결책.
