---
title: (번역) A Functional Approach to Android Architecture using Kotlin
tags: fp, reactive, kotlin, android
key : post-functional-approach-to-android
header:
theme: dark
background: 'linear-gradient(135deg, rgb(34, 139, 87), rgb(139, 34, 139))'
---

## A Functional Approach to Android Architecture using Kotlin
[원본 링크](https://academy.realm.io/posts/mobilization-2017-jorge-castillo-functional-android-architecture-kotlin/)<br>
<br>
코틀린은 떠오르는 가장 강력한 함수형 언어이자,<br>
세계의 개발자들이 개인 프로페셔널 프로젝트에 사용 중이다.<br>
이번 발표에서, 조지는 우리의 안드로이드 앱을 위한 함수형 기반 아키텍쳐 구현을<br>
어떻게 할지 배울 수 있는 코틀린 접근 방식을 소개할 것 이다.<br>
그는 또한 그가 스페인 커뮤니티에서 작업하고 있는 `KATEGORY` 라고<br>
부르는 함수형 프로그래밍 라이브러리를 소개할 것이다.<br>

---

### Introduction
내이름은 조지 카스틸로 이고 GoMore 라는 덴마크 코펜하겐에 위치한 글로벌 회사에서 일하고 있습니다.<br>
GoMore는 카풀, 자동차 렌탈, 리스에 대해 다루고 있습니다.<br>

### What is Functional Programming?
관심사 분리는 함수형 프로그래밍에서 중요하다.<br>
순수성은 함수형 프로그래밍의 또 다른 일부이다.<br>
내가 함수 하나의 파라미터에 대해 1000번 호출하면,<br>
나는 매번 같은 결과를 얻게될것을 예상할 수 있다.<br>
코드는 결정가능하고 예측가능해야 한다.<br>
함수형 언어는 또한 참조 투명성과 가독성에 장점을 가지고 있다.<br>
함수의 입력 타입과 반환 값을 알수 있어야 한다.<br>
함수형 프로그래밍은 상태를 피한다.<br>

### Functional Features of Kotlin
코틀린은 다른 함수형 프로그래밍 언어들에서 찾을 수 있는 기능들을 가지고 있다.<br>
하스켈에서 고차함수들,  퍼스트 클래스 함수, 연산자 오버로딩 그리고 타입 추론 등.<br>
<br>
코틀린은 함수형 언어로 디자인 되지 않았다, 하지만 함수형 언어처럼 사용할 수 있다.<br>
<br>
`KATEGORY`의 개발 동기는 코틀린에서 함수형 프로그래밍을 할 수 있는 데이터 타입과<br>
추상들을 제공하는 라이브러리를 만드는 것이다.<br>
<br>
이 새 라이브러리를 사용하여 어떤 가능한 시스템 안에서 찾을 수 있는 <br>
핵심 문제를 해결하는 것을 해보자.<br>
<br>
자바 가상 머신에서 동작하는 언어를 사용할 때 : <br>
1) 오류와 성공의 모델링
2) 비동기 코드 그리고 스레딩
3) 사이드 이팩트
4) 디펜던시 인젝션
5) 테스팅

예를 들면, 데이터베이스 또는 API 같은 외부 소스에 쿼리할 때 성공이나 오류 결과를 모델로 만들 수 있다.<br>
우리는 객체 지향 언어들에서 매 시간 사이드 이팩트를 다룬다.<br>
그것은 상태와 매우 연관되어 있기 때문이다.<br>
객체 지향 언어들에서, 각 객체는 자신의 메모리 상태를 가지고 있어, 사이드 이팩트를 가지는 경향이 있다.<br>

