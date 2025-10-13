# 서론

피드줍줍 서비스는 무한스크롤 방식으로 새로운 피드백을 불러와 화면에 보여준다. 그런데 어느 날 200개의 피드백을 스크롤하여 표시했을 때는 문제가 없었지만, 그 이상을 가져오자 화면이 버벅거리고 모달을 열 때도 한 박자 늦게 반응하는 현상이 발생했다. 이번 글에서는 이러한 성능 문제를 어떻게 발견했고, 어떤 과정을 거쳐 해결했는지 정리해 보려고 한다.

# 현재 구조

우선 문제의 원인을 찾기 전, 문제가 발생하는 페이지인 `UserDashboard`페이지의 구조를 살펴보면 아래와 같다.

![image.png](attachment:ca9e1bae-0853-499d-8e17-ebe62389fd9a:image.png)

이를 트리 구조로 바꿔서 보면 아래와 같다.

![image.png](attachment:d17d56f6-c53c-42ec-937c-0f1fb61677ad:image.png)

# 문제 찾기

우선 문제의 원인을 찾기 위해서, 브라우저 메인 쓰레드의 과부화 상태를 확인하기 위해서 무한스크롤을 할 때, 브라우저 메인 쓰레드에 어떤 일이 일어나는 지 측정해봤다.

그 결과 아래와 같이 나왔다.

![image.png](attachment:771c8599-f7c7-464e-8b06-64efa95d9ff8:image.png)

결과를 살펴 보니, 메인 쓰레드의 일이 너무 많았고, 그 결과 16ms이내 한 프레임을 그리지 못하는 탓에 렉 걸리고 늦게 반응하는 현상이 발생했다. 함수 호출을 살펴보니, 새로운 피드백을 받아올 때 매번 기존 피드백들도 새롭게 리렌더링이 발생하고 있었다. 이는 리액트 devtools를 확인해봐도 알 수 있다. `Highlight updates when components render` 설정 옵션을 켜고 무한스크롤 시 렌더링을 측정해보았다.

[화면 기록 2025-09-20 20.10.58.mov](attachment:8efc4dee-ae48-461c-9763-4964173d2eda:화면_기록_2025-09-20_20.10.58.mov)

그 결과 매번 새로운 피드백을 받아올 때, 모든 컴포넌트가 새롭게 렌더링되는 것을 확인할 수 있었다. **Profiler**로 확인해보면 더 명확하다.

![image.png](attachment:37ce8791-611f-433f-95e6-cef43d1253d1:image.png)

새로 피드백을 받아올 때, `UserDashboard` 페이지 컴포넌트 전체가 렌더링되는 것을 확인할 수 있다.

### 이게 왜 문제가 될까?

우선 브라우저 메인 쓰레드의 1프레임을 그림으로 알아보면 아래와 같다.

![image.png](attachment:ffbd03fa-a3b0-47e1-b2c3-54a170c06864:image.png)

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

![image.png](attachment:cf961f3a-065d-4a6b-ae1e-b3710f735e02:image.png)

구조를 개선하기 위해서 feedbacks란 상태를 올바른 위치로 옮겨주어야 한다. 지금은 UserFeedback 컴포넌트들에만 사용되므로 우선 map 메서드로 그냥 렌더링 했던 UserFeedback 컴포넌트들을 UserFeedbackList 컴포넌트로 묶고 feedbacks란 상태를 UserFeedbackList 컴포넌트로 옮겨주었다. 이렇게 구조를 개선하고 다시 살펴보면 다음과 같다.

![image.png](attachment:830c46b5-134c-478e-b15c-35c117c3558f:image.png)

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

![image.png](attachment:7c581549-a1a4-4e36-86c9-74ab6d077706:image.png)

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

![image.png](attachment:e8e3bf45-08a9-4cec-a567-8da892008704:image.png)

# 결과

이렇게 최적화를 완료했다. 최적화 전과 후를 비교해보자. 전에는 모든 컴포넌트가 렌더링 되었지만 최적화 후에는 꼭 필요한 부분인 새롭게 피드백을 가져오는 부분만 렌더링이 발생했다.

