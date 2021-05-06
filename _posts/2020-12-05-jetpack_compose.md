# Jetpack Compose 란?

네이티브 안드로이드 UI를 만들기 위한 최신 UI 툴킷이자, 
함수 집합으로 UI를 디자인하는 선언적 UI 시스템 입니다.
함수 안에서 함수를 호출하고 이는 UI를 빌드하는데 필요한 트리 구조를 형성합니다.  
트리 구조는 다른 말로 UI Graph 라고 합니다.

## 주요 요소

### Composable

컴포즈 앱은 컴포저블 함수들로 구성됩니다.
컴포저블 함수는 일반 코틀린 함수에 `@Composable` 덧붙여 생성할 수 있습니다.

컴포지션은 UI를 기술하는 컴포저블의 트리 구조
컴포지션은 초기 컴포지션을 통해서만 생성되고 리컴포지션을 통해서만 업데이트될 수 있습니다.

### Modifier

모디파이어는 일반적인 코틀린 오브젝트로, 
UI엘리먼트가 부모레이아웃 내에서 어떻게 배치되고(ex : align), 
어떻게 동작해야하는지(ex : onClick)에 대한 정보를 가지고 담고 있습니다.

코틀린 객체이므로 변수에 할당하고 재사용 가능하며 확장함수를 통해 여러 모디파이어를 하나로 병합할 수도 있습니다.

### State

컴포즈의 UI 컴포넌트는 상태를 가지지 않습니다.
하지만 변화하는 UI를 표현하기 위해서는 상태를 추적하고 반응하기 위한 구조가 필요하고 컴포즈에서 이를 위해 제공하는 것이 바로 State 입니다.
State는 컴포즈 안에서 관찰 가능한 타입으로, LiveData나 Flow등 다양한 리액티브 스트림의 컴포즈 버전입니다.
State에 변경사항이 생길 경우 자동으로 컴포지션 그래프가 재구성되도록 예약이 이루어 집니다.

```
위와 같은 스트림을 사용하지 않고 컴포즈 내부에서 상태를 가지는 값을 선언하기 위해 
컴포즈는 remember 라는 것 또한 제공한다.
remember는 컴포저블 함수가 유지되는 동안 상태의 유지와 추적 매커니즘을 제공해 준다.
초기 컴포지션 시점에만 불변 혹은 가변 값으로 초기화 되며 시간이 지나 
재 컴포지션 될 때는 저장된 값을 반환하기 때문에 과거 상태를 유지하는데 유용하다.
```

### Effect

Compose는 크게 두가지의 흐름을 가집니다.
- Composition Phase - 어떤 위젯을 그릴지에 대해 정의 하는 시점으로 이 시점에 컴포즈의 UI트리가 구성된다.
- Execution Phase - 실제 컴포넌트를 화면에 렌더링 하는 단계.

이처럼 컴포즈는 모든 작용들이 즉시 적용되지 않고 함수형 프로그래밍 패러다임의 지연 평가 개념과 같이 
실제 수행은 최대한 뒤로 미루어지며, 실행 시점은 컴포즈의 런타임에 의해서 관리됩니다.

컴포저블 함수의 스코프를 벗어난 모든 작용들을 이팩트 라고 부르며,
선언된 이팩트는 런타임에 의해 적절한 시점에 실행 및 제거가 이루어 집니다.