### Modeling Error and Success Cases
만약 우리가 자바에서 오랫동안 동작하는 순수 프로세스를 상상해보면, 다음과 같을 것이다:
```
// Vanilla Java approach: Exceptions + callbacks
public class GetHeroesUseCase {
	public GetHeroesUseCase(HeroesDataSource dataSource, Logger logger) {
		/* …  */
}

// void : 무엇을 반환하는지 리턴 타입이 반영하지 않음
// callback : 참조 투명성이 깨짐: 에러 타입은?
public void get(int page, Callback<List<SuperHero>> callback) {
	try {
		List<SuperHero> heroes = dataSource.getHeroes(page);
		callback.onSuccess(heroes);
	} catch (IOException e) {
		//  스레드 제한 초과시 Catch + callback
		logger.log(e);
		callback.onError(“some error”);
	}
}
}
```
여기 이 코드는 히어로를 불러온다.<br>
우리는 도메인 레이어 내의 연결 혹은 메인 로직을 랩핑하는 유스케이스를 가지고 있다.<br>
유스케이스에는 히어로에 대한 페이지를 요청하고 비동기 호출이기 때문에 <br>
결과를 제공하는 콜백 메소드를 가지고 있다.<br>
우리는 데이터 소스와 패브릭을 이용하여 구현한 로거를 주입한다.<br>
<br>
그러면 우리는 이전 스레드에 유효한 히어로 리스트를 통지한다.<br>
만약 무언가 잘못 된다면, 외부 소스 시스템으로 부터 생성된 예외를 잡아내고<br>
이전 스레드에 콜백을 사용해 통지하기를 원할 것이다.<br>
아마도 당신은 또한 패브릭 로거 구현을 사용해 에러를 기록하기를 원할 것이고,<br>
이 경우 당신은 미래에 도메인 레이어 안에서 발생한 에러에 대해 무언가 단서를 가지길 원할 것이다.<br>
<br>
하지만, 이것은 이상적이지 않다. 먼저 도메인 레이어의 예외를 감쌀 필요가 있다.<br>
그리고 다른 통지 시스템에 매핑하고, 이전 스레드에 콜백을 이용해 통지 합니다.<br>
<br>
여기에서, 에러를 전파하는 두가지 방법이 있다. <br>
그리고 우리는 두 방법 간에 전환하거나 교체할 필요가 있다.<br>
예외는 스레드 제한을 뛰어넘을 수 없기 때문이다. 만약 여기서 예외를 잡지 않는다면, <br>
이 코드를 실행 또는 호출하여 다른 스레드에서 실행하는 어떤 프레젠터 또는 뷰모델이 있을때 <br>
예를 들면 `ThreadPoolexecutor`를 사용하는 경우 ,에러를 전부 잃게 될 것이다.<br>
예외가 스레드 제한을 뛰어넘을 수 없으므로, 프레젠터는 영원히 응답을 기다릴<br>
수도 있어 프레젠터에 절대 닫지 않을 것이다.<br>
<br>
우리의 참조 투명성은 깨졌다. 메소드의 반환 타입은 메소드가 무엇을 반환하는지 반영하지 못한다.<br>
우리는 콜백을 이용할 필요가 있고 콜백은 에러 타입에 대해 어떠한 것도 알려주지 않는다.<br>
그 의미는 콜백의 내부를 알고 통지의 구현을 알 필요가 있다.<br>
이 함수는 에러를 반환할 가능성이 있다. 이것은 이상적이지 않다.<br>
<br>
### Alternative 1: A Wrapper Class
이와 같은 랩퍼 클래스 안에 결과를 랩핑해보자
```
// Alternative 1 : Result wrapper(Error + Success)

// Wrapper type
public Result(ErrorType error, SuccessType success) {
	this.error  = error;
	this.success = success;
}

public enum Error {
	NETWORK_ERROR, NOT_FOUND_ERROR, UNKNOWN_ERROR
}

public class GetHeroesUseCase {
	/* … */
	// Result : 우리는 명백히 속임수를 사용하고 있다.
	// 우리는 비동기를 무시하고 있지만, 최소한 우리는 명시적인 반환 타입을 가지고 있다.
	Public Result<Error, List<SuperHero>> get(int page) {
	Result<Error, List<SuperHero>> result = dataSource.getHeroes(page);
	If (result.isError()) {
		logger.log(result.getError());
	}
	return result;
	}
}
```
이 생성자에서, 에러 또는 성공을 가질 수 있고, 헬퍼 메소드는 다른 방법으로 반응할 수 있다.<br>
우리는 아직 코틀린을 사용하지 않기 때문에 우리는 도메인 레이어 안에서 예상되는<br>
에러를 enum으로 선언하길 원한다.<br> 
우리는 데이터 소스가 히어로의 리스트 또는 에러를 리턴 가능성을 명백하게 <br>
반환 결과로 표현하도록 리팩토링 할 수 있다.<br>
<br>
이 접근법의 문제는 비동기 스레딩에서는 동작하지 않는다.<br>

