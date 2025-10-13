# 서론

피드줍줍 서비스는 무한스크롤 방식으로 새로운 피드백을 불러와 화면에 보여준다. 그런데 어느 날 200개의 피드백을 스크롤하여 표시했을 때는 문제가 없었지만, 그 이상을 가져오자 화면이 버벅거리고 모달을 열 때도 한 박자 늦게 반응하는 현상이 발생했다. 이번 글에서는 이러한 성능 문제를 어떻게 발견했고, 어떤 과정을 거쳐 해결했는지 정리해 보려고 한다.

# 현재 구조

우선 문제의 원인을 찾기 전, 문제가 발생하는 페이지인 `UserDashboard`페이지의 구조를 살펴보면 아래와 같다.

<img width="1490" height="1382" alt="image" src="https://github.com/user-attachments/assets/f08949e8-df7c-44fa-a8ba-b3bafe613a29" />


이를 트리 구조로 바꿔서 보면 아래와 같다.

<img width="2534" height="906" alt="image" src="https://github.com/user-attachments/assets/e2b88bab-df04-4c3f-9867-04d9dd81c97c" />


# 문제 찾기

우선 문제의 원인을 찾기 위해서, 브라우저 메인 쓰레드의 과부화 상태를 확인하기 위해서 무한스크롤을 할 때, 브라우저 메인 쓰레드에 어떤 일이 일어나는 지 측정해봤다.

그 결과 아래와 같이 나왔다.

<img width="2608" height="932" alt="image" src="https://github.com/user-attachments/assets/07d2f0dc-7ac5-47e3-ad0b-6d11119e6f41" />


결과를 살펴 보니, 메인 쓰레드의 일이 너무 많았고, 그 결과 16ms이내 한 프레임을 그리지 못하는 탓에 렉 걸리고 늦게 반응하는 현상이 발생했다. 함수 호출을 살펴보니, 새로운 피드백을 받아올 때 매번 기존 피드백들도 새롭게 리렌더링이 발생하고 있었다. 이는 리액트 devtools를 확인해봐도 알 수 있다. `Highlight updates when components render` 설정 옵션을 켜고 무한스크롤 시 렌더링을 측정해보았다.

[화면 기록 2025-09-20 20.10.58.mov](attachment:8efc4dee-ae48-461c-9763-4964173d2eda:화면_기록_2025-09-20_20.10.58.mov)

그 결과 매번 새로운 피드백을 받아올 때, 모든 컴포넌트가 새롭게 렌더링되는 것을 확인할 수 있었다. **Profiler**로 확인해보면 더 명확하다.

<img width="3490" height="1006" alt="image" src="https://github.com/user-attachments/assets/d3ef5676-81a3-4323-ba5a-477966d93dac" />


새로 피드백을 받아올 때, `UserDashboard` 페이지 컴포넌트 전체가 렌더링되는 것을 확인할 수 있다.

### 이게 왜 문제가 될까?

우선 브라우저 메인 쓰레드의 1프레임을 그림으로 알아보면 아래와 같다.

<img width="2060" height="788" alt="image" src="https://github.com/user-attachments/assets/6c4ee54b-ac67-4d0f-8996-c2bd12ae04f9" />


메인 쓰레드는 저 하나의 사이클(1프레임)을 16ms 이내에 완료해야지 사용자에게 부드러운 화면을 보여줄 수 있다. 만약 하나의 사이클(1프레임)이 16ms 가 넘어가게 되면 내가 겪은 문제인 사용자 이벤트 처리 지연과 프레임 드랍과 같은 문제가 발생한다. 지금 내 코드에서는 새로운 데이터를 불러올 때마다, 기존의 데이터를 사용하던 UserFeedback 전체가 렌더링되고 있다. 즉 10개의 새로운 데이터만을 추가적으로 들고왔지만 기존 200개의 데이터를 사용하는 컴포넌트도 다시 계산되고 있다는 뜻이다. 이렇게 되면 메인 쓰레드는 불필요한 계산 때문에 리액트 렌더링에 많은 시간을 사용하게 된다. 그렇기 때문에 이후 작업들이 딜레이 되면서 문제가 되었던 것이다.

