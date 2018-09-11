# Application Development

## Basic API Concepts

### Overview

- 스트림: StreamingExecutionEnvironment, DataStream API 사용
- 배치: ExecutionEnvironment, DataSet API 사용



#### DataSet and DataStream

- DataStream: unbounded 스트림에 사용
- DataSet: 한정적인 데이터에 사용(bounded 스트림)

한번 생성하면 추가하거나 지울 수 없음.
map, fliter와 같은 메서드를 사용하면 즉시 생성



#### Anatomy of a Flink Program

프로그램 구조

1. execution environment 획득
2. 데이터 로드/생성
3. 데이터 transformations 명시
4. 계산 결과를 어디에 둘 것인지 명시
5. 실행

- DataSet API 패키지: [org.apache.flink.api.java](https://github.com/apache/flink/blob/master//flink-java/src/main/java/org/apache/flink/api/java)
- DataStream API 패키지: [org.apache.flink.streaming.api](https://github.com/apache/flink/blob/master//flink-streaming-java/src/main/java/org/apache/flink/streaming/api)

**StreamExecutionEnvironment 얻는 방법**

```java
getExecutionEnvironment()  // 일반적으로 사용
createLocalEnvironment()
createRemoteEnvironment(String host, int port, String... jarFiles)
```

**파일로부터 데이터 읽기**

```java
final StreamExecutionEnvironment env =
    StreamExecutionEnvironment.getExecutionEnvironment();

// 데이터스트림 생성 -> 다른 데이터스트림으로 변환 가능
DataStream<String> text = env.readTextFile("file:///path/to/file");
```

**스트림 변환**

```java
DataStream<String> input = ...;

// map(): 변환 함수, 여기서는 String을 Integer로 변환
DataStream<Integer> parsed = input.map(new MapFunction<String, Integer>() {
    @Override
    public Integer map(String value) {
        return Integer.parseInt(value);
    }
});
```

**sink 생성 -> 출력**

```java
writeAsText(String path)
print()
```

**실행**

```java
execute()  // StreamExecutionEnvironment에서 실행
```



#### Lazy Evaluation

Flink 프로그램은 메인 메서드를 실행한 즉시 실행되지 않음.
각 연산자들을 프로그램 계획에 추가한 후, `execute()` 메서드 실행 시 실제로 프로그램이 실행.



#### Specifying Keys

- 키를 정의하는 변환: join, coGroup, keyBy, groupBy 등
- 키로 그룹화된 데이터를 사용하는 변환: Reduce, GroupReduce, Aggregate, Windows 등

**DataSet 그룹화**

```java
DataSet<...> input = // [...]
DataSet<...> reduced = input
  .groupBy(/*define key here*/)
  .reduceGroup(/*do something*/);
```

**DataStream 그룹화**

```java
DataStream<...> input = // [...]
DataStream<...> windowed = input
  .keyBy(/*define key here*/)
  .window(/*window specification*/);
```

Flink는 key-vlaue를 기반으로 하지 않기 때문에 물리적으로 key-value 타입을 만들지 않아도 됨.
key는 가상의 키임.



#### Define keys for Tuples

```java
DataStream<Tuple3<Integer,String,Long>> input = // [...]
KeyedStream<Tuple3<Integer,String,Long>,Tuple> keyed = input.keyBy(0)
```

tuple을 첫번째 필드(Integer)로 그룹화

```java
DataStream<Tuple3<Integer,String,Long>> input = // [...]
KeyedStream<Tuple3<Integer,String,Long>,Tuple> keyed = input.keyBy(0,1)
```

첫번째, 두번째 필드를 composite key(복합키)로 하여 그룹화

**nested Tuples**

```java
DataStream<Tuple3<Tuple2<Integer, Float>,String,Long>> ds;
```

여기서 `keyBy(0)`은 Integer, Float을 포함한 Tuple2를 키로 사용



#### Define keys using Field Expressions

Field expression은 Tuple과 POJO와 같은 nested 타입의 필드 선택을 쉽게 해줌.

**POJO의 일반적인 그룹화**

```java
// some ordinary POJO (Plain old Java Object)
public class WC {
  public String word;
  public int count;
}
DataStream<WC> words = // [...]
// word 필드로 그룹화
DataStream<WC> wordCounts = words.keyBy("word").window(/*window specification*/);
```

**Field expression 구문**

- POJO 필드 선택: 필드명
- Tuple 필드 선택: 필드명 or 인덱스번호(f0, 0, f1, 1, ...)
- nested 필드 선택:  "."으로 구분(아래 예제 참고)

**예제**

```java
public static class WC {
  public ComplexNestedClass complex; //nested POJO
  private int count;
  // getter / setter for private field (count)
  public int getCount() {
    return count;
  }
  public void setCount(int c) {
    this.count = c;
  }
}
public static class ComplexNestedClass {
  public Integer someNumber;
  public float someFloat;
  public Tuple3<Long, Long, String> word;
  public IntWritable hadoopCitizen;
}
```

- "count": WC 클래스의 count 필드
- "complex": ComplexNestedClass의 모든 필드
- "complex.word.f2":  Tuple3의 3번째 필드(String)
- "complex.hadoopCitizen": IntWritable의 모든 필드



#### Define keys using Key Selector Functions

Key selector는 하나의 요소를 입력받아 하나의 키를 반환

```java
// some ordinary POJO
public class WC {public String word; public int count;}

DataStream<WC> words = // [...]
KeyedStream<WC> keyed = words
  .keyBy(new KeySelector<WC, String>() {
     public String getKey(WC wc) { return wc.word; }
   });
```



#### Specifying Transformation Functions

대부분의 변환은 사용자가 정의를 해야함

**Implementing an interface**

가장 기본적인 방법

```java
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
};
data.map(new MyMapFunction());
```

**Anonymous classes**

```java
data.map(new MapFunction<String, Integer> () {
  public Integer map(String value) { return Integer.parseInt(value); }
});
```

**Java 8 Lambdas**

```java
data.filter(s -> s.startsWith("http://"));

data.reduce((i1,i2) -> i1 + i2);
```

**Rich functions**

Rich function을 사용하여 변환을 정의할 수 있음.

```java
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
};
```

아래와 같이 사용할 수 있음

```java
class MyMapFunction extends RichMapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
};
data.map(new MyMapFunction());
```

anonymous class로도 표현 가능

```java
data.map(new RichMapFunction<String, Integer>() {
  public Integer map(String value) { return Integer.parseInt(value); }
});
```

Rich function은 map, reduce와 같은 사용자 정의 함수 외에도
open, close, getRuntimeContext, setRuntimeContext 메서드도 제공.



#### Supported Data Types

효율성을 위해 Flink는 데이터 타입을 제한

1. **Java Tuples** and **Scala Case Classes**
2. **Java POJOs**
3. **Primitive Types**
4. **Regular Classes**
5. **Values**
6. **Hadoop Writables**
7. **Special Types**

**Tuples and Case Classes**

```java
DataStream<Tuple2<String, Integer>> wordCounts = env.fromElements(
    new Tuple2<String, Integer>("hello", 1),
    new Tuple2<String, Integer>("world", 2));

wordCounts.map(new MapFunction<Tuple2<String, Integer>, Integer>() {
    @Override
    public Integer map(Tuple2<String, Integer> value) throws Exception {
        return value.f1;
    }
});

wordCounts.keyBy(0); // also valid .keyBy("f0")
```

**POJOs**

제한사항

- 클래스는 public 이어야 함
- 인자가 없는 public 생성자가 있어야 함
- 모든 필드는 public이거나 getter, setter가 있어야 함
- 모든 필드의 타입은 Flink가 지원하는 타입이어야 함

```java
public class WordWithCount {
    public String word;
    public int count;
    public WordWithCount() {}
    public WordWithCount(String word, int count) {
        this.word = word;
        this.count = count;
    }
}

DataStream<WordWithCount> wordCounts = env.fromElements(
    new WordWithCount("hello", 1),
    new WordWithCount("world", 2));

wordCounts.keyBy("word"); // key by field expression "word"
```

**Primitive Types**

Integer, String, Double과 같은 모든 Java와 Scala의 primitive types 지원

**General Class Types**

대부분의 Java와 Scala의 클래스를 지원
단, 직렬화할 수 없는 필드를 가진 클래스는 제외(file pointer, I/O stream 등)

**Values**

직렬화와 직렬화 해제를 수동으로 사용 가능

**Hadoop Writables**

org.apache.hadoop.Writable 인터페이스를 구현한 타입 사용 가능
`write()`, `readFields()` 메서드에 정의된 직렬화 로직 사용 가능

**Special Types**

Scala의 Either, Option, Try와 같은 special types 사용 가능

**Type Erasure & Type Inference**

Type erasure: runtime 시점에서 객체의 인스턴스의 제네릭 타입을 알 수 없음.(`DataStream<String>`과 `DataStream<Long>` 을 JVM은 같은 것으로 봄)

Flink는 프로그램 실행 전에 실행 정보를 알아야 하기 때문에 `DataStream.getType()` 메서드를 제공 -> TypeInformaion 인스턴스 반환



#### Accumulators & Counters

Accumulator.add(V value) 메서드를 통해 값을 더할 수 있음

Flink는 다음과 같은 built-in accumulators 제공(Accumulator 인터페이스 구현)

- IntCounter, LongCounter, DoubleCounter
- Histogram: 내부적으로 Integer to Integer의 map. 값의 분포를 계산할 때 사용. 예) 라인별 단어수의 분포

**사용법**

accumulator(여기서는 counter) 객체 생성

```java
private IntCounter numLines = new IntCounter();
```

accumulator 객체 등록(주로 rich 함수의 `open()`에서), 이름 정의

```java
getRuntimeContext().addAccumulator("num-lines", this.numLines);
```

`open()`, `close()` 메서드와 같이 원하는 곳에서 사용.

```java
this.numLines.add(1);
```

결과는 `JobExecutionResult` 객체(`execute()` 메서드가 반환)에 저장됨

```java
myJobExecutionResult.getAccumulatorResult("num-lines");
```

**Custom accumulators**

Accumulator 또는 SimpleAccumulator 인터페이스를 구현하여 자신만의 accumulator 생성 가능



## Streaming (DataStream API)

### Overview

DataStream 프로그램은 데이터 스트림의 변환(예, 필터링, updating state, windows 정의, 합계)을 구현하는 프로그램이다.
데이터 스트림은 소스(예, message queues, 소켓 스트림, 파일)로부터 생성된다.
결과는 sinks를 통해 반환되고 이는 파일, 표준 출력이 될 수 있다.
Flink 프로그램은 다양한 contexts, standalone, embedded에서 동작한다.
실행은 local JVM, 다양한 machines의 clusters에서 일어난다.



#### Example Program

Streaming window word count 애플리케이션
5초마다 웹 소켓으로 들어오는 단어의 수 카운트

```java
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.util.Collector;

/**
 * 윈도우: nc -l -p 9999 입력 후 실행
 * 리눅스: nc -lk 9999 입력 후 실행
 */
public class WindowWordCount {

    public static void main(String[] args) throws Exception {

        // 실행 환경 설정
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<Tuple2<String, Integer>> dataStream = env
                // 소켓으로부터 문자열을 받음
                .socketTextStream("localhost", 9999)
                // 문자열을 단어별로 나눔
                .flatMap(new Splitter())
                // String을 키로 설정
                .keyBy(0)
                // time windows 5초로 설정
                .timeWindow(Time.seconds(5))
                .sum(1);

        dataStream.print();

        env.execute("Window WordCount");
    }

    public static class Splitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String sentence, Collector<Tuple2<String, Integer>> out) throws Exception {
            for (String word: sentence.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }
    
}
```



#### Data Sources

`StreamExecutionEnvironment.addSource(sourceFunction)` 메서드를 이용해 소스를 가져올 수 있다.
`SourceFunction` 을 구현해서 자신만의 커스텀 소스를 만들 수 있다.
`ParallelSourceFunction`  또는 `RichParallelSourceFunction` 을 구현해서 병렬 소스도 만들 수 있다.

**File-based:**

- `readTextFile(path)` - 텍스트 파일(`TextInputFormat` )을 라인별로 String으로 반환
- `readFile(fileInputFormat, path)`  - 지정된 형식의 파일 읽음
- `readFile(fileInputFormat, path, watchType, interval, pathFilter, typeInfo)` 

**Socket-based:**

- `socketTextStream`  

**Collection-based:**

- `fromCollection(Collection)` - Java 컬렉션으로 데이터 스트림 생성
- `fromCollection(Iterator, Class)`  - iterator로 데이터 스트림 생성, 반환하는 데이터 타입을 Class로 명시
- `fromParallelCollection(SplittableIterator, Class)` 
- `fromElements(T ...)`
- `generateSequence(from, to)` 

**Custom:**

- `addSource` - 예, `addSource(new FlinkKafkaConsumer08<>(...))`를 이용하여 Apache Kafka로 부터 읽을 수 있음. [connectors](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/connectors/index.html) 참조



#### DataStream Transformations

[operators](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/operators/index.html) 참조



#### Data Sinks

Data sinks는 DataStream을 사용하여 파일, 소켓, 외부 시스템으로 전달

- `writeAsText()` / `TextOutputFormat`  -  `toString()`을 이용해 String으로 저장
- `writeAsCsv(...)` / `CsvOutputFormat`  - 컴마로 구분된 튜블로 저장. Row와 field는 설정 가능
- `print()` / `printToErr()`  - Standard out으로 출력. prefix(msg)로 앞에 내용을 붙일 수 있음. 병렬처리의 경우 앞에 식별자 출력
- `writeUsingOutputFormat()` / `FileOutputFormat`  - 메서드와 base class 출력
- `writeToSocket`  - `SerializationSchema`에 따라 소켓으로 저장
- `addSink`  - 커스텀 sink 함수 호출. Flink는 sink 함수로 구현된 Kafka와 같은 다른 시스템에 대한 connector를 제공

`write()` 메서드는 디버깅용 -> 체크포인트를 만들지 않음.

exactly-once를 이용하여 안정적으로 스트림을 파일 시스템으로 보내려면 `flink-connector-filesystem`을 이용하는 것이 좋음. 커스텀으로 하러면 `addSink(...)`  이용



#### Iterations

Interative streaming programs: 데이터 스트림이 끝나지 않는 경우, 어느 부분이 반복으로 피드백되고 어느 부분이 downstream으로 전달되는지 지정해야 함

`IterativeStream` 정의

```java
IterativeStream<Integer> iteration = input.iterate();
```

transformations을 이용하여 반복 실행될 로직을 지정

```java
DataStream<Integer> iterationBody = iteration.map(/* this is executed many times */);
```

반복을 종료하거나 마지막을 정의하려면, `closeWith(feedbackStream)` 메서드 호출

```java
iteration.closeWith(iterationBody.filter(/* one part of the stream */));
DataStream<Integer> output = iterationBody.filter(/* some other part of the stream */);
```

예제) 0이 될 때까지 1씩 감소하는 프로그램