### Alternative 2: RxJava
```
// Alternative 2: RxJava
public class GetHeroesUseCaseRx {	     // 스레딩은 스케쥴러를 이용해 쉽게 다룰 수 있다.

	public Single<List<SuperHero>> get() {
		// 결과 측면에서(에러와 성공) 싱글 스트림은 적절하다.
      return dataSource.getHeroes()
			    .map(this::discardNonValidHeroes)
			    .doOnError(logger::log);
	}

	private List<SuperHero> discardNonValidHeroes(List<SuperHero> superHeroes) {
		return superHeroes;
	}
}

public class HeroesNetworkDataSourceRx {
	public Single<List<SuperHero>> getHeroes() {
		return Single.create(emitter -> {
			List<SuperHero> heroes = fetchSuperHeroes();
			If (everythingIsAlright()) {
				emitter.onSuccess(heroes);
			} else if (heroesNotFound()) {
				emitter.onError(new RxErrors.NotFoundError());
			} else {
				emitter.onError(new RxErrors.UnknownError());
			}
		});
	}
}
```
다른 대안은 RxJava를 사용하는 것이다.<br>
예를 들면, 우리는 히어로들을 얻는 Marvel API를 사용하는 데이터 소스 API 구현을 가진다.<br> 
우리는 Single 을 사용하여 옵저버블을 만든다.<br>
생성된 옵저버블은 한번만 수행된다.<br>
옵저버를 구독하는 순간 히어로를 가져온다.<br>
모든 것 정상적으로 종료되었을때 알림을 만든다.<br>
그리고 에러일 때도 마찮가지이다.<br>
<br>
`onError` 단말 이벤트는 RxJava에서 에러를 전파하는 최고의 방법은 아니다.<br>
다른 시스템과 에러로 부터 복구 연산들을 가진다.<br>
`dataSource` 구현이 구독 가능한 옵저버블을 반환한다고 가정한다.<br>
<br>
유스케이스는 이제 간단합니다.<br>
이제 당신은 옵저버블에 비지니스 로직을 맵할 수 있다.<br>
예를 들면, 당신은 초상화를 가지지 않은 히어로들을 취소하고 에러들에 반응할 수 있다.<br>
모든 것은 하나의 스트림으로 랩핑된다.<br>
에러와 성공 결과 양쪽을 고려하여 포함한 데이터의 싱글 스트림을 가지고 있는 편이 더 낫다.<br>
이전 예제보다 이것이 낫다.<br>
하지만 우리는 아직 리턴 타입에 에러 타입을 포함하고 있지 않다.<br>
<br>
하지만 Maybe 타입 같은 다른 타입 사용할 수 있다. <br>
당신은 스케쥴러를 사용하여 스레드를 다룰 수 있다.<br>
이것은 RxJava Api 와 잘 통합 되어 있다.<br>
### Alternative 3: KΛTEGORY
```
// Alternative 3: Either<Error, Success>
// 도메인 에러를 지원하는 봉인 계층
sealed class CharacterError {
	object AuthenticationError : CharacterError()
	object NotFoundError : CharacterError()
	object UnknownServerError : CharacterError()
}

/* data source impl */
fun getAllHeroes(service: HeroesService): Either<CharacterError, List<SuperHero>> = 
  try {
    Right(service.getCharacters().map { SuperHero(it.id, it.name, it.thumbnailUrl, it.description) })
  } catch (e: MarvelAuthApiException) {
    Left(AuthenticationError)
  } catch (e: MarvelApiException) {
    // 예측된 도메인 에러를 외부 레이어의 예외로 변환
    If (e.httpCode == HttpURLConnection.HTTP_NOT_FOUND) {
      Left(NotFoundError)
    } else {
      Left(UnknownServerError)
  }
}

fun getHeroesUseCase(dataSource: HeroesDataSource, logger: Logger): Either<Error, List<SuperHero>> = 
	// We fold() over the Either for effects depending on the side
	dataSource.getAllHeroes().fold(
		{ logger.log(it); Left(it) },
		{ Right(it) })
```
이제 같은 문제를 KATEGORY를 사용하여 풀어 봅시다.<br>
카테고리는 Either 라고 불리는 새로운 클래스를 제공한다.<br>
`Either`는 `Maybe`와 비슷하다. 그 값은 left 혹은 right가 될 수 있다.<br>
두개의 서로 다른 값을 담고 있는 이유는 그것이 sealed class 이기 때문이다.<br>
<br>
우리는 예제를 위해 코틀린을 사용할 것이다.<br>
우리는 도메인 예외 또는 에러들의 모델을 실드 클래스로 가지고 있다.<br>
에러들을 폐쇄 계층으로 가지기 때문이며, 그리고 실드 클래스는 이들을 모델링 하기에 좋은 방법이다.<br>
<br>
우리는 새롭게 `dataSource`의 다른 구현을 가질 수 있다.<br>
우리는 함수 파라미터와 같이 디펜던시를 전달하고, 서비스를 사용하여 캐릭터를 가지고 온다.<br>
그리고 결과는 히어로의 모음 이거나 네트워크의 DTO가 될 것이다.<br>
<br>
fold 함수를 Either에 적용할 수 있기 때문에 유스케이스는 매우 간단해 보인다.<br>
이는 `dataSource`에 의해 반환 된다.<br>
fold의 의미는 두개의 다른 람다를 필요로 하며 에러 혹은 성공 값이냐에 따라 적용된다.<br>