<aside>

왜 16.67ms일까?

일반적으로 모니터의 주사율은 60Hz, 즉 1초에 60번 화면을 갱신한다. 이는 곧 1초(1000ms)를 60으로 나눈 값, 약 **16.67ms마다 한 번씩 새로운 프레임을 그린다**는 의미다. 만약 한 프레임의 렌더링이나 연산이 16.67ms를 초과하면, 그 프레임은 제때 화면에 그려지지 못하고 다음 주기에 밀려나면서 화면이 끊기거나 사용자 입력 반응이 늦어지는 문제가 발생한다. 따라서 **60FPS를 안정적으로 유지하려면 각 프레임 처리를 반드시 16.67ms 이내에 끝내야 한다.**

</aside>

실제 코드로 살펴보자.

```tsx
export default function **UserDashboard**() {
  const apiUrl = '~~~'
  const {
	  // feedback 정보를 가지는 상태
    **items: feedbacks,**
    fetchMore,
    hasNext,
    loading,
  } = useCursorInfiniteScroll<
    FeedbackType,
    'feedbacks',
    FeedbackResponse<FeedbackType>
  >({
    url: apiUrl,
    key: 'feedbacks',
    size: 10,
    enabled: shouldUseInfiniteScroll,
  });

  return (
    <div css={dashboardLayout}>
      <DashboardOverview />
      <FilterSection />
      <div>
        <FeedbackBoxList>
          {feedbacks.map((feedback: FeedbackType) => (
            <UserFeedback
             {...feedback}
            />
          ))}
        </FeedbackBoxList>
      ...
    </div>
  );
}
```

위 코드는 실제 코드를 간략화한 버전이다. 코드를 살펴보면 무한스크롤로 가져오는 피드백 상태인 `feedbacks` 상태를. `UserDashboard` 에서 관리해서 발생한 문제임을 알 수 있다.

무한스크롤로 새로운 피드백을 가져오면 `feedbacks` 상태가 바뀌고 상태가 바뀌니 상태를 가진 `UserDashboard` 컴포넌트가 랜더링되면서 하위에 존재하는 모든 컴포넌트가 렌더링 되면서 발생한 문제였다.

즉 문제 상황을 정리해보면 다음과 같다.

1. `feedbacks` 상태가 `UserDashboard`에서 관리되어서 불필요하게 다른 컴포넌트가 렌더링되는 문제
2. 새로운 피드백을 가져와서 렌더링 될 때, 같은 데이터를 사용하는 `UserFeedback` 컴포넌트도 모두 새롭게 계산 되는 문제

# 문제 해결1 - 구조 개선

첫번 째 문제인 “`feedbacks` 상태가 `UserDashboard`에서 관리되어서 불필요하게 다른 컴포넌트가 렌더링되는 문제”는 구조를 개선해서 해결했다.

다시 현재 구조를 간략화하게 살펴보면 아래와 같다.

<img width="2536" height="972" alt="image" src="https://github.com/user-attachments/assets/4196e4cb-8569-4beb-abd6-b9a2d745d4ca" />


구조를 개선하기 위해서 feedbacks란 상태를 올바른 위치로 옮겨주어야 한다. 지금은 UserFeedback 컴포넌트들에만 사용되므로 우선 map 메서드로 그냥 렌더링 했던 UserFeedback 컴포넌트들을 UserFeedbackList 컴포넌트로 묶고 feedbacks란 상태를 UserFeedbackList 컴포넌트로 옮겨주었다. 이렇게 구조를 개선하고 다시 살펴보면 다음과 같다.

<img width="2542" height="1100" alt="image" src="https://github.com/user-attachments/assets/0652e2f9-c863-4fac-acc9-546ccb486848" />


실제 코드로도 살펴보자. 이전에 데이터 패칭과 `feedbacks`라는 상태가 `UserFeedbackList` 컴포넌트 내부로 들어가 `UserDashboard` 컴포넌트 자체가 상당히 가벼워지고, 훨씬 더 이해하기 좋은 형태로 변경되었다.

