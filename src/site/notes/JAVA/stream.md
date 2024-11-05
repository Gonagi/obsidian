---
{"dg-publish":true,"permalink":"/JAVA/stream/","created":"2024-10-15T19:33:52.538+09:00"}
---

# java.util.stream
- 함수형 프로그래밍 스타일을 사용해 컬렉션(배열, 리스트 등)의 요소들을 편리하게 처리할 수 있는 기능
- Java 8에서 도입
### 컬렉션과의 차이점
1. 저장공간X : 자료 구조가 아니다, 연산을 수행하는 통로, 혹은 파이프라인의 역할을 한다.
2. 함수적 특징 : 원본을 수정하지 않고 결과만 생성한다.
3. 지연 연산을 한다 : 중간 연산과 최종 연산으로 나눠지지만 최종연산에서만 실제 처리가 이루어 진다.
4. 고정된 크기를 가지지 않는다 : 유한/무한한 크기를 가질 수 있다.
5. 소모품 : Stream은 사용할때마다 생성해야 한다. 사용하면 사라짐
### Stream 작업 및 파이프라인
- 중간, 최종 연산으로 나눠지며, 이들이 결합하여 stream pipeline을 만든다.
- #### 중간연산
	- 새로운 스트림을 생성한다.
	- 항상 지연 연산을 한다 (lazy)
	- 종단 연산이 실행되기 전까지 실행되지 않는다.
	- `.filter`, `.map` 등이 있다.
	- ##### stateless operation
		- `.filter`, `.map`와 같은 무상태 연산자는 새 요소를 만드는 작업을 진행할 때 이전 요소의 상태를 유지하지  않는다.
		- 독립적으로 작동한다.
		- 최소한의 데이터 버퍼링으로 순차적이든 병렬적이든 한번의 순회로도 가능하다.
	- ##### stateful operation
		- `distinct`, `sorted`와 같은 유상태 연산자는 새 요소에 접근할 때 이전 요소의 상태를 유지해야 한다.
		- 데이터 전체를 순회한 후 결과를 내야해서 병렬처리에서 주의해야 한다.
		- 데이터를 여러번 순회해야 할 수도 있어서 리소스가 많이 소모되고 효율성도 저하될 수 있다.
- #### 종단연산
	- 종단 연산은 Stream을 순회한 후에 결과나 side-effect를 생성한다.
	- 종단 연산이 수행된 이후에는 해당 `Stream`은 더이상 사용할 수 없다.
	- `Stream`의모든 데이터를 순회하면서 파이프라인에 정의된 모든 연산을 수행하고 결과를 반환한다.(eager)
	- `.forEach`, `.reduce`
	- 예외적으로`iterator()`, `spliterator()`은 lazy하다.
		- 스트림에서 제공하는 기본적인 연산들이 충분하지 않을 때, ==직접 순회나 처리를 할 수 있는 제어수단(escape hatch)==으로 사용한다.
- #### Stream의 이점
	``` java
	widgets.stream()
			.filter(a -> a.getColr() == RED)
			.mapToInt(b -> b.getWeight())
			.sum();
	```
	- 다음과 같이 `filter - map - sum`연산 3개가 데이터를 ==한번만 순회하면서== 필요한 모든 연산을 할 수 있다.
	- 다음 방식으로 중간 상태를 최소화하고 연산의 효율성을 높일 수 있다.
- #### 일부 연산은 단축 연산으로 간주된다.
	- 무한한 입력 `Stream`에 대해서도 `limit`과 같은 ==중간 연산==을 통해 유한한 `Stream`으로 결과를 생성할 수 있다.
	- 무한한 입력`Stream`에 대해서도 `anyMatch`와 같은 ==종단 연산==을 통해 유한한 시간내에 종료될 수 있다.
	- 단축 연산만으로 `Stream`을 유한한 시간 내에 처리해 줄 수 없기 때문에 연산의 복잡성, 데이터의 특성, 파이프라인의 다른 연산들도 고려해야 한다.
---
### 병렬처리
- `Stream`은 집계 연산(Aggregate Function)의 파이프라인으로 재구성하여 병렬처리를 용이하게 해준다.
- 모든 `Stream` 연산은 직렬 또는 병렬적으로 실행할 수 있다.
	- 병렬을 요청하지 않으면 직렬로 실행된다.
	- `Collection.stream()`, `Collection.parllelStream()`
	- `InStream.range(int, int)`, `BaseStream.parallel()`
	``` java
	int sumOfWeights = widgets.parallelStream()
							   .filter(b -> b.getColor() == RED)
							   .mapToInt(b -> b.getWeight())
							   .sum();
	```
	- `Stream`의 순차, 병렬 모드는 `BaseStream.isParallel()`메서드로 확인할 수 있으며, `BaseStream.sequential()`, `BaseStream.parallel()` 연산으로 `Stream`의 모드를 변환시킬 수 있다. (두 연산이 같이 있으면 마지막 연산을 따라간다.)
	- `findAny()`와 같이 비결정적인 작업을 제외하고, 순차적이든 병렬적이든 그 결과는 동일하다.
	- `Stream`내 연산자의 매개변수로 람다 식이 자주 사용되는데 ==비간접적, 무상태==여야만 한다..
		- 매개변수로 Function과 같은 함수형 인터페이스의 인스턴스이며, 람다 표현식이나 메서드 참조인 경우가 많다.
### 비간접적
- `Stream`은 `non-thread-safe`한 컬렉션인 `ArrayList`도 집계 연산을 병렬적으로 처리할 수 있게 해준다.
- 집계 연산의 병렬적 처리는 `Stream Pipeline`을 실행하는 동안 데이터 소스와의 간섭을 방지할 수 있는 경우에만 가능하다.
- `iterator()`, `spliterator()`을 제외하고, 종단연산이 실행될 때 연산이 실행되며. 종단 연산이 끝날 때 `Stream`의 실행 끝난다.
- 간섭을 방지한다는 것은 `Stream Pipeline`이 실행되는 동안 데이터가 전혀 수정되지 않는 것을 뜻한다.
- 소스가 변경되면 안되는 일반적인 `Stream`와 달리 ==동시성 컬렉션(`Concurrent collections`)을 소스로 사용하는 `Stream`은 여러 스레드가 동시에 데이터를 수정할 수 있는 환경에서도 안전하게 작동==할 수 있도록 설계되었다.
---
[공식문서, Package java.util.stream](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/package-summary.html)
[wlsdl00 (2023.11) Stream이란? with 공식문서](https://gaebalog.tistory.com/entry/Java-Stream%EC%9D%B4%EB%9E%80)