```
// Alternative 3: Either<Error, Success>
// > 프레젠테이션 코드는 이렇게 보일 수 있다.
// 지금은 아규먼트 디펜던시에 대해서는 잊어버리자, 우리는 곧 암시적으로 전달 할 것이다.
fun getSuperHeroes(view: SuperHeroesListView, logger: Logger, dataSource: HeroesDataSource) {
	getHeroesUseCase(dataSource, logger).fold(
		{ error -> drawError(error, view) },
		{ heroes -> drawHeroes(heroes, view) })
}
private fun drawError(error:CharacterError, view: HeroesView) {
	when (error) {
		is NotFoundError -> view.showNotFoundError()
		is UnknownServerError -> view.showGenericError()
		is AuthenticationError -> view.showAuthenticationError()
	}
}
private fun drawHeroes(success: List<SuperHero>, view: SuperHeroesListView) {
	view.drawHeroes(success.map {
		RenderableHero(
			It.name,
			it.thumbnailUrl)
	})
}	// 하지만 아직, Async + Threading 은???
```
새로운 모델의 형태는 간단하다고 할 수 있습니다.<br>
이것은 또한 반환된 Either를 접을 수 있습니다.<br> 
그리고 이를 다른 렌더링 이펙트 혹은 뷰의 다른 동작에 성공/실패에 따라 적용할 수 있습니다.<br>
<br>
메소드를 보다시피 화면에 에러 화면 혹은 영웅의 리스트를 보여준다 - `drawError` 와 `drawHeroes`<br>
<br>
에러 처리를 위해 실드 클래스 안에 가지고 있는 에러타입들에 대해 매칭 하고 있다.<br>
우리는 코틀린 when 표현식과 실드 클래스가 주는 철저 평가 와 암시적 변환의 장점을 얻고 있다.<br>
에러들은 그 타입에 따라 다른 효과가 적용될 것이다.<br>
<br>
### Asynchronous Code and Threading
스레드와 동시성을 관리할 조금 다른 대안이 있다.<br>
자바 프로젝트를 상기해보자.<br>
우리는 에러를 맵핑하기 위해 ThradPoolExecutor와 예외 그리고 콜백을 사용했다.<br>
RxJava는 스케쥴러와 옵저버블, 에러 구독을 사용한다.<br>
<br>
KATEGORY 에서는 다르게 관리한다.<br>
여기는 외부 데이터 소스 계산에 적용된 것이다.<br>
예를 들면 api 클라이언트에서 api를 호출하던가, <br>
데이터 베이스에 입력 할 수 있는 데이터 소스 구현 - 이것은 외부 연산과 같다.<br>
이것은 IO 연산이다. 우리는 IO Monad로 감싸는 것이 필요하다.<br>
```
// IO<Either<CharacterError, List<SuperHero>>>
// > IO 는 연산을 감싼다. 이것은 CharacterError 나 List<SuperHero>중 하나를 반환한다.

/* network data source */
fun getAllHeros(service: HeroeService, logger: Logger): 
IO<Either<CharacterError, List<SuperHero>>> =    // 매우 명시적인 결과 타입
    runInAsyncContext (     // 우리는 코틀린 코루틴을 사용해 비동기 컨텍스트 안에서 작업을 실행한다.
        f = { queryForHeroes(service) },     // 이것은 IO로 랩핑된 연산을 반환한다.
        onError = { logger.log(it); it.toCharactorError.left() },
        onSuccess = { mapHeroes(it).right() }
        AC = IO.asyncContext()
    )
```
리턴 타입을 여기에서도 계속 사용한다.<br> 
우리는 Either를 가진다. 이것은 에러 혹은 벨리드한 히어로 리스트 일 수 있다.<br>
이것은 이제 IO로 랩핑되어 있다. IO 는 연산을 랩핑한다.<br>
이것은 스레딩 타입 이자, 에러 혹은 벨리드 히어로 리스트를 중 하나를 반환한다.<br>
동일한 타입의 안에 다른 것이 모델되어 있다.<br>
IO 연산에 어떤 것을 리턴할 수 있는 이중성을 가진다.<br>
<br>
이 유스케이스도 쉽다, 하지만 이번에는 데이터 소스로부터 돌려받는 것은 IO와 Either가 아니다.<br> 
결과적으로 맵을 두번할 필요가 있다.<br>
첫번째 맵핑은 Either를 반환하고, 유효하지 않은 히어로를 무시하기 위해 Either에 다시 맵핑을 한다.
```
// IO<Either<CharacterError, List<SuperHero>>>

/* Use case */
fun getHerosUseCase(service: HeroeService, logger: Logger): 
IO<Either<CharacterError, List<SuperHero>>> =    // 매우 명시적인 결과 타입
    getAllHeroesDataSource (service, logger).map { it.map { discardNonValidHeroes(it) } }

fun getSuperHeroes(view: SuperHeroesListView, service: HeroeService, logger: Logger) =
    getHerosUseCase(service, logger).unsafeRunAsync { it.map { maybeHeroes ->
        maybeHeroes.fold(
        { error -> drawError(error, view) },
        { success -> drawHeroes(success, view) }
        )
    }}

// >> 이팩트는 여기에서 적용한다, 유휴상태가 아니다.
```
프레젠테이션 로직은 뭔가 흥미로운 것을 한다.<br>
Either를 감싼 IO를 돌려주기 때문이다.<br>
이것은 효과를 적용하여 IO 를 해결하는 순간이다.<br>
나는 권한을 얻어, 실행을 명시적으로 알린다.<br>
그리고 언폴드, 내부 계산에서 실제 실행된다.<br>
그리고 결과는 경우에 따라 다른 효과를 적용하기 위해서 다시 폴드 된다.<br>

