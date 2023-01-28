# Chapter 7 - 병렬 데이터 처리와 성능
외부 반복 -> 내부 반복으로 바꾸면 네이티브 자바 라이브러리가 스트림 요소의 처리를 제어할 수 있습니다.  
컴퓨터의 멀티코어를 활용해서 파이프라인 연산을 실행할 수 있다는 점이 가장 중요한 특징입니다.  
  
예를 들어 Java 7이 등장하기 전에는 데이터 컬렉션을 병렬로 처리하기가 어려웠습니다.과정은 다음과 같습니다.  
1. 우선 데이터를 서브파트로 분할  
2. 분할된 서브파트를 각각의 스레드로 할당  
3. 스레드로 할당한 다음에는 의도치 않은 레이스 컨디션이 발생하지 않도록 적절한 동기화  
4. 마지막으로 부분 결과를 합침  
Java 7은 더 쉽게 병렬화를 수행하면서 에러를 최소화할 수 있도록 `포크/조인 프레임워크` 기능을 제공합니다.  

여러 청크(정보 조각)를 병렬로 처리하기 전에 병렬 스트림이 요소를 여러 청크로 분할하는 방법을 설명합니다.  
커스텀 Spliterator를 직접 구현하면서 분할 과정을 우리가 원하는 방식으로 제어하는 방법도 설명합니다.    
  
## 7.1 병렬 스트림
컬렉션에 parallelStream을 호출하면 병렬 스트림이 생성됩니다.  
**병렬 스트림이란 각각의 스레드에서 처리할 수 있도록 스트림 요소를 여러 청크로 분할한 스트림입니다.**  
따라서 병렬 스트림을 이용하면 모든 멀티코어 프로세서가 각각의 청크를 처리하도록 할당할 수 있습니다.  
  
숫자 n을 인수로 받아서 1부터 n까지의 모든 숫자의 합계를 반환하는 메서드를 구현한다고 가정해봅니다.  
무한 스트림을 만든 다음에 인수로 주어진 크기로 스트림을 제한하고, 두 숫자를 더하는 BinaryOperator로 리듀싱 작업을 수행할 수 있습니다.  
```java
public static long sequentialSum(long n) {
		return Stream.iterate(1L, i -> i + 1)
			.limit(n)
			.reduce(0L, Long::sum);
	}
```
전통적인 자바에서는 다음과 같이 반복문으로 이를 구현할 수 있습니다.  
```java
public long iterativeSum(long n) {
		long result = 0;
		for (long i = 1L; i <= n; ++i) {
			result += i;
		}
		return result;
	}
```
특히 n이 커진다면 이 연산을 병렬로 처리하는 것이 좋을 것입니다.  
결과 변수는 어떻게 동기화해야 할까? 몇 개의 스레드를 사용해야 할까? 숫자는 어떻게 생성할까? 생성된 숫자는 누가 더할까?  
  
병렬 스트림을 이용하면 걱정, 근심 없이 모든 문제를 쉽게 해결할 수 있습니다.  
  
### 7.1.1 순차 스트림을 병렬 스트림으로 변환하기
순차 스트림에 parallel 메서드를 호출하면 기존의 함수형 리듀싱 연산(숫자 합계 계산)이 병렬로 처리됩니다.  
```java
	public long parallelSum(long n) {
		return Stream.iterate(1L, i -> i + 1)
			.limit(n)
			.parallel() // 병렬 스트림으로 변환 
			.reduce(0L, Long::sum);
	}
```
이전 코드와 다른 점은 스트림이 여러 청크로 분할되어 있다는 것입니다.  
리듀싱 연산을 여러 청크에 병렬로 수행할 수 있습니다. 마지막으로 리듀싱 연산으로 생성된 부분 결과를 다시 리듀싱 연산으로 합쳐서 전체 스트림의 리듀싱 결과를 도출합니다.  
  
사실 순차 스트림에 parallel을 호출해도 스트림 자체에는 아무 변화도 일어나지 않습니다.  
내부적으로 이후 연산이 병렬로 수행해야 함을 의미하는 불리언 플래그가 설정됩니다.  
반대로 sequential로 병렬 스트림을 순차 스트림으로 바꿀 수 있습니다.  
  