```java
DataStream<Long> someIntegers = env.generateSequence(0, 1000);

// IterativeStream 정의
IterativeStream<Long> iteration = someIntegers.iterate();

// 반복적으로 1씩 감소하도록
DataStream<Long> minusOne = iteration.map(new MapFunction<Long, Long>() {
  @Override
  public Long map(Long value) throws Exception {
    return value - 1 ;
  }
});

// filter를 이용해 0보다 클 경우 필터링
DataStream<Long> stillGreaterThanZero = minusOne.filter(new FilterFunction<Long>() {
  @Override
  public boolean filter(Long value) throws Exception {
    return (value > 0);
  }
});

iteration.closeWith(stillGreaterThanZero);

// filter를 이용해 0보다 작거나 같을 경우 필터링
DataStream<Long> lessThanZero = minusOne.filter(new FilterFunction<Long>() {
  @Override
  public boolean filter(Long value) throws Exception {
    return (value <= 0);
  }
});
```



#### Execution Parameters

`StreamExecutionEnvironment` 에는 실행 환경을 설정할 수 있는 `ExecutionConfig` 가 들어있음.
[execution configuration](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/execution_configuration.html) 참조

##### Fault Tolerance

[State & Checkpointing](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/state/checkpointing.html)에 체크포인트 메커니즘을 설정하는 방법이 나와있음