### Lazy Evaluation
하지만 이것은 완벽하지 않다.<br>
나는 프레젠터와 테마모델 안에서 사이드 이팩트를 적용하길 원하지 않는다.<br>
이상적으로, 프레임워크와 묶인 시스템의 끝으로 사이드 이팩트를 밀어 내는 것을 원한다.<br>
안드로이드의 경우 뷰의 구현에 해당한다.<br>
나는 사이드 이팩트를 적용할 때를 뷰가 결정하도록 두길 원한다.<br>
그러면 사이드 이팩트를 피한 나의 나머지 아키텍쳐는 완전히 순수해 질 것이다.<br>
<br>
이 문제를 해결하려면, 지연평가를 사용해야 한다.<br>
우리의 아키텍쳐의 모든 단계는 이미 계산된 값 대신 함수를 반환하도록 모델링 되어있다.<br>
이 의미는 실행되길 원하는 뭔가를 반환하지만 아직 실행되지 않았다.<br>
이것은 함수형 프로그래밍과 강력하게 관련되어 있다.<br>

### Dependency Injection
여러 계층을 거쳐 디팬던시를 전달하는 것은 이상적이지 않으므로, 피해야 한다.<br>

#### Reader Monad
우리는 리더 모나드 라고 불리는 것을 솔루션으로 사용할 수 있다.<br>
리더 모나드는 연산을 감싸 지연하는 것을 한다.<br> 
리더는 컨텐스트를 가지며, 디펜던시들은 암시적으로 전달된다.<br>
<br>
리더 모나드는 약점을 가지고 있다.<br>
함수실행 지연 계산 실행을 지연시키는 문제를 해결하거나 암시적으로 디펜던시를 전달하더라도,<br>
함수형 프로그래밍에서 계산된 반환값을 이르는 모나딕 값을 다루는 것을 할 수 없다.<br>
그것은 단지 Either, IO 일 뿐이다.<br>