![image.png](attachment:9aa16447-16fa-4a08-ab26-e7b4bd9fafeb:image.png)

이런 최적화 결과는 개발자 도구 성능 탭에서도 크게 체감할 수 있었다.

![image.png](attachment:6adb861a-8ceb-49be-a6aa-2858f4615d90:image.png)

최적화 전에는 브라우저 메인 쓰레드에 무수히 많은 작업이 존재하고 많은 프레임 드랍이 존재했지만, 최적화 후에는 프레임 드랍과 메인 쓰레드의 작업이 눈에 띄게 줄어든 것을 확인할 수 있었다.

## 대시보드 무한스크롤

### 전체 페이지가 리렌더링되는 문제

무한 스크롤시 그냥 모든 관리자 대시보드 컴포넌트가 리랜더링이 발생한다.

![image.png](attachment:a791eb0a-c2a8-4433-a7d5-8b3a89f4ea00:image.png)

위 이미지는 무한스크롤 순간을 profiler로 측정한 것으로 이전 피드백 데이터와 심지어 관리자 통계마저 리랜더링이 발생하는 것을 볼 수 있다.

사실 당연한 결과인데, 피드백 데이터를 받아오는 과정에서 `AdminDashboard`가 리렌더링이 되고 있다.

`AdminDashboard` 코드를 보니, `AdminDashboard` 컴포넌트에서 데이터 로딩하는 로직이 작성되어있어, 무한스크롤로 데이터 로딩시 모든 컴포넌트가 리렌더링이 발생하고 있다.

```tsx
export function AdminDashBoard(){
	... 여러가지 코드

	const apiUrl = createFeedbacksUrl({
    organizationId,
    sort: selectedSort,
    filter: selectedFilter,
    isAdmin: true,
  });

  const {
    items: feedbacks,
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
  });

	useGetFeedback({ fetchMore, hasNext, loading });

	return (
	    <section css={dashboardLayout}>
	      <DashboardOverview />

	      <FilterSection
	        selectedFilter={selectedFilter}
	        onFilterChange={handleFilterChange}
	        selectedSort={selectedSort}
	        onSortChange={handleSortChange}
	        isAdmin={true}
	      />


	      <FeedbackBoxList>
	        {feedbacks.map((feedback) => (
	          <AdminFeedbackBox
	            key={feedback.feedbackId}
	            feedbackId={feedback.feedbackId}
	            onConfirm={openFeedbackCompleteModal}
	            onDelete={openFeedbackDeleteModal}
	            type={feedback.status}
	            content={feedback.content}
	            postedAt={feedback.postedAt}
	            isSecret={feedback.isSecret}
	            likeCount={feedback.likeCount}
	            userName={feedback.userName}
	            category={feedback.category}
	            comment={feedback.comment}
	          />
	        ))}
	      </FeedbackBoxList>

	      <FeedbackStatusMessage
	        loading={loading}
	        filterType={selectedFilter as FeedbackFilterType}
	        hasNext={hasNext}
	        feedbackCount={feedbacks.length}
	      />

	      {hasNext && <div id='scroll-observer' style={{ minHeight: '1px' }} />}
	      ...
	  }
  }
```

사실 데이터 패칭 로직이 `AdminDashboard`에 있을 이유가 없다. 데이터가 필요한 곳은 **피드백 리스트**이므로, 데이터 패칭 로직을 피드백 리스트 컴포넌트 내부로 옮겨보자. 우선, 그러기 위해서는 무한스크롤을 감지하는 요소와 모두 보았을 때 나오는 ui도 피드백 리스트 컴포넌트로 내부로 옮기자.

근데 `FeedbackBoxList` 컴포넌트는 단지 스타일만 가지는 컴포넌트다. 이렇게 만든 이유는 사용자와 관리자 대시보드 모두 필요한 컴포넌트이므로 이렇게 구현했다. 따라서, `FeedbackBoxList`에서 관리자 데이터 패칭을 진행하면 사용자 대시보드에서 재사용을 하지 못하므로, `AdminFeedbackList` 라는 컴포넌트를 하나 만들어서 `FeedbackBoxList`를 **랩핑하였다.**