예를 들어 다음과 같은 코드에서
```java
stream.parallel()
      .filter(...)
      .sequential()
      .map(...)
      .parallel()
      .reduce();
```
parallel과 sequential 중 최종적으로 호출된 메서드가 전체 파이프라인에 영향을 미칩니다. 위의 코드는 전체적으로 병렬로 실행됩니다.  
  
병렬 스트림은 내부적으로 ForkJoinPool 을 사용합니다.  
기본적으로 ForkJoinPool은 프로세서 수, 즉 `Runtime.getRuntime().availableProcessors()`가 반환하는 값에 상응하는 스레드를 갖습니다.  
  
`System.getProperty("java.util.concurrent.ForkJoinPool.common.parallelism", "12");`
이 코드는 전역 설정 코드입니다. 이후의 모든 병렬 스트림 연산에 영향을 줍니다.  
현재는 하나의 병렬 스트림에 사용할 수 있는 특정한 값을 지정할 수 없습니다.  
  
### 7.1.2 스트림 성능 측정  
벤치마크는 컴퓨팅에서 특정 오브젝트에 대해 일반적으로 수많은 표준 테스트와 시도를 수행함으로써 오브젝트의 상대적인 성능 측정을 목적으로 컴퓨터 프로그램을 실행하는 행위입니다.    
성능을 최적화할 때는 `세 가지 황금 규칙`을 기억해야 합니다.  
1. 측정  
2. 측정  
3. 측정!!  
따라서 `자바 마이크로벤치마크 하니스(JMH)`라는 라이브러리를 이용해 작은 벤치마크를 구현할 것입니다.  
JMH를 이용하면 간단하고, 어노테이션 기반 방식을 지원하며, 안정적으로 자바 프로그램이나 자바 가상 머신(JVM)을 대상으로 하는 다른 언어용 벤치마크를 구현할 수 있습니다.  
  
사실 JVM으로 실행되는 프로그램을 벤치마크하는 작업은 쉽지 않습니다.  
1. 핫스팟이 바이트코드를 최적화하는데 필요한 준비 시간  
2. 가비지 컬렉터로 인한 오버헤드  
등과 같은 여러 많은 요소를 고려해야 하기 때문입니다.  
  
JMH로 벤치마크 성능 테스트를 돌려본 결과입니다.  
```java
Benchmark                              Mode  Cnt   Score    Error  Units
ParallelStreamBenchmark.iteraviteSum   avgt    4   3.711 ±  0.771  ms/op 
ParallelStreamBenchmark.parallelSum    avgt    4  71.493 ± 10.724  ms/op
ParallelStreamBenchmark.sequentialSum  avgt    4  63.634 ±  2.426  ms/op
```
전통적인 for 루프를 사용해 반복하는 방법이 더 저수준으로 동작할 뿐 아니라 특히 기본값을 박싱하거나 언박싱할 필요가 없으므로 더 빠를 것이라 예상할 수 있습니다.  
순차적 스트림에 비해 약 17 배 빠른 것으로 나타납니다.  
  
병렬 스트림을 사용하는 버전은 순차 버전에 비해서도 느린 것으로 나타났습니다. 두 가지 문제를 발견할 수 있습니다.  
- 반복 결과로 박싱된 객체가 만들어지므로 숫자를 더하려면 언박싱을 해야 합니다.  
- 반복 작업은 병렬로 수행할 수 있는 `독립 단위로 나누기`가 어렵습니다.  
이전 연산의 결과에 따라 다음 함수의 입력이 달라지기 때문에 iterate 연산을 청크로 분할하기가 어렵습니다.  
  
이와 같은 상황에서는 병렬 리듀싱 연산이 수행되지 않습니다. 리듀싱 과정을 시작하는 시점에 전체 숫자 리스트가 준비되지 않았으므로 스트림을  
병렬로 처리할 수 있도록 청크로 분할할 수 없습니다.  
  
결국 스레드를 할당하는 오버헤드만 증가하게 됩니다.  
  