```
역주 : 모나딕 값이란?
"모나드와 관계된" 이라는 의미로, "모나딕 값"은 모나드 타입 인스턴스를 타입으로 가지는 값을 의미한다.
위의 글의 의미는 리더로 감싼 계산은 컨텍스트를 가진 모나드 타입에 의존할 뿐이지 구체적인
값을 직접 다루지 않는다는 의미이다.
```

#### ReaderT
결과적으로 Kleisli가 존재한다. Kleisli는 그저 향상된 Reader 이다.<br> 
이것은 자주 “Reader Transform It”  또는 타입 알리아스를 사용해 ReaderT 로 불린다.<br>
```
// ReaderT<IdHK, D, IO<Either<CharacterError, List<SuperHero>>>>
// 우리는 여기에서 부터 타입을 조금씩 죽이기 시작한다, 우리는 해결책을 찾을 것이다.
/* data source could look like this */
// 암시적 디펜던시가 더이상 필요 없다.
fun getHeroes() :
ReaderT<IdHK, D, IO<Either<CharacterError, List<SuperHero>>>> = 
    // Reader.ask() 는 Reader { D -> D }를 승급한다, 그래서 맵핑할 때 우리는 D에 접근 할 수 있다.
    Reader.ask<GetHeroesContext>().map({ ctx ->
        runInAsyncContext(
            f = { ctx.apiClient.getHeroes()  },
            onError = { it.toCharacterError().left() },
            onSuccess = { it.right() },
            AC = ctx.threading
        )
   })

```
이전과 같이 Reader로 감쌌다.<br> 
이제 우리는 에러 혹은 영웅들의 리스트 둘중 하나를 가질 수 있는 지연된 IO 연산을 가진다.<br>
여기의 리턴 타입은 우리가 정확히 원하는 데로이다.<br> 
Reader로 감싸진 연산이고, 더이상 디펜던시들을 명시적으로 전달할 필요가 없기 때문에<br>
우리는 자동으로 접근할 수 있다.<br>
이것은 ask 라고 불리는 함수로 인해 할 수 있는 것이다.<br>
그것은 자동으로 타입 연산을 감싼다.<br>
```
// ReaderT<IdHK, D, IO<Either<CharacterError, List<SuperHero>>>>

/* use case */
fun getHeroUseCase() = fetchAllHeroes().map { io ->
    io.map { maybeHeroes ->
        maybeHeroes.map { discardNonValidHeroes(it) }
    }
}

/* presenter code */
fun getSuperHeroes() = Reader.ask<GetHeroesContext>().flatMap (
{ (_, view: SuperHeroesListView) -> // Context 디스트럭쳐링
    getHeroUseCase().map ({ io ->
        io.unsafeRunAsync { it.map { maybeHeroes ->
            maybeHeroes.fold(
                { error -> drawError(error, view)  },
                { success -> drawHeroes(view, success) }
            )
        }
    })
})
```
프레젠터는 이처럼 생겼다.<br> 
우리는 또다시 컨텍스트 값로 부터의 Reader를 ask로 승급 하여 flatMap을 넘어 컨텍스트에 엑세스 할 수 있다.<br>
괄호의 내부는 컨텍스트 자신이다.<br> 
하지만 나는 데이터 클래스 디컨스트럭션을 적용하였다.<br>
이는 언어의 기능으로 빠르게 클래스 자신의 프로퍼티에 접근할 수 있게 해준다.<br>