```tsx
export default function AdminFeedbackList({
  selectedFilter,
  selectedSort,
  openFeedbackCompleteModal,
  openFeedbackDeleteModal,
}: AdminFeedbackListProps) {
  const { organizationId } = useOrganizationId();

  const apiUrl = createFeedbacksUrl({
    organizationId,
    sort: selectedSort,
    filter: selectedFilter,
    isAdmin: true,
  });

  const {
    items: feedbacks,
    fetchMore,
    hasNext,
    loading,
  } = useCursorInfiniteScroll<
    FeedbackType,
    "feedbacks",
    FeedbackResponse<FeedbackType>
  >({
    url: apiUrl,
    key: "feedbacks",
    size: 10,
  });

  useGetFeedback({ fetchMore, hasNext, loading });

  return (
    <div>
      <FeedbackBoxList>
        {feedbacks.map((feedback) => (
          <AdminFeedbackBox
            key={feedback.feedbackId}
            feedbackId={feedback.feedbackId}
            onConfirm={openFeedbackCompleteModal}
            onDelete={openFeedbackDeleteModal}
            type={feedback.status}
            content={feedback.content}
            postedAt={feedback.postedAt}
            isSecret={feedback.isSecret}
            likeCount={feedback.likeCount}
            userName={feedback.userName}
            category={feedback.category}
            comment={feedback.comment}
          />
        ))}
      </FeedbackBoxList>
      <div>
        <FeedbackStatusMessage
          loading={loading}
          filterType={selectedFilter as FeedbackFilterType}
          hasNext={hasNext}
          feedbackCount={feedbacks.length}
        />

        {hasNext && <div id="scroll-observer" style={{ minHeight: "1px" }} />}
      </div>
    </div>
  );
}
```

이제 `AdminFeedbackList` 컴포넌트에 데이터 패칭 로직이 옮겨지면서 `AdminDashBaord` 컴포넌트의 역할이 훨씬 줄어들었고 코드도 깔끔해졌다.

```tsx
<AdminFeedbackList
  selectedFilter={selectedFilter}
  selectedSort={selectedSort}
  openFeedbackCompleteModal={openFeedbackCompleteModal}
  openFeedbackDeleteModal={openFeedbackDeleteModal}
/>
```

`AdminDashBoard` 컴포넌트에서 위와 같이 `AdminFeedbackList`만 호출하면 된다.

이렇게 하고 다시 profiler를 측정하면 아래와 같다.

![image.png](attachment:8ba2743a-6380-4217-a04c-c4ceff5ef12d:image.png)

방금 `AdminDashBoard` 전체가 리렌더링이 된것과 달리, 지금은 `AdminFeedbackList` 컴포넌트만 리렌더링이 발생하고 있다!

### 새로운 피드백 로딩시 기존 피드백 리렌더링이 되는 문제

`AdminDashBoard` 전체가 리렌더링되는 문제는 해결했는데, 새로운 피드백 로딩시 기존 피드백이 리렌더링되는 문제는 해결하지 못했다.

이를 해결하기 위해, props가 변경되지 않으면 리렌더링이 발생하지 않도록 하기 위해 `React.memo`를 사용했다.

![image.png](attachment:3333e485-bd5d-4fb1-8dd6-9a27a5dd5102:image.png)

이렇게 했을 때 새로 데이터를 불러오는 피드백 컴포넌트만 리렌더링이 발생한 것을 볼 수 있다!!!!!

## 액션 버튼 눌렀을 때 과도한 리렌더링 문제

관리자가 피드백의 삭제버튼과 완료 버튼을 눌렀을 때 과도한 리렌더링이 발생한다.

[화면 기록 2025-09-16 13.54.53.mov](attachment:203b67ee-d7ed-422b-9fca-f29cf4435365:화면_기록_2025-09-16_13.54.53.mov)

![image.png](attachment:42bc9044-dc6b-4152-8776-8034c206f9f2:image.png)

삭제 버튼을 눌렀을 때 모든 컴포넌트가 리렌더링되는 것을 볼 수 있다.

우선 모달창을 열 때 리렌더링되면 안되는 요소인 `피드백 리스트`, `통계`, `필터` 의 리렌더링을 막아보자.

