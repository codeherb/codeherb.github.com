# Jetpack Compose Modifier 란?

컴포저블을 꾸며주는 도구로, 일반적인 코틀린 오브젝트로 구성되어 있다.
단순한 정보의 입력(size, padding, margin ...)부터 
하이레벨 인터렉션 (clickable, scrollable, draggable, zoomable)을 담을 수 있으며,
일반 코틀린 오브젝트 이므로 변수에 넣어놓고 재 사용 가능한 한편, 모디파이어를 결합할 수도 있다.

## 스코프

일부 레이아웃의 경우, 자신의 스코프 내에서만 사용 할 수 있는 별도의 모디파이어를 가지고 있다.
개별 모디파이어는 코틀린 확장 함수와 팩토리 패턴을 활용한 함수를 통해 생성 가능하며,
이때 레이아웃 스코프에 대한 확장 함수를 정의 함으로써 사용가능 한 모디파이어에 대한
컴파일 타임 타입 체킹을 제공한다.
```
@LayoutScopeMarker
@Immutable
interface RowScope {
    @Stable
    fun Modifier.weight(
        /*@FloatRange(from = 0.0, fromInclusive = false)*/
        weight: Float,
        fill: Boolean = true
    ): Modifier
    
    ...
}
```
```
Row {
    Surface(
       modifier = Modifier.weight(1f),
       color = MaterialTheme.colors.onSurface.copy(alpha = 0.2f)
    ) {
       ...
    }
}
```

## 모디파이어 순서

// TODO

## 커스터마이징

// TODO