```
// ReaderT<IdHK, D, IO<Either<CharacterError, List<SuperHero>>>>

// > ReaderT 덕분에 전체 연산 트리가 지연되었다.
// > 이것은 완전 순수하다. 이팩트가 아직 실행 되지 않으므로
// > 이팩트들이 수행되는 순간, 당신은 간단하게 당신이 원하는 곳에 컨텍스트를 전달하여 실행할 수 있다.

/* 우리는 안전하지 않은 이팩트를 뷰의 구현에서 수행하고 있다. */
Override fun onResume() {
    /* presenter call */
    getSuperHeroes().run(heroesContext)
}
//> 테스트 시나리오 상에서, 목킹한 가짜 디펜던시로 부터 제공받은 콘텍스트를 전달하면 된다.
```

테스트를 할 때, 디펜던시들을 목킹한 다른 컨텍스트를 전달한다.<br>
예를 들면 컨텍스트를 데거 그래프처럼 쉽게 생각 할 수 있다.<br>
그것은 모든 제공된 디펜던시들의 데펜던시 바인딩들이나 디펜턴시 트리가 될 수 있다.<br>

#### Monad Transformers
우리가 이전에 가진 모든 중첩 타입을 제거할 필요가 있다.<br> 
당신의 스택의 다른 타입의 모나드를 가질때 모나드는 매우 우아하게 구성되지 않는다.<br>
모나드 트랜스포머를 사용하는 것은 이미 존재하는 모나드에 새로운 힘을 주는 방법이다.<br>
만약 IO를 가진다면, 나는 IO에 Either의 힘을 선물하고 동작하게 만들수 있다.<br>
<br>
우리는 IO를 가지고 Either와 Readers의 힘을 가지길 원한다<br> 
그래서 끝에 우리의 스택이 가지고 있는 모든 관심사를 해결할 수 있는 슈퍼 타입을 가질 것이다.<br> 
우리는 이것은 AsyncResult라고 부를 수 있다.<br>

```
// AsyncResult<D, A>
//> 에러 핸들링, 비동기, IO 연산, 디펜던시 인젝션에 주의

/* data source */
fun <D : SuperHeroesContext> fetchAllHeroes(): AsyncResult<D, List<SuperHero>> = 
	AsyncResult.monadError<D>().binding {      // 바인딩은 모나드 표현식의 일부이다.
		val query = buildFetchHeroesQuery() // async 호출을 그들이 sync 인것 처럼 
		val ctx = AsyncResult.ask<D>().bind()// 순차적으로 코딩
		runInAsyncContext(
			f = { fetchHeroes(ctx, query) },
      onErroor = { liftError<D>(it) },
      onSuccess = { liftSuccess(it) },
      AC = ctx.threading<D>()
		).bind()
	}

//> 모나드 바인딩은 이미 lift 하고 flatMap 된 모나드 컨텍스트 결과를 반환한다.
```

