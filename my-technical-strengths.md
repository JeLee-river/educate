# `transform`의 렌더링 최적화 원리

## 배경

프로젝트에서 Carousel 컴포넌트의 스와이프 기능을 구현한 후 코드 리뷰를 받던 중, 스와이프 이벤트마다 **불필요한 리렌더링**이 발생하는 문제를 확인했다.

React DevTools Profiler로 측정한 결과, **스와이프 이벤트가 발생할 때마다 컴포넌트가 리렌더링**되고 있었다. 스와이핑 되는 매 프레임마다 상태 업데이트가 발생했고, 이는 불필요한 렌더링을 유발하였다.

<img width="912" height="636" alt="image" src="https://github.com/user-attachments/assets/e0f4c150-16eb-4960-9763-06a270873651" />


<br>

### 기존 코드의 문제점

```tsx
const [swipeOffset, setSwipeOffset] = useState(0);

const handleTransitionEnd = () => {
  setIsSwiping(true);
  swipeStartRef.current = clientX;
  setSwipeOffset(0);
};

const swipeMove = (clientX: number) => {
  if (!isSwiping) return;
  const offset = clientX - swipeStartRef.current;
  setSwipeOffset(offset); // 매 프레임마다 상태 업데이트 발생
};
```

이 코드의 핵심 문제는 **스와이프 오프셋을 React 상태로 관리**했다는 점이다. 사용자가 스와이프할 때마다 `setSwipeOffset`이 호출되었고 상태가 변경되어 리렌더링을 유발했다.

https://github.com/user-attachments/assets/01e3b8a4-84ab-4348-86fa-e6dd1b3e0161


<br>

### 개선된 코드: 직접 DOM 조작과 Transform 활용

아래의 변경사항을 적용해 불필요한 리렌더링을 막고 최적화를 적용했다.

1. **`useRef`로 변경**: 상태 대신 Ref를 사용해 리렌더링 방지
2. **직접 DOM 조작**: `style.transform`을 직접 수정해 React 렌더 사이클 우회
3. **`transform` 속성 사용**: 브라우저 렌더링 최적화 활용

```tsx
const swipeOffsetRef = useRef(0);
const slideWrapperRef = useRef<HTMLDivElement>(null);

const handleTransitionEnd = () => {
  setIsSwiping(true);
  swipeStartRef.current = clientX;
  swipeOffsetRef.current = 0;
};

const swipeMove = (clientX: number) => {
  if (!isSwiping) return;
  const offset = clientX - swipeStartRef.current;
  swipeOffsetRef.current = offset;
  updateTransform(); // React 상태 없이 직접 DOM 업데이트
};

const updateTransform = useCallback(() => {
  if (!slideWrapperRef.current) return;
  const offset = swipeOffsetRef.current;
  slideWrapperRef.current.style.transform = `translateX(calc(-${
    slideIndex * 100
  }% + ${offset}px))`;
}, [slideIndex]);
```

<img width="911" height="531" alt="image" src="https://github.com/user-attachments/assets/a02fa6fa-89f1-4b88-a36c-8a08e897baf3" />


<br>

## 브라우저 렌더링 파이프라인

문제는 해결했지만, `transform`이 어떻게 렌더링 성능을 최적화하는지는 구체적으로 알지 못했다. 위 사례에서는 렌더링 최적화 보다는 스와이핑 기능을 구현하기 위해 transform을 선택한 것에 더 가깝지만, `right`, `left` 등의 속성 대신 `transform`이 렌더링에 유리한 이유에 의문이 있었다.

이에 따라, 구체적으로 내부 동작을 조사했다.

### 브라우저 렌더링의 4단계

우선 `transform`의 렌더링 최적화에 대해 이해하려면 브라우저가 화면을 그리는 과정을 이해해야 한다.
대략적인 순서는 다음과 같다.

#### 1. **Style 계산 (Recalculate Style)**

CSS 규칙을 계산하여 각 DOM 노드의 최종 스타일을 계산한다.

#### 2. **Layout (Reflow)**

각 요소의 크기와 위치를 계산한다. 예를 들어 `width`, `height`, `top`, `left` 등이 변경되면 이 단계가 다시 실행된다.

#### 3. **Paint**

각 레이어를 비트맵으로 래스터화(그리기)한다. 색상, 그림자, 배경 등 시각적 속성이 여기서 처리된다.

#### 4. **Composite**

GPU가 여러 레이어를 합성하여 최종 프레임을 생성한다.

### 속성별 렌더링 비용

| 속성                             | 영향 단계                  | 비용                       |
| -------------------------------- | -------------------------- | -------------------------- |
| `width`, `height`, `top`, `left` | Layout → Paint → Composite | **높음** (전체 파이프라인) |
| `color`, `background`            | Paint → Composite          | **중간** (Paint부터)       |
| `transform`, `opacity`           | Composite                  | **낮음** (Composite만)     |

<br>

### 왜 `transform`은 렌더링 최적화가 가능할까

`transform`과 `opacity`는 **합성 전용 속성(Compositor-Only Properties)** 이다.

합성 전용 속성은 아래의 특징을 가진다.

- **Layout을 건드리지 않으며,** 문서 흐름에 영향을 주지 않는다.
- **Paint를 건드리지 않으며,** 이미 래스터화된 비트맵을 재사용한다.
- **Composite만 실행하여** GPU가 이미 그려진 레이어를 변환(이동, 회전, 크기 조정)하기만 하면 된다.

따라서 메인 스레드가 JS 실행과 스타일 계산 등으로 바빠도, 합성 스레드는 **이미 래스터화된 비트맵들을 계속 합성**하여 부드러운 프레임을 유지할 수 있다.

<br>

## 합성 레이어(Compositing Layer)

### 합성 레이어란?

브라우저는 특정 조건을 만족하는 요소를 **별도의 합성 레이어**로 승격(Promote)시킨다. 승격된 요소는 다음과 같은 특징을 가진다.

1. **한 번만 래스터화**된다. (비트맵으로 그려짐)
2. **애니메이션 중에는 합성만 반복**한다. (다시 그리지 않음)
3. **GPU 가속**을 받는다.

<br>

### 레이어로 승격되는 조건

- `transform` 또는 `opacity` 애니메이션이 적용된 요소
- `will-change: transform` 또는 `will-change: opacity`가 선언된 요소
- 3D 변환 (`translateZ(0)`, `rotate3d()` 등)
- `<video>`, `<canvas>` 요소
- `position: fixed` (일부 경우)

승격되면 그 레이어는 한 번 페인트(래스터화)된 후, **애니메이션 중에는 합성만 반복한다.**

<br>

### 그렇다면 합성 레이어가 무조건 좋을까?

합성 레이어가 항상 좋은 것은 아니다. 다음 비용을 고려해야 한다

#### 1. **메모리 비용**

각 레이어는 **GPU 텍스처 메모리**를 차지한다. 큰 요소를 다수 승격하면 **메모리 급증 + 래스터화 비용이 증가한다.**

#### 2. **래스터화 비용**

레이어 내용이 자주 변경되면 **매번 다시 래스터화**해야 한다. 또한 스크린 밖으로 크게 스케일하거나, 자주 내용이 바뀌는 노드를 승격하면 래스터 빈도가 늘어 오히려 성능이 악화될 수 있다.

#### 3. **`will-change` 남용**

```css
// 모든 요소에 적용
* {
  will-change: transform, opacity;
}
```

위와 같이 모든 요소에 `will-change`를 적용하는 등, 이를 남용하면 수백 개의 레이어가 생성되면서 메모리 폭증으로 오히려 성능 저하될 수 있다.
