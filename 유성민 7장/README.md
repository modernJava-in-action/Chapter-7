## 병렬 데이터 처리와 성능
### 병렬 스트림
  컬렉션에 parallelStream을 호출하면 병렬 스트림이 생성된다. 이를 이용하면 병렬 모든 멀티코어 프로세서가 각각의 청크를 처리하도록 할당할 수 있다. 간단한 예를 통해서 병렬 스트림에 대해서 알아보자. 아래는 숫자 n을 인수로 받아서 1부터 n까지의 모든 숫자의 합계를 반환하는 순차 스트림을 사용하는 메서드이다.
```java
  public static long sequentialSum(long n) {
    return Stream.iterate(1L, i -> i + 1)
                 .limit(n)
                 .reduce(Long::sum)
                 .get();
  }
```
  <br>

### 스트림 성능 측정
아래 JMH를 이용한 벤치마크 코드를 통해서 각 스트림의 성능을 측정해보자.
```java
import java.util.concurrent.TimeUnit;
import java.util.stream.LongStream;
import java.util.stream.Stream;

import org.openjdk.jmh.annotations.Benchmark;
import org.openjdk.jmh.annotations.BenchmarkMode;
import org.openjdk.jmh.annotations.Fork;
import org.openjdk.jmh.annotations.Level;
import org.openjdk.jmh.annotations.Measurement;
import org.openjdk.jmh.annotations.Mode;
import org.openjdk.jmh.annotations.OutputTimeUnit;
import org.openjdk.jmh.annotations.Scope;
import org.openjdk.jmh.annotations.State;
import org.openjdk.jmh.annotations.TearDown;
import org.openjdk.jmh.annotations.Warmup;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.RunnerException;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;

@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)          //벤치마크 대상 메서드를 실행하는 데 걸린 평균 시간 측정
@OutputTimeUnit(TimeUnit.MILLISECONDS)    //벤치마크 결과를 밀리초 단위로 출력
@Fork(value = 2, jvmArgs = { "-Xms4G", "-Xmx4G" })    //4Gb의 힙 공간을 제공한 환경에서 두 번 벤치마크를 수행해 결과의 신뢰성 확보
@Measurement(iterations = 2)
@Warmup(iterations = 3)
public class ParallelStreamBenchmark {

  private static final long N = 10_000_000L;

  @Benchmark
  public long iterativeSum() {
    long result = 0;
    for (long i = 1L; i <= N; i++) {
      result += i;
    }
    return result;
  }

  @Benchmark
  public long sequentialSum() {
    return Stream.iterate(1L, i -> i + 1).limit(N).reduce(0L, Long::sum);
  }

  @TearDown(Level.Invocation)     //벤치마크 실행후 가비지 컬렉터 실행
  public void tearDown() {
    System.gc();
  }

  public static void main(String[] args) throws RunnerException {

    Options build = new OptionsBuilder()
      .include(ParallelStreamBenchmark.class.getSimpleName())
      .build();

    new Runner(build).run();
  }
}
```
  위 벤치마크를 실행했을때 parallelSum의 결과가 sequentialSum 보다 빠를 것이라고 예상할수 있지만 실제로는 parallelSum이 다섯배나 더 느리다. 이는 반복 결과로 박싱된 객체가 만들어지므로 숫자를 언박싱 해야 하고, 반복 작업은 병렬로 수행할 수 있는 독립 단위로 나누기 어렵기 때문이다. 이렇듯 병렬 프로그래밍을 완전히 이해하지 않고 사용할 경우 선능이 더 나빠질수도 있기 때문에 내부적으로 어떤 일이 일어나는지 이해하고 반드시 확인해봐야 한다.
  <br>

**특화된 메서드 사용**
  위 병렬 스트림의 성능 문제를 해결해보자. 멀티코어 프로세스를 활용해서 효과적으로 병렬로 실행하려면 LongStream.rangeClosed라는 메서드를 사용해볼수 있다. 이 메서드는 iterate에 비해 다음과 같은 장점을 제공한다.

  - LongStream.rangeClosed는 기본형 long을 사용하기 때문에 박싱과 언박싱 오버헤드가 사라진다.
  - LongStream.rangeClosed는 쉽게 청크로 분할할 수 있는 숫자 범위를 생산한다. 
  <br>
```java
  @Benchmark
  public long rangedSum() {
    return LongStream.rangeClosed(1, N).reduce(0L, Long::sum);
  }

  @Benchmark
  public long parallelRangedSum() {
    return LongStream.rangeClosed(1, N).parallel().reduce(0L, Long::sum);
  }
```
  위는 이를 활용한 예제 코드이다. rangedSum도 sequentialSum의 벤치마킹 결과보다 빠르지만 이에 prallel을 추가한 parallelRangedSum이 더욱 빠르다. 이렇듯 올바른 자료구조를 선택해야 병렬 실행도 최적의 성능을 발휘할 수 있다는 사실을 확인할 수 있다.
  하지만 병렬화가 완전 공짜는 아니라는 사실을 기억하자. 병렬화를 이용하려면 스트림을 재귀적으로 분할해야 하고, 각 서브스트림을 서로 다른 스레드의 리듀싱 연산으로 할당하고, 이들 결과를 하나의 값으로 합쳐야 한다. 또한 코어 간에 데이터 전송을 생각보다 비싸기 때문에 코어 간에 데이터 전송 시간보다 훨씬 오래 걸리는 작업만 병렬로 다른 코어에서 수행하도록 하는것이 바람직하다.