```tsx
export default function **UserDashboard**() {

  return (
    <div css={dashboardLayout}>
      <DashboardOverview />
      <FilterSection />

      /*fedbacks 상태와 UserFeedback컴포넌트들을 UserFeedbackList컴포넌트로 묶어주기*/
      <UserFeedbackList />
    </div>
  );
}
```

이렇게 하고, Profiler를 다시 확인해본 결과 다음과 같이 이제는 불필요하게 다른 컴포넌트가 렌더링되는 것을 막을 수 있었고, 렌더링 범위를 `UserFeedbackList` 컴포넌트로 좁힐 수 있었다.

<img width="2322" height="568" alt="image" src="https://github.com/user-attachments/assets/2430a9cf-ac5b-4de6-ae43-23d1fe923b92" />


이렇게 1번 문제인 “`feedbacks` 상태가 `UserDashboard`에서 관리되어서 불필요하게 다른 컴포넌트가 렌더링되는 문제”를 해결했다.

# 문제 해결2 - react api 사용

이제 2번 문제인 “새로운 피드백을 가져와서 렌더링 될 때, 같은 데이터를 사용하는 `UserFeedback` 컴포넌트도 모두 새롭게 계산 되는 문제”를 해결해 보자.

앞서 `UserFeedbackList` 컴포넌트로 묶고 feedbacks란 상태를 해당 컴포넌트에 넘겨도 아직도 발생하는 문제인 기존 데이터 즉 똑같은 데이터를 사용해도 발생하는 렌더링 문제다. 사실 우리 서비스의 렉을 유발하는 핵심 문제는 이거다.

사실 이거는 생각보다 쉽게 해결할 수 있다. 기존 데이터를 사용하는 경우 즉 props가 이전 렌더링과 동일하다면 렌더링 되는 것을 막아주는 React api에서 제공하는 `React.memo`를 사용해주면 된다.

```tsx

export default function **React.memo**(UserFeedback({
 ...
}: UserFeedbackBox) {
  const theme = useAppTheme();

  return (
    <FeedbackBoxBackGround type={type} customCSS={customCSS}>
      <FeedbackBoxHeader
        userName={userName + (isMyFeedback ? ' (나)' : '')}
        type={type}
        feedbackId={feedbackId}
        category={category}
      />
      ...
      <FeedbackBoxFooter
        type={type}
        isLiked={isLiked}
        postedAt={postedAt}
        isSecret={isSecret}
        feedbackId={feedbackId}
        likeCount={likeCount}
      />
    </FeedbackBoxBackGround>
  );
})

```

위 코드와 같이 UserFeedback 컴포넌트를 React.memo로 감싼 컴포넌트를 내보내도록 설정했다.

<aside>

React.memo란?

React.memo는 props가 바뀌지 않으면 컴포넌트가 렌더링을 건너뛰게 해주는 고차 컴포넌트다.

</aside>

이렇게 코드를 수정하고 다시 Profiler를 측정해본 결과 새로 가져오는 피드백만 렌더링이 발생했다.

<img width="2280" height="828" alt="image" src="https://github.com/user-attachments/assets/586e228e-2e9e-424a-8ba9-71249c3277b7" />


# 결과

이렇게 최적화를 완료했다. 최적화 전과 후를 비교해보자. 전에는 모든 컴포넌트가 렌더링 되었지만 최적화 후에는 꼭 필요한 부분인 새롭게 피드백을 가져오는 부분만 렌더링이 발생했다.

<img width="2570" height="608" alt="image" src="https://github.com/user-attachments/assets/0d496910-9f66-4fee-ae09-faa435c3ffe1" />


이런 최적화 결과는 개발자 도구 성능 탭에서도 크게 체감할 수 있었다.

<img width="1628" height="1322" alt="image" src="https://github.com/user-attachments/assets/c8a07645-3ca9-451b-9be8-09e8a70186fc" />


최적화 전에는 브라우저 메인 쓰레드에 무수히 많은 작업이 존재하고 많은 프레임 드랍이 존재했지만, 최적화 후에는 프레임 드랍과 메인 쓰레드의 작업이 눈에 띄게 줄어든 것을 확인할 수 있었다.