##### Controlling Latency

데이터를 한번에 보내기 위해 버퍼를 사용한다. Flink config 파일을 통해 버퍼 크기를 설정 가능하다. 
그러나 스트림이 들어오는 속도가 느리면 버퍼가 채워지지 않아 속도 지연이 발생한다. 
이 경우, `env.setBufferTimeout(timeoutMillis)` 를 설정하여 버퍼가 채워지지 않아도 설정한 시간이 지나면 자동으로 데이터를 보낸다. 기본값은 100ms이다.

```java
LocalStreamEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();
env.setBufferTimeout(timeoutMillis);

env.generateSequence(1,10).map(new MyMapper()).setBufferTimeout(timeoutMillis);
```

Throughtput 최대화: `setBufferTimeout(-1)` 으로 설정 시 timeout을 삭제(버퍼가 꽉 채워져야만 데이터를 전송)
Latency 최소화: timeout을 0에 가까운 값(5 or 10)으로 설정. 0으로 하면 성능 저하가 있을 수 있음.



#### Debugging

##### Local Execution Environment

`LocalStreamEnvironment` 은 동일한 JVM에서 Flink를 시작. IDE에서 LocalEnvironment를 통해 break point를 정할 수 있고 디버깅을 쉽게 할 수 있음.

```java
final StreamExecutionEnvironment env = 
    StreamExecutionEnvironment.createLocalEnvironment();

DataStream<String> lines = env.addSource(/* some source */);
// build your program

env.execute();
```