우선 각각의 요소에 `React.memo`를 적용하여 부모 컴포넌트가 리렌더링이 발생하더라도, props가 변경되지 않으면 리렌더링이 되지 않도록 한다.

[화면 기록 2025-09-16 14.13.35.mov](attachment:ba129f4c-d05a-4e74-941b-2ea56db7b278:화면_기록_2025-09-16_14.13.35.mov)

![image.png](attachment:a125fe1e-9d90-4bb8-ae6c-bd4ee4fb084b:image.png)

이렇게 적용했을 때 통계와 필터 영역은 리렌더링이 발생하지 않지만, `AdminFeedbackList`는 리렌더링이 발생하는 것을 볼 수 있다. 그이유는

모달을 여는 역할을 하는 함수인 `openFeedbackCompleteModal`, `openFeedbackDeleteModal` 함수를 리턴하는 훅인 `useAdminModal` 이 `AdminDashboard` 컴포넌트가 매번 리랜더링될 때마다 재실행되면서, 모달을 여는 역할하는 함수들이 재생성 된다.

```
 <AdminFeedbackList
        selectedFilter={selectedFilter}
        selectedSort={selectedSort}
        openFeedbackCompleteModal={openFeedbackCompleteModal}
        openFeedbackDeleteModal={openFeedbackDeleteModal}
      />
```

위 코드와 같이 매번 재생성되는 함수를 props로 넘기게 되면서 `AdminFeedbackList`의 **`props`**가 **리렌더링마다 변경되면서 리렌더링이 발생한다.**

그래서 이를 막기 위해서, 훅에서 반환하는 함수들에 `useCallback`을 사용하여 의존성 배열이 변경되지 않으면 리렌더링 발생하지 않도록 했다.

이렇게 하고 결과를 보면

[화면 기록 2025-09-16 14.19.15.mov](attachment:7cd5f885-56b2-47b5-8177-7809f9c2ae32:화면_기록_2025-09-16_14.19.15.mov)

![image.png](attachment:b7e1a23a-3530-4a38-9be4-66fffec8c4b9:image.png)

짠! 이제는 딱 필요한 부분인 `ConfirmModal`만 렌더링되는 것을 확인할 수 있다.

## 응원 버튼 눌렀을 때 통계 전체가 리렌더링 되는 문제

응원버튼을 눌렀을 때 리렌더링이 불필요한 통계 컴포넌트가 리렌더링이 발생하고 있다.

[화면 기록 2025-09-16 14.34.38.mov](attachment:64ddbe5e-6b63-491b-8263-312f73d0ca53:화면_기록_2025-09-16_14.34.38.mov)

![image.png](attachment:74df4d1d-1a5f-4353-832d-ffe70477ef01:image.png)

왜 이런가 살펴보니, `DashboardOverview.tsx`에서 응원하기 데이터와 통계 데이터를 불러오고 있다. 그러다 보니, 응원하기 데이터가 변경되어도 통계 컴포넌트가 리렌더링이 발생하는 것이였다.

그래서 이를 해결하기 위해, `OverviewHeader`라는 **랩핑 컴포넌트**를 만들어, **groupName, 응원하기 데이터를 가져왔다.**

```tsx
export default function OverviewHeader() {
  const { organizationId } = useOrganizationId();

  // 조직 이름, 응원하기 정보 데이터 가져오기
  const { groupName, totalCheeringCount } = useOrganizationName({
    organizationId,
  });
  const { handleCheerButton, animate } = useCheerButton({
    organizationId,
  });

  return (
    <div css={headerContainer}>
      <div css={headerText}>
        <p css={titleText(theme)}>{groupName}</p>
        <p css={panelCaption(theme)}>지금까지의 피드백</p>
      </div>
      <div css={headerCheerButton}>
        <div css={cheerButtonLayout}>
          <CheerButton
            totalCheeringCount={totalCheeringCount}
            onClick={handleCheerButton}
            animate={animate}
          />
        </div>
      </div>
    </div>
  );
}
```

해당 컴포넌트를, `DashboardOverview`에서 불러와서 사용한다.