이것은 모든 우리의 관심들을 하나의 타입으로 감싼다.<br>
그리고 3개의 다른 중첩 레벨이 아니다.<br>
우리는 첫번째로 코틀린에서 KATEGORY를 사용한 모나드 바인딩 제공한다.<br>
이것은 하스켈에서의 do 표현식과 같다.<br>
<br>
나는 이제 더 많은 비동기 호출을 필요한 만큼 수행할 수 있다. <br>
이것은 순차적으로 완료할 수 있다.<br>
```
// AsyncResult<D, A>
/* use case */
fun <D : SuperHeroesContext> getHeroesUseCase(): AsyncResult<D, List<CharacterDtoo>> =
    fetchAllHeroes<D>().map { discardNonValidHerooes(it) }

/* presenter */
fun getSuperHeroes(): AsyncResult<GetHeroesContext, Unit> =
    getHeroesUseCase<GetHeroesContext>()
        .map { heroesToRenderableMoodels(it) }
        .flatMap { drawHeroes(it) }
        .handleErrorWith { displayErrors(it) }

 /* view impl */
override fun onResume() {
    getSuperHeroes().unsafePerformEffects(heroesContext)
}
//> 테스트 시나리오로 돌아와서, 목킹이 필요한 가짜 디펜던시들을 가진 다른 컨텍스트를 전달하면 된다.
```
프레젠테이션 코드 변경할 필요가 없다, 그리고 이것이 더 간결해 보인다.

### Advanced Functional Programming Styles

#### Tagless Final
Tagless Final은 모든 실행 체인 그리고 IO, Reader나 Either 같은 구체 클래스를 사용하지 않지만<br>
타입 클래스들을 사용한 정의에 대한 것이다. 타입 클래스는 행위를 정의 한다.<br>
만약 우리가 행위에 기반한 실행 트리를 정의한다면, 
우리는 결정될 시점에 구현 상세를 나중에 전달 할 수 있다.<br>
우리의 전체 실행 트리는 구체적인것이 아닌 추상에 의존하여 정의 된다.<br>
리포지토리에 문제를 매우 잘 설명하는 [예제와 풀리퀘스트](https://github.com/JorgeCastilloPrz/ArrowAndroidSamples/pull/2)가 있다.<br>

#### Free Monads
프리 모나드 유사한 컨셉이다.<br>
Free 는 연산을 감싸는 한 방법이자 완전히 추상적인 정의이다.<br>
우리는 중첩 연산들인 추상 구문 트리를 Free를 적용하여 합성한다.<br> 
그래서 각 레벨의 하나에서 우리는 연산을 담고 있는 Free를 반환한다.<br>
결과적으로, 우리는 의미나 구현의 상세를 마지막 순간까지 선언하지 않는다.<br>
<br>
Free는 쉽게 디펜던시 인젝션을 대체할 수 있다.<br>
우리는 구현의 상세를 마지막 순간까지 지연시키기 때문이다.<br>
그러므로 디펜던시 인젝션은 최근에는 필요치 않다.<br>
Free 인스턴스로 구성된 것을 테스트하려면 다른 모든 인터프리터<br>
또는 원하는 것을 모의하는 것과 같은 다른 구현 세부 정보를 전달하는 다른 인터프리터를 사용해야 한다.<br>
동일한 안드로이드 앱을 Free 모나드 스타일을 사용하여 구현한 [샘플 리포지토리](https://github.com/JorgeCastilloPrz/ArrowAndroidSamples/pull/6)가 있다.<br>

### Conclusions
우리가 오늘 배운 패턴들은 어떤 플랫폼이나 시스템에도 적용할 수 있다<br>
디펜던시 인젝션과 같은 완전히 일반적인 문제들을 해결했기 때문이다.<br>
<br>
백엔드 또는 프론트앤드 또는 어떤 것이든 코틀린을 사용한다면 완전히 똑같은 패턴을 가질 수 있다.<br>
코틀린을 사용하지 않는다고 해도, 당신이 스칼라 또는 하스켈 여전히 같은 컨셉을 적용할 수 있다.<br>
그들은 완전히 일반적인 문제들을 위한 랩핑 솔루션이게 때문이다.<br>
<br>
함수형 프로그래밍은 한번 수정한 것에 대해, 다시는 하지 않는 것이다.<br>
<br>
모든 샘플 코드는 샘플 리포에서 볼수 있다.<br>
동일한 어플리케이션 구현을 [각각 다른 4가지 접근법으로 구현](https://github.com/JorgeCastilloPrz/ArrowAndroidSamples)한 것을 볼 수 있다.<br>