##### Collection Data Sources

Flink는 테스트하기 쉬운 Java 컬렉션을 이용한 data source를 제공

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

// Create a DataStream from a list of elements
DataStream<Integer> myInts = env.fromElements(1, 2, 3, 4, 5);

// Create a DataStream from any Java collection
List<Tuple2<String, Integer>> data = ...
DataStream<Tuple2<String, Integer>> myTuples = env.fromCollection(data);

// Create a DataStream from an Iterator
Iterator<Long> longIt = ...
DataStream<Long> myLongs = env.fromCollection(longIt, Long.class);
```

##### Iterator Data Sink

Flink는 디버깅 목적으로 DataStream 결과를 수집하는 sink 제공

```java
import org.apache.flink.streaming.experimental.DataStreamUtils

DataStream<Tuple2<String, Integer>> myResult = ...
Iterator<Tuple2<String, Integer>> myOutput = DataStreamUtils.collect(myResult)
```



### Event Time

#### Overview

##### Event Time / Processing Time / Ingestion Time

- Processing time: 각 작업을 실행하는 시간
- Event time: 생산 장치로부터 이벤트가 발생하는 시간
- Ingestion time: 이벤트가 Flink로 들어가는 시간



##### Setting a Time Characteristic

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);
// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.addSource(new FlinkKafkaConsumer09<MyEvent>(topic, schema, props));

stream
    .keyBy( (event) -> event.getUser() )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) -> a.add(b) )
    .addSink(...);
```