```tsx
import DashboardPanel from "@/domains/components/DashboardPanel/DashboardPanel";
import { useAppTheme } from "@/hooks/useAppTheme";
import useUserOrganizationsStatistics from "@/domains/hooks/useUserOrganizationsStatistics";

import { useOrganizationId } from "@/domains/hooks/useOrganizationId";
import { panelLayout } from "./DashboardOverview.style";
import OverviewHeader from "./OverviewHeader";

export default function DashboardOverview() {
  const { organizationId } = useOrganizationId();
  const theme = useAppTheme();
  const { statistics } = useUserOrganizationsStatistics({
    organizationId,
  });
  return (
    <>
      **
      <OverviewHeader />
      **
      <div css={panelLayout}>
        {DASH_PANELS.map((panel, idx) => (
          <DashboardPanel
            key={idx}
            title={panel.title}
            content={panel.content}
            caption={panel.caption}
            color={panel.color}
          />
        ))}
      </div>
    </>
  );
}
```

이렇게 했을 때 `DashboardOverview`에서 `DashboardPanel`은 반복문으로 사용하는 것이 어색해서 해당 부분도 `DashboardPanelContent` 컴포넌트로 분리했다.

```tsx
import OverviewHeader from "./OverviewHeader/OverviewHeader";
import DashboardPanelContent from "./DashboardPanelContent/DashboardPanelContent";
import React from "react";

function DashboardOverview() {
  return (
    <>
      <OverviewHeader />
      <DashboardPanelContent />
    </>
  );
}

export default React.memo(DashboardOverview);
```

그러면 `DashboardOverview`가 이렇게 바뀐다!

[화면 기록 2025-09-16 14.53.56.mov](attachment:be961846-9012-4b92-b58d-bbe1fb9b11f8:화면_기록_2025-09-16_14.53.56.mov)

![image.png](attachment:b3442553-12b4-486e-9291-a8e0ae75935f:image.png)

이제 응원버튼을 눌렀을 때 해당 부분만 변경되는 것을 볼 수 있다

## 더보기 버튼 눌렀을 때 리렌더링 되는 문제

[화면 기록 2025-09-16 15.16.35.mov](attachment:185c7d03-ba46-408b-8a6a-7fe462397d2c:화면_기록_2025-09-16_15.16.35.mov)

![image.png](attachment:54875d57-4b7e-4503-afe7-ba6740f3b82f:image.png)

더보기 버튼을 클릭하는데 뒤로가기 버튼, 텍스트가 리렌더링되는 것을 볼 수 있다.

`Header` 코드를 보면 아래와 같다.

```tsx
import { useAppTheme } from "@/hooks/useAppTheme";
import MoreVerticalIcon from "../icons/MoreVerticalIcon";
import {
  arrowTitleContainer,
  captionSection,
  header,
  headerSection,
  headerSubtitle,
  headerTitle,
  MoreButton,
  moreMenu,
  moreMenuContainer,
} from "./Header.style";

import Button from "../@commons/Button/Button";

import ArrowLeftIcon from "../icons/ArrowLeftIcon";
import MoreMenu from "@/components/Header/MoreMenu/MoreMenu";
import useMoreMenuManager from "@/components/Header/hooks/useMoreMenuManager";
import { useLayoutConfig } from "@/hooks/useLayoutConfig";
import useNavigation from "@/domains/hooks/useNavigation";

export default function Header() {
  const theme = useAppTheme();
  const { goBack } = useNavigation();
  const { layoutConfig } = useLayoutConfig();

  const { isOpenMoreMenu, toggleMoreMenu, moreButtonRef, closeMoreMenu } =
    useMoreMenuManager();

  const { title, subtitle, hasMoreIcon, showBackButton } = layoutConfig.header;

  return (
    <header css={header(theme)}>
      <div css={arrowTitleContainer}>
        {showBackButton && (
          <Button onClick={goBack}>
            <ArrowLeftIcon color={theme.colors.white[100]} />
          </Button>
        )}
        <div css={headerSection}>
          <div css={captionSection}>
            <p css={headerTitle(theme)}>{title}</p>
            <p css={headerSubtitle(theme)}>{subtitle}</p>
          </div>
        </div>
      </div>
      {hasMoreIcon && (
        <div
          css={moreMenuContainer}
          ref={moreButtonRef as React.RefObject<HTMLDivElement>}
        >
          <Button onClick={toggleMoreMenu} customCSS={MoreButton}>
            <MoreVerticalIcon />
          </Button>
          {isOpenMoreMenu && (
            <div css={moreMenu}>
              <MoreMenu closeMoreMenu={closeMoreMenu} />
            </div>
          )}
        </div>
      )}
    </header>
  );
}
```