#### 병렬 스트림의 올바른 사용법
  - 확신이 서지 않으면 직접 측정하라. 항상 병렬 스트림이 더 빠른것은 아니기 때문에.
  - 박싱을 주의하라. 오토 박싱과 언박싱은 성능을 크게 저하시킬 수 있다. 최대한 기본형 특화 스트림을 활용하는것이 좋다.
  - 순차 스트림보다 병렬 스트림에서 성능이 떨어지는 연산이 있다. 특히 요소의 순서에 의존하는 limit나 findFirst에는 병렬을 사용하지 말고, findAny 등 순서에 상관이 없을때 사용하는것이 좋다. 순서에 상관이 없는 limit을 사용하려면 정렬된 스트림에 unordered를 호출한 후 limit을 호출하는 것이 더 효율적이다.
  - 스트림에서 수행하는 전체 파이프라인 연산 비용을 고려하라.
  - 소량의 데이터에서는 병렬 스트림이 도움 되지 않는다.
  - 스트림을 구성하는 자료구조가 적절한지 확인하라. ArrayList는 LinkedList 보다 효율적으로 분할할 수 있다. LinkedList를 분할하려면 모든 요소를 탐색해야하기 때문이다.
  - 스트림의 특성과 파이프라인의 중간 연산이 스트림의 특성을 어떻게 바꾸는지에 따라 분해 과정의 성능이 달라질 수 있다.
  - 최종 연산의 병합 과정 비용을 살펴보라. 
  <br>
### 포크/조인 프레임워크
#### RecursiveTask 활용
  스레드 풀을 이용하려면 `RecursiveTask<R>`의 서브 클래스를 만들어야 한다. 여기서 R은 병렬화된 태스크가 생성하는 결과 형식 또는 결과가 없을 때는 Recursive Action 형식이다. RecursiveTask를 정의하려면 추상 메서드 compute를 구현해야 한다.
  `protected abstract R compute();`
  compute 메서드는 태스크를 서브태스크로 분할하는 로직과 더 이상 분할할 수 없을 때 개별 서브태스크의 결과를 생산할 알고리즘을 정의한다. 따라서 대부분의 compute 메서드 구현은 다음과 같은 의사코드 형식을 유지한다.
```java
  if (태스크가 충분히 작거나 더 이상 분할할 수 없으면) {
    순차적으로 태스크 계산
  } else {
    태스크를 두 서브태크스로 분할
    태스크가 다시 서브태스크로 분할되도록 이 메서드를 재귀적으로 호출함
    모든 서브태스크의 연산이 완료될 때까지 기다림
    각 서브태스크의 결과를 합침
  }
```
  일반적으로 애플리케이션에서는 필요한 곳에서 언제든 가져다 쓸 수 있도록 ForkJoinPool을 한 번만 인스턴스화해서 정적 필드에 싱글턴으로 저장한다. 
  <br>
  
  포크/조인 프레임워크를 이용해서 범위의 숫자를 더하는 문제를 통해서 사용 방법을 알아보자.
```java
import static modernjavainaction.chap07.ParallelStreamsHarness.FORK_JOIN_POOL;

import java.util.concurrent.ForkJoinTask;
import java.util.concurrent.RecursiveTask;
import java.util.stream.LongStream;

public class ForkJoinSumCalculator extends RecursiveTask<Long> {

  public static final long THRESHOLD = 10_000;

  private final long[] numbers;
  private final int start;
  private final int end;

  public ForkJoinSumCalculator(long[] numbers) {
    this(numbers, 0, numbers.length);
  }

  private ForkJoinSumCalculator(long[] numbers, int start, int end) {
    this.numbers = numbers;
    this.start = start;
    this.end = end;
  }

  @Override
  protected Long compute() {
    int length = end - start;
    if (length <= THRESHOLD) {
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
    for (int i = start; i < end; i++) {
      sum += numbers[i];
    }
    return sum;
  }

  public static long forkJoinSum(long n) {
    long[] numbers = LongStream.rangeClosed(1, n).toArray();
    ForkJoinTask<Long> task = new ForkJoinSumCalculator(numbers);
    return FORK_JOIN_POOL.invoke(task);
  }

}
```
  ForkJoinSumCalculator를 ForkJoinPool로 전달하면 풀의 스레드가 ForkJoinSumCalculator의 compute 메서드를 실행하면서 작업을 수행한다. 아직 태스크의 크기가 크다고 판단되면 숫자 배열을 반으로 분할해서 두 개의 새로운 ForkJoinSumCalculator로 할당한다. 그러면 다시 ForkJoinPool이 새로 생성된 ForkJoinSumCalculator를 실행하고, 이 과정이 주어진 과정을 만족할때까지 반복된다. 