##### Event Time and Watermarks

##### Watermarks in Parallel Streams

##### Late Elements

##### Debugging Watermarks



## Libraries

### Complex Event Processing(CEP)

CEP 라이브러리를 이용해 끝없는 데이터 스트림에서 이벤트 패턴을 감지하여 중요한 데이터를 파악할 수 있다.

#### Getting Started

CEP dependency 추가

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-cep_2.11</artifactId>
  <version>1.6.0</version>
</dependency>
```

패턴 매칭을 적용하려는 DataStream의 이벤트는 `equals()`와 `hashCode()` 메서드를 구현해야한다.
왜냐하면 CEP는 이벤트를 비교하고 매칭하기위해 이 메서드를 사용하기 때문이다.

```java
DataStream<Event> input = ...

Pattern<Event, ?> pattern = Pattern.<Event>begin("start").where(
        new SimpleCondition<Event>() {
            @Override
            public boolean filter(Event event) {
                return event.getId() == 42;
            }
        }
    ).next("middle").subtype(SubEvent.class).where(
        new SimpleCondition<SubEvent>() {
            @Override
            public boolean filter(SubEvent subEvent) {
                return subEvent.getVolume() >= 10.0;
            }
        }
    ).followedBy("end").where(
         new SimpleCondition<Event>() {
            @Override
            public boolean filter(Event event) {
                return event.getName().equals("end");
            }
         }
    );