### 더 특화된 메서드 사용
LongStream.rangeClosed 는 iterate에 비해 다음과 같은 장점을 제공합니다.  
- 기본형 long을 직접 사용하므로 박싱과 언박싱 오버헤드가 사라집니다.  
- 쉽게 청크로 분할할 수 있는 숫자 범위를 생산합니다. 예를 들어 1-20 범위의 숫자를 각각 1-5, 6-10, 11-15, 16-20 범위의 숫자로 분할할 수 있습니다.  
```java
@Benchmark
	public long rangedSum() {
		return LongStream.rangeClosed(1, REPEAT_NUMBER)
			.reduce(0L, Long::sum);
	}
```
벤치마크 결과는 다음과 같습니다.  
```java
ParallelStreamBenchmark.rangedSum      avgt    4   3.994 ±  0.213  ms/op
```
기존 iterate 팩토리 메서드로 생성한 순차 버전에 비해 이 예제의 숫자 스트림 처리 속도가 더 빠릅니다.  
**특화되지 않은 스트림을 처리할 때는 오토박싱, 언박싱 등의 오버헤드를 수반하기 때문입니다.**  
상황에 따라서는 어떤 알고리즘을 병렬화하는 것보다 적절한 자료구조를 선택하는 것이 더 중요하다는 사실을 단적으로 보여줍니다.  
병렬 스트림을 적용하면 다음과 같습니다.  
```java
@Benchmark
	public long parallelRangedSum() {
		return LongStream.rangeClosed(1 ,REPEAT_NUMBER)
			.parallel()
			.reduce(0L, Long::sum);
	}
ParallelStreamBenchmark.parallelRangedSum  avgt    4   7.076 ± 26.950  ms/op
```
실질적으로 리듀싱 연산이 병렬로 수행됩니다.  
병렬화를 이용하려면 스트림을 재귀적으로 분할해야 하고, 각 서브스트림을 서로 다른 스레드의 리듀싱 연산으로 할당하고, 이들 결과를 하나의 값으로 합쳐야 합니다.  
따라서 **코어 간에 데이터 전송 시간보다 훨씬 오래 걸리는 작업만 병렬로 다른 코어에서 수행하는 것**이 바람직합니다.  
  
### 7.1.3 병렬 스트림의 올바른 사용법
병렬 스트림을 잘못 사용하면서 발생하는 많은 문제는 **공유된 상태를 바꾸는 알고리즘을 사용**하기 때문에 일어납니다.  
```java
public static long sideEffectSum(long n) {
		Accumulator accumulator = new Accumulator();
		LongStream.rangeClosed(1, n).forEach(accumulator::add);
		return accumulator.total;
	}
	
	public static class Accumulator {
		private long total = 0;
		
		public void add(long value) {
			total += value;
		}
	}
```
위 코드는 본질적으로 순차 실행할 수 있도록 구현되어 있으므로 병렬로 실행하면 참사가 일어납니다.  
특히 total을 접근할 때마다(다수의 스레드에서 동시에 데이터에 접근하는) 데이터 레이스 문제가 일어납니다.  
동기화로 문제를 해결하다보면 결국 병렬화라는 특성이 없어져 버릴 것입니다.  
```java
public static long sideEffectParallelSum(long n) {
		Accumulator accumulator = new Accumulator();
		LongStream.rangeClosed(1, n).parallel().forEach(accumulator::add);
		return accumulator.total;
	}
```
실행 결과를 출력하면 올바른 결과값이 나오지 않습니다.  
여러 스레드에서 누적자, 즉 total += value를 실행하면서 이런 문제가 발생했습니다. total += value는 아토믹 연산이 아닙니다.  
결국 여러 스레드에서 공유하는 객체의 상태를 바꾸는 블록 내부에서 add 메서드를 호출하면서 이 같은 문제가 발생합니다.  
  
이 예제처럼 병렬 스트림을 사용했을 때 이상한 결과에 당황하지 않으려면 상태 공유에 따른 부작용을 피해야 합니다.  
  