#### 포크/조인 프레임워크를 제대로 사용하는 방법
  - 두 서브태스크가 모두 시작된 다음에 join을 호출해야 한다. 그렇지 않으면 각각의 서브태스크가 다른 태스크가 끝나길 기다리고, 이능 성능저하로 이어진다.
  - RecursiveTask 내에서는 ForkJoinPool의 invoke 메서드를 사용하지 말아야 한다. 순차 코드에서 병렬 계산을 시작할 때만 invoke를 사용한다.
  - 서브태스크에 fork 메서드를 호출해서 ForkJoinPool의 일정을 조절할 수 있다.
  - 포크/조인 프레임워크를 이용하는 병렬 계산은 디버깅하기 어렵다.
  - 멀티코어에 포크/조인 프레임워크를 사용하는 것이 순차 처리보다 무조건 빠를 거라는 생각은 버려야 한다.
  <br>

### Spliterator 인터페이스
  자바 8은 Spliterator라는 Splitable Iterator의 역할을 하는 인터페이스를 제공한다. 이 인터페이스는 소스의 요소 탐색 기능을 제공한다는 점은 Iterator와 같지만 병렬 작업에 특화되어있다. 커스텀 Spliterator를 직접 구현해야 하는 것은 아니지만 이 인터페이스를 이해한다면 병령 스트림 동작을 훨씬 잘 이해할수 있다.
  Spliterator Interface:
```java
  public interface Spliterator<T> {
    boolean tryAdvance(Consumer<? super T> action);
    Spliterator<T> trySplit();
    long estimateSize();
    int characteristics();
  }
```
  - 여기서 T는 Spliterator에서 탐색하는 요소의 형식을 가리킨다.
  - tryAdvance 메서드는 Spliterator의 요소를 하나씩 순차적으로 소비하면서 탐색해야 할 요소가 남아있으면 참을 반환한다 (Iterator의 동작과 같다).
  - trySplit은 Spliterator의 일부 요소를 분할해서 두 번째 Spliterator를 생성하는 메서드이다.
  - estimateSize 메서드로 탐색해야 할 요소 수 정보를 제공할 수 있다.
  - 
  <br>

#### 분할 과정
  Spliterator도 재귀적으로 trySplit을 호출하는 과정을 반복하면서 여러 스트림으로 분할된다. trySplit이 null을 반환하면 더 이상 자료구조를 분할할 수 없음을 의미한다.

#### Spliterator 특성
  Spliterator는 characteristics라는 추상 메서드도 정의한다. 이 메서드는 Spliterator 자체의 특성 집합을 포함하는 int를 반환한다. 아래는 Spliterator의 특성 정보 리스트이다:
  - ORDERED : 리스트처럼 요소에 정해진 순서가 있으므로 Spliterator는 요소를 탐색하고 분할할 때 이 순서에 유의해야 한다.
  - DISTINCT : x, y 두 요소를 방문했을 때 x.equals(y)는 항상 false를 반환한다.
  - SORTED : 탐색된 요소는 미리 정의된 정렬 순서를 따른다.
  - SIZED : 크기가 알려진 소스로 Spliterator를 생성했으므로 estimatedSize()는 정확한 값을 반환한다.
  - NON-NULL : 탐색하는 모든 요소는 null이 아니다.
  - IMMUTABLE : 이 Spliterator의 소스는 불변이다. 요소를 탐색하는 동안 요소 추가, 삭제, 수정이 불가능하다.
  - CONCURRENT : 동기화 없이 Spliterator의 소스를 여러 스레드에서 동시에 고칠 수 있다.
  - SUBSIZED : 이 Spliterator 그리고 분할되는 모든 Spliterator는 SIZED 특성을 갖는다.


### 정리
  - 내부 반복을 이용하면 명시적으로 다른 스레드를 사용하지 않고도 스트림을 병렬로 처리할 수 있다.
  - 간단하게 스트림을 병렬로 처리할 수 있지만 항상 병렬 처리가 빠른 것은 아니다.
  - 병렬 스트림으로 데이터 집합을 병렬 실행할 때 특히 처리해야 할 데이터가 아주 많거나 각 요소를 처리하는 데 오랜 시간이 거릴때 성능을 높일 수 있다.
  - 가능하면 기본형 특화 스트림을 사용하여 오토 박싱, 언박싱을 피하는 것이 성능을 크게 향상 시킨다.
  - Spliterator는 탐색하려는 데이터를 포함하는 스트림을 어떻게 병렬화 할지 정의한다.