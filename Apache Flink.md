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

`StreamExecutionEnvironment.addSource(sourceFunction)` 메서드를 이용해 소스를 가져올 수 있음