`공유된 가변 상태를 피해야 한다` 는 사실을 기억해야 합니다.(병렬 스트림이 올바르게 동작하려면)  
### 7.1.4 병렬 스트림 효과적으로 사용하기
양을 기준으로 병렬 스트림 사용을 결정하는 것은 바람직하지 않습니다.  
- 확신이 서지 않으면 직접 측정하라. 적절한 벤치마크로 직접 성능을 측정하는것이 바람직하다.  
- 박싱을 주의하라. 자동 박싱과 언박싱은 성능을 크게 저하시킬 수 있는 요소이다. IntStream, LongStream, DoubleStream 와 같은 기본형 특화 스트림 이용  
- 순차 스트림보다 병렬 스트림에서 성능이 떨어지는 연산이 있다. limit이나 findFirst처럼 요소의 순서에 의존하는 연산을 병렬 스트림에서 수행하려면 비싼 비용을 치러야 한다.  
- 스트림에서 수행하는 전체 파이프라인 연산 비용을 고려하라. 처리해야 할 요소 수(N) * 하나의 요소를 처리하는데 드는 비용(Q). Q가 높아질수록 병렬 스트림으로 성능 개선할 가능성이 높아진다.  
- 소량의 데이터에서는 병렬 스트림이 도움이 되지 않는다.  
- 스트림을 구성하는 자료구조가 적절한지 확인하라. 예를 들어 ArrayList를 LinkedList 보다 효율적으로 분할할 수 있다. LinkedList를 분할하려면 모든 요소를 탐색해야 함. 
또한 range 팩토리 메서드로 만든 기본형 스트림도 쉽게 분해할 수 있다. 마지막으로 커스텀 Spliterator를 구현해서 분해 과정을 완벽하게 제어할 수 있다.  
- 스트림의 특성과 파이프라인의 중간 연산이 스트림의 특성을 어떻게 바꾸는지에 따라 분해 과정의 성능이 달라질 수 있다.  
SIZED 스트림(크기가 알려진 소스) 는 정확히 같은 크기의 두 스트림으로 분할할 수 있으므로 효과적으로 스트림을 병렬 처리할 수 있다.  
필터 연산이 있으면 스트림의 길이를 예측할 수 없으므로 효과적으로 스트림을 병렬 처리할 수 있을지 알 수 없게 된다.    
- 최종 연산의 병합 과정(예를 들어 Collector의 combiner 메서드) 비용을 살펴보라. 병합 과정의 비용이 비싸다면 병렬 스트림으로 얻은 성능의 이익이 서브스트림의 부분 결과를 합치는 과정에서 상쇄될 수 있다.  

## 7.2 포크/조인 프레임워크
포크/조인 프레임워크는 병렬화할 수 있는 작업을 재귀적으로 작은 작업으로 분할한 다음에 서브태스크 각각의 결과를 합쳐서 전체 결과를 만들도록 설계되었습니다.  
포크/조인 프레임워크에서는 서브태스크를 스레드 풀(ForkJoinPool)의 작업자 스레드에 분산 할당하는 ExecutorService 인터페이스를 구현합니다.  
  
### 스레드 풀 개념
- 스레드 풀은 작업 처리에 사용되는 스레드를 제한된 개수만큼 정해 놓고 작업 큐에 들어오는 작업들을 하나씩 스레드가 맡아 처리합니다.  
- 작업 처리가 끝난 스레드는 다시 작업 큐에서 새로운 작업을 가져와 처리합니다.  
- 작업 처리 요청이 폭증해도 작업 큐라는 곳에 작업이 대기하다가 여유가 있는 스레드가 그것을 처리하므로 스레드의 전체 개수는 일정하며 애플리케이션의 성능도 저하되지 않습니다.  
  
### 7.2.1 RecursiveTask 활용 
스레드 풀을 이용하려면 `RecursiveTask<R>`의 서브클래스를 만들어야 합니다.  
여기서 R은 병렬화된 태스크가 생성하는 결과 형식입니다. 결과가 없을 때는 `RecursiveAction` 형식입니다. (결과가 없더라도 다른 비지역 구조를 바꿀 수 있습니다)  
`RecursiveTask`를 정의하려면 추상 메서드 compute 를 구현해야 합니다.  
```java
protected abstract R compute();
```
compute 메서드는 태스크를 서브태스크로 분할하는 로직과  
더 이상 분할할 수 없을 때 개별 서브태스크의 결과를 생산할 알고리즘을 정의합니다.  
  
따라서 대부분의 compute 메서드 구현은 다음과 같은 의사코드 형식을 유지합니다.  
```java
if (태스크가 충분히 작거나 더 이상 분할할 수 없으면) {
	순차적으로 태스크 계산
} else {
	태스크를 두 서브태스크로 분할
	태스크가 다시 서브태스크로 분할되도록 이 메서드를 재귀적으로 호출함
	모든 서브태스크의 연산이 완료될 때까지 기다림
	각 서브태스크의 결과를 합침 
}
```
이 알고리즘은 분할 후 정복 알고리즘의 병렬화 버전입니다.  
분할 정복 알고리즘은 그대로 해결할 수 없는 문제를 작은 문제로 분할하여 문제를 해결하는 방법이나 알고리즘입니다.  
  