PatternStream<Event> patternStream = CEP.pattern(input, pattern);

DataStream<Alert> result = patternStream.select(
    new PatternSelectFunction<Event, Alert>() {
        @Override
        public Alert select(Map<String, List<Event>> pattern) throws Exception {
            return createAlertFrom(pattern);
        }
    }
});
```



### The Pattern API

Pattern API를 사용하면 입력 스트림에서 추출하려는 복잡한 패턴 시퀀스를 정의할 수 있다.

복잡한 패턴 시퀀스는 여러개의 단순 패턴으로 구성된다.
단순 패턴은 **patterns**라 부르고, 스트림에서 찾는 복잡한 패턴 시퀀스는 **pattern sequence**라 부른다.
**match**는 모든 패턴을 방문하는 입력 이벤트의 시퀀스이다.

- 패턴은 고유한 이름을 가져야한다.
- 패턴이름에 ":"을 쓰면 안된다.

#### Individual Patterns

패턴은 singleton 패턴, looping 패턴이 있다.

- Singleton 패턴: 하나의 이벤트만 받음
- Looping 패턴: 하나 이상의 이벤트를 받을 수 있음

예) "a b+ c? d"에서 a, c?, d는 singleton 패턴이고, b+는 looping 패턴

기본적으로 패턴은 싱글톤 패턴이고, [Quantifiers](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/libs/cep.html#quantifiers)를 이용해 루핑 패턴으로 변환할 수 있다.
각 패턴은 이벤트를 받아들이는 [Conditions](https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/libs/cep.html#conditions)을 가질 수 있다.

**Quantifiers**

아래 메서드들을 이용해 루핑 패턴을 명시할 수 있다.

- `pattern.oneOrMore()`: 하나 또는 하나 이상의 이벤트를 발생시키는 패턴(예, 앞에서의 b+)
- `pattern.times(#ofTimes)`: 특정 횟수의 이벤트를 발생시키는 패턴(예, a 4개)
- `pattern.times(#fromTimes, #toTimes)`: 특정 범위내 횟수의 이벤트를 발생시키는 패턴(예, a 2-4개)
- `pattern.greedy()`: 루핑 패턴을 greedy하게 만듦

```java
// start: 패턴

// expecting 4 occurrences
 start.times(4);

 // expecting 0 or 4 occurrences
 start.times(4).optional();

 // expecting 2, 3 or 4 occurrences
 start.times(2, 4);

 // expecting 2, 3 or 4 occurrences and repeating as many as possible
 start.times(2, 4).greedy();

 // expecting 0, 2, 3 or 4 occurrences
 start.times(2, 4).optional();

 // expecting 0, 2, 3 or 4 occurrences and repeating as many as possible
 start.times(2, 4).optional().greedy();

 // expecting 1 or more occurrences
 start.oneOrMore();

 // expecting 1 or more occurrences and repeating as many as possible
 start.oneOrMore().greedy();

 // expecting 0 or more occurrences
 start.oneOrMore().optional();

 // expecting 0 or more occurrences and repeating as many as possible
 start.oneOrMore().optional().greedy();

 // expecting 2 or more occurrences
 start.timesOrMore(2);

 // expecting 2 or more occurrences and repeating as many as possible
 start.timesOrMore(2).greedy();

 // expecting 0, 2 or more occurrences and repeating as many as possible
 start.timesOrMore(2).optional().greedy();
```

**Conditions**

들어오는 이벤트가 패턴으로 받아들여지기 위해 충족시켜아하는 조건(condition)을 지정할 수 있다.
예) 값이 5보다 커야하거나, 이전에 받아들여진 이벤트 값의 평균을 넘어야 한다.