문제는 더보기 관련 상태가 `Header`컴포넌트에 그대로 노출되어 있어, 더보기 버튼을 눌렀을 때 `Header` 컴포넌트 전체가 리렌더링되는 것이었다.

이것도 아까와 똑같이 **컴포넌트로 분리하고 분리한 컴포넌트 내부에서만 상태가 변경되도록 변경**해보자.

```tsx
{
  hasMoreIcon && (
    <div
      css={moreMenuContainer}
      ref={moreButtonRef as React.RefObject<HTMLDivElement>}
    >
      <Button onClick={toggleMoreMenu} customCSS={MoreButton}>
        <MoreVerticalIcon />
      </Button>
      {isOpenMoreMenu && (
        <div css={moreMenu}>
          <MoreMenu closeMoreMenu={closeMoreMenu} />
        </div>
      )}
    </div>
  );
}
```

```tsx
{
  hasMoreIcon && <HeaderMoreIcon />;
}
```

이렇게 컴포넌트를 분리하고 다시 측정하면 해결된다!

[화면 기록 2025-09-16 15.24.53.mov](attachment:2540ec8a-1a48-4397-b9d0-db57cf28f7ff:화면_기록_2025-09-16_15.24.53.mov)

![image.png](attachment:df124758-9fac-4c91-8d5e-687e27a7d819:image.png)

---

# 사용자

## 무한스크롤 시 피드백 리스트 전체 리렌더링되는 문제

[화면 기록 2025-09-16 15.33.05.mov](attachment:cd28b71e-1ae9-433a-ae01-4bf8765814b2:화면_기록_2025-09-16_15.33.05.mov)

![image.png](attachment:155b46b2-7882-490e-bb9a-db741ef1af77:image.png)

사용자 대시보드 페이지에서도 동일하게 피드백 리스트를 불러올때 과도한 리렌더링이 발생하고 있다.

관리자 대시보드와 똑같이 사용자 대시보드에서도 `UserFeedbackList` **컴포넌트를 만들어 해당 컴포넌트에서 데이터 패칭하는 로직을 작성**하고 `UserFeedbackBox`에 `react.memo`를 적용해보자.

그러면 아래와 같이 사용자 대시보드 컴포넌트가 작성된다.

```tsx
return (
  <div css={dashboardLayout}>
    <DashboardOverview />
    <FilterSection
      selectedFilter={selectedFilter}
      onFilterChange={handleFilterChange}
      selectedSort={selectedSort}
      onSortChange={handleSortChange}
      isAdmin={false}
    />
    <UserFeedbackList
      selectedFilter={selectedFilter}
      selectedSort={selectedSort}
    />
    <FloatingButton
      icon={<ArrowIcon />}
      onClick={handleNavigateToOnboarding}
      inset={{ bottom: "32px", left: "100%" }}
      customCSS={goOnboardButton(theme)}
    />
    {showButton && (
      <FloatingButton
        icon={<ArrowUpIcon />}
        onClick={scrollToTop}
        inset={{ bottom: "32px" }}
        customCSS={goTopButton(theme)}
      />
    )}
  </div>
);
```

이렇게 하고 다시 측정했는데 **똑같이 과도한 리렌더링이 발생한다…**

왜 그런가 `React Devtools profier`를 자세히 보니

![image.png](attachment:b6069911-dbb0-405c-aa78-f8be8318c762:image.png)

요기 `props`가 변경되어서 렌더링이 발생했다는 것을 알 수 있다. 그중 `customCSS`가 **매번 변경되는 것**을 알 수 있다.