`ForkJoinSumCalculator`  
```java
public class ForkJoinSumCalculator extends RecursiveTask<Long> {

	private static final long THRESHOLD = 10_000; // 이 값 이하의 서브태스크는 더 이상 분할할 수 없다.
	private final long[] numbers;
	private final int start; // 이 서브태크스에서 처리할 배열의 초기 위치
	private final int end; // 이 서브태스크에서 처리할 배열의 최종 위치

	public ForkJoinSumCalculator(long[] numbers) { // 메인 태스크를 생성할 때 사용할 공개 생성자
		this(numbers, 0, numbers.length);
	}

	private ForkJoinSumCalculator(long[] numbers, int start, int end) { // 메인 태스크의 서브태스크를 재귀적으로 만들 때 사용할 비공개 생성자
		this.numbers = numbers;
		this.start = start;
		this.end = end;
	}

	@Override
	protected Long compute() {
		int length = end - start;
		if (length <= THRESHOLD) { // 기준값과 같거나 작으면 순차적으로 결과를 계산한다.
			return computeSequentially();
		}

		ForkJoinSumCalculator leftTask = new ForkJoinSumCalculator(numbers, start, start + length / 2);
		leftTask.fork();

		ForkJoinSumCalculator rightTask = new ForkJoinSumCalculator(numbers, start + length / 2, end);

		Long rightResult = rightTask.compute();
		Long leftResult = leftTask.join();
		return leftResult + rightResult;
	}

	private long computeSequentially() {
		long sum = 0;
		for (int i = start; i < end; ++i) {
			sum += numbers[i];
		}
		return sum;
	}

	public long computeSequentiallyWithStream() {
		return Arrays.stream(numbers)
			.sum();
	}
}
```
위 메서드는 n까지의 자연수 덧셈 작업을 병렬로 수행하는 방법을 더 직관적으로 보여줍니다.  
다음 코드처럼 ForkJoinSumcalculator의 생성자로 원하는 수의 배열을 넘겨줄 수 있습니다.  
```java
public static long forkJoinSum(long n) {
		// n까지의 자연수 덧셈 작업을 병렬로 수행하는 방법
		long[] numbers = LongStream.rangeClosed(1, n).toArray();
		ForkJoinTask<Long> task = new ForkJoinSumCalculator(numbers);
		return new ForkJoinPool().invoke(task);
	}
```
LongStream으로 n까지의 자연수를 포함하는 배열을 생성했습니다.  
그리고 생성된 배열을 ForkJoinSumCalculator의 생성자로 전달해서 ForkJoinTask를 만들었습니다.  
마지막으로 생성한 태스크를 새로운 ForkJoinPool의 invoke 메서드로 전달했습니다.  
  
ForkJoinPool에서 실행되는 마지막 invoke 메서드의 반환값은 ForkJoinSumCalculator에서 정의한 태스크의 결과가 됩니다.  
  
일반적으로 애플리케이션에서는 둘 이상의 ForkJoinPool을 사용하지 않습니다.  
즉, 소프트웨어의 필요한 곳에서 언제든 가져다 쓸 수 있도록 ForkJoinPool을 한 번만 인스턴스화해서 **정적 필드에 싱글턴으로 저장합니다.**  
  
ForkJoinPool을 만들면서 인수가 없는 디폴트 생성자를 이용했는데, 이는 JVM에서 이용할 수 있는 모든 프로세서가 자유롭게 풀에 접근할 수 있음을 의미합니다.  
더 정확하게는 Runtime.availableProcessors의 반환값으로 풀에 사용할 스레드 수를 결정합니다.  
  