condition은 `pattern.where()`, `pattern.or()`, `pattern.until()` 메서드를 통해 지정할 수 있다.
이들은 `IterativeCondition` 또는 `SimpleCondition`가 될 수 있다.

**Iterative Conditions**: 가장 일반적으로 사용. 이전에 받아들여진 이벤트의 특성이나 통계를 통해 다음의 이벤트를 받아들이는 조건을 지정


아래 예제는 이름이 "foo"로 시작하고, 이전에 허용된 이벤트 가격과 현재 이벤트 가격의 합이 5.0을 넘지 않는 iterative condition이다. Iterative condition은 루핑 패턴과 조합되었을 때 효과적이다.

```java
middle.oneOrMore()
    .subtype(SubEvent.class)
    .where(new IterativeCondition<SubEvent>() {
        @Override
        public boolean filter(SubEvent value, Context<SubEvent> ctx) throws Exception {
            if (!value.getName().startsWith("foo")) {
                return false;
            }
    
            double sum = value.getPrice();
            for (Event event : ctx.getEventsForPattern("middle")) {
                sum += event.getPrice();
            }
            return Double.compare(sum, 5.0) < 0;
        }
    });
```

**Simple condition**: 이벤트 자체의 속성에만 기반하여 이벤트를 허용할지를 결정

```java
start.where(new SimpleCondition<Event>() {
    @Override
    public boolean filter(Event value) {
        return value.getName().startsWith("foo");
    }
});
```

`pattern.subtype(subClass)` 메서드를 통해 허용 이벤트의 타입을 제한할 수 있다.

```java
start.subtype(SubEvent.class).where(new SimpleCondition<SubEvent>() {
    @Override
    public boolean filter(SubEvent value) {
        return ... // some condition
    }
});
```

**Combining Conditions**: 모든 컨디션은 조합하여 사용할 수 있다. 각 컨디션은 AND로 조합되지만 `OR()` 메서드를 사용하여 OR 로직을 사용할 수 있다.

```java
pattern.where(new SimpleCondition<Event>() {
    @Override
    public boolean filter(Event value) {
        return ... // some condition
    }
}).or(new SimpleCondition<Event>() {
    @Override
    public boolean filter(Event value) {
        return ... // or condition
    }
});
```

**Stop condition**: 루핑 패턴의 경우 stop condition을 지정할 수 있다. 예를 들어,

- 패턴: "(a+ until b)"
- 들어온 이벤트: "a1" "c" "a2" "b" "a3"
- 결과: {a1 a2} {a1} {a2} {a3}

{a1 a2 a3}, {a2 a3} 은 stop condition으로 인해 반환되지 않는다.

| 패턴 연산자 | 설명 |
| ----------- | ---- |
|             |      |
|             |      |
|             |      |
|             |      |
|             |      |
|             |      |
|             |      |
|             |      |
|             |      |
|             |      |



#### Combining Patterns



#### Groups of patterns



#### After Match Skip Strategy



### Detecting Patterns

#### Selecting from Patterns

#### Handling Timed Out Partial Patterns



### Handling Lateness in Event Time



### Examples