```tsx
 <FeedbackBoxList>
          {displayFeedbacks.map((feedback: FeedbackType) => (
            <UserFeedbackBox
              userName={feedback.userName}
              key={feedback.feedbackId}
              type={feedback.status}
              content={feedback.content}
              postedAt={feedback.postedAt}
              isLiked={getFeedbackIsLike(feedback.feedbackId) || false}
              isSecret={feedback.isSecret}
              feedbackId={feedback.feedbackId}
              likeCount={feedback.likeCount}
              comment={feedback.comment}
              isMyFeedback={myFeedbacks.some(
                (myFeedback) => myFeedback.feedbackId === feedback.feedbackId
              )}
              **customCSS={[
                feedback.feedbackId === highlightedId ? highlightStyle : null,
              ]}**
              category={feedback.category}
            />
          ))}
        </FeedbackBoxList>
```

위 코드를 보면 `customCSS`를 배열로 넘겨주는데 **이게 매번 새로 계산되면서 새로운 배열**이 `props`로 넘어가면서 발생한 문제다.

이를 해결하기 위해서, `UserFeedbackBox`에 `customCSS`를 계산해서 넘겨주던 코드를 `isHighlighted`라는 `props`를 넘기는 방식으로 수정하고, `UserFeedbackBox`에서 `isHighlighted`값에 따라 CSS를 적용해주는 방식으로 수정했다.

```tsx
  <FeedbackBoxList>
          {displayFeedbacks.map((feedback: FeedbackType) => (
            <UserFeedbackBox
              userName={feedback.userName}
              key={feedback.feedbackId}
              type={feedback.status}
              content={feedback.content}
              postedAt={feedback.postedAt}
              isLiked={getFeedbackIsLike(feedback.feedbackId) || false}
              isSecret={feedback.isSecret}
              feedbackId={feedback.feedbackId}
              likeCount={feedback.likeCount}
              comment={feedback.comment}
              isMyFeedback={myFeedbacks.some(
                (myFeedback) => myFeedback.feedbackId === feedback.feedbackId
              )}
              **isHighlighted={feedback.feedbackId === highlightedId}**
              category={feedback.category}
            />
          ))}
        </FeedbackBoxList
```

```tsx

function UserFeedbackBox({
...
}: UserFeedbackBox) {

  return (
    <FeedbackBoxBackGround
      type={type}
      // css로직을 여기서 결정
      **customCSS={[isHighlighted ? highlightStyle : null]}**
    >
      <FeedbackBoxHeader
        userName={userName + (isMyFeedback ? ' (나)' : '')}
        type={type}
        feedbackId={feedbackId}
        category={category}
      />
      <div css={isSecret ? secretText(theme) : undefined}>
        {isSecret ? (
          isMyFeedback ? (
            <FeedbackText type={type} text={content} />
          ) : (
            <p>비밀글입니다.</p>
          )
        ) : (
          <FeedbackText type={type} text={content} />
        )}
        {isSecret && <LockIcon />}
      </div>
      {(!isSecret || isMyFeedback) && type === 'CONFIRMED' && comment && (
        <FeedbackAnswer answer={comment} />
      )}

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
}

export default React.memo(UserFeedbackBox);

```

이렇게 하고 측정하면 해결된다!

[화면 기록 2025-09-16 16.21.59.mov](attachment:9bea61f8-8099-4d35-84e2-d6fc48ee2e25:화면_기록_2025-09-16_16.21.59.mov)

![image.png](attachment:c128fe52-024d-4ef8-a3cf-9b5b31390d1d:image.png)

---

# 개발자 도구 성능 탭 비교

![개선 전](attachment:16c96ab9-e317-40b8-8717-31b08b6f8351:image.png)

개선 전

![개선 후](attachment:e5bb57cb-cdfb-41b8-aa45-ee2e332f322e:image.png)

개선 후

실제로 fram drop이 많이 사라졌고, 무거운 작업도 많이 사라진것을 볼 수 있다!!

![image.png](attachment:1a6bfa4d-403f-4add-8a6d-cc73ca8f4da5:image.png)

[]()

![image.png](attachment:a652b336-9e3e-483f-bb31-6cdfe8b478ae:image.png)