`사용할 수 있는 프로세서`라는 이름과는 달리 실제 프로세서 외에 `하이퍼스레딩과 관련된 가상 프로세서도 개수에 포함합니다.  
### ForkJoinSumCalculator 실행
ForkJoinSumCalculator를 ForkJoinPool로 전달하면 풀의 스레드가 ForkJoinSumCalculator의 compute 메서드를 실행하면서 작업을 수행합니다.  
  
compute 메서드는 병렬로 실행할 수 있을만큼 태스크의 크기가 충분히 작아졌는지 확인하며, 아직 태스크의 크기가 크다고 판단되면  
숫자 배열을 반으로 분할해서 두 개의 새로운 ForkJoinSumCalculator로 할당한다.  
그러면 다시 ForkJoinPool이 새로 생성된 ForkJoinSumCalculator를 실행한다.  
  
이 과정이 재귀적으로 반복되면서 주어진 조건(예제에서는 덧셈을 수행할 항목이 만 개 이하)을 만족할 때까지 태스크 분할을 반복한다.  
이제 각 서브태스크는 순차적으로 처리되며 포킹 프로세스로 만들어진 이진트리의 태스크를 루트에서 역순으로 방문합니다.  
  
즉, 각 서브태스크의 부분 결과를 합쳐서 태스크의 최종 결과를 계산합니다.  
  
### 7.2.2 포크/조인 프레임워크를 제대로 사용하는 방법
- join 메서드를 태스크에 호출하면 태스크가 생산하는 결과가 준비될 때까지 호출자를 블록시킨다.따라서 두 서브태스크가 모두 시작된 다음에 join을 호출해야 한다.  
- RecursiveTask 내에서는 ForkJoinPool의 invoke 메서드를 사용하지 말아야 한다. 대신 compute나 fork 메서드를 직접 호출할 수 있다.  
- 서브태스크에 fork 메서드를 호출해서 ForkJoinPool의 일정을 조절할 수 있다. 왼쪽 작업과 오른쪽 작업 모두에 fork를 호출하는 것보다는  
한쪽 작업에는 fork 보다는 compute를 호출하는 것이 효율적이다. 두 서브태스크의 한 스레드에는 같은 스레드를 재사용할 수 있으므로 오버헤드를 피할 수 있다.  
- 포크/조인 프레임워크를 이용하는 병렬 계산은 디버깅하기 어렵다.  
- 멀티코어에 포크/조인 프레임워크를 사용하는 것이 순차 처리보다 무조건 빠를 생각은 버려야 한다.  

병렬 처리로 성능을 개선하려면 태스크를 여러 독립적인 서브태스크로 분할할 수 있어야 한다.  
각 서브태스크의 실행시간은 새로운 태스크를 포킹하는 데 드는 시간보다 길어야 한다.  
  
### 7.2.3 작업 훔치기
work stealing  
작업 훔치기 기법에서는 ForkJoinPool의 모든 스레드를 거의 공정하게 분할한다.  
  
각각의 스레드는 자신에게 할당된 태스크를 포함하는 이중 연결 리스트를 참조하면서 작업이 끝날 때마다 큐의 헤드에서 다른 태스크를 가져와서 작업을 처리한다.  
  
다른 스레드는 바쁘게 일하고 있는데 한 스레드는 할일이 다 떨어진 상황  
-> 할일이 없어진 스레드가 유휴 상태(어떠한 프로그램에 의해서도 사용되지 않는 상태)로 바뀌는 것이 아니라 다른 스레드 큐의 꼬리에서 작업을 훔쳐온다.  
모든 태스크가 작업을 끝낼 때까지, 즉 모든 큐가 빌 때까지 이 과정을 반복한다. 따라서 태스크의 크기를 작게 나누어야 작업자 스레드 간의 작업부하를 비슷한 수준으로 유지할 수 있다.  
  
다음 절에서는 스트림을 자동으로 분할하는 기법인 Spliterator 를 설명합니다.  
## 7.3 Spliterator 인터페이스
Java 8에서는 Spliterator라는 새로운 인터페이스를 제공합니다. Spliterator는 '분할할 수 있는 반복자'(Splitable iterator)라는 의미입니다.  
Iterator 처럼 Spliterator는 소스의 요소 탐색 기능을 제공한다는 점은 같지만 Spliterator는 병렬 작업에 특화되어 있습니다.  
  
Java 8은 컬렉션 프레임워크에 포함된 모든 자료구조에 사용할 수 있는 디폴트 Spliterator 구현을 제공합니다.  
컬렉션은 spliterator라는 메서드를 제공하는 Spliterator 인터페이스를 구현합니다.  
  
```java
public interface Spliterator<T> {

}
```






  









