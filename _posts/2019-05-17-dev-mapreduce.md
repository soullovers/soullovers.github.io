---
layout: post
title: "하둡완벽가이드 > 2. 맵리듀스"
categories:
  - dev
tags:
  - data
  - hadoop
  - mapreduce
---

# 맵리듀스

> - 대용량 데이터를 분산 처리하기 위한 목적으로 개발된 프로그래밍 모델
> - 대표적인 대용량 데이터 처리를 위한 병렬 처리 기법
> - 충분한 장비만 있다면 대규모 분석 가능
> - 사용자가 Map과 Reduce 두 함수를 이용하여 데이터를 처리
> - 다양한 언어로 작성가능


## 2.1 기상데이터 셋
- 20세기 전체 데이터로 연도별 최고 기온 구하기
```
0057
332130   # USAF 기상관측소 식별자
99999    # WBAN 기상관측소 식별자
19500101 # 관측날짜
0300     # 관측시간
4
+51317   # 위도 (위도 x 1000)
+028783  # 경도 (경도 x 1000)
FM-12
+0171    # 고도 (meters)
99999
V020
320      # 바람방향 (도)
1        # 특성코드
N
0072
1
00450    # 구름코드 (meters)
1        # 특성코드
C
N
010000   # 가시거리 (meters)
1        # 특성코드
N
9
-0128    # 기온 (섭씨온도 x 10)
1        # 특성코드
-0139    # 이슬점 온도 (섭씨온도 x 10)
1        # 특성코드
10268    # 대기 입력 (헥토파스칼 x 10)
1        # 특성코드
```

## 2.2 유닉스 도구로 데이터 분석하기
- awk(행기반 데이터를 처리하기 위한 유닉스 도구)로 분석
```
#!/usr/bin/env bash
for year in all/*
do
  echo -ne `basename $year .gz`"\t"
  gunzip -c $year | \
    awk '{ temp = substr($0, 88, 5) + 0;
           q = substr($0, 93, 1);
           if (temp !=9999 && q ~ /[01459]/ && temp > max) max = temp }
         END { print max }'
done
```
> 실행환경 : EC2 고성능 CPU XL인스턴스 1대
> 실행시간 : 42분


#### 1대의 머신에서 병렬로 처리시 문제점
- 일을 동일한 크기로 나누는 것이 어렵다.(연도별로 파일의 크기가 다름)
- 전체 수행시간은 가장 긴 파일을 처리하는 프로세스의 처리시간에 의해 결정
- 독립적인 프로세스의 결과를 합치는데 더 많은 처리가 필요할수 있다.
- 단일 머신의 처리능력의 한계



## 2.3 하둡으로 데이터 분석하기
### 2.3.1 맵과 리듀스
- Map
	- 데이터를 key-value 데이터 셋으로 변환처리
	- 각 행에서 원하는 데이터 추출.
	- 잘못된 데이터를 걸러주는 역할을 수행
- Reduce 
	- 맵함수의 결과(key-value쌍)가 key를 기준으로 정렬되고 그룹화되어 reduce 함수로 전달된다.
	- key별로 그룹화된 리스트에서 최종 결과를 추출


- Shuffle(맵리듀스 프레임워크)
	- Map에서 출력된 key-value쌍을 수신할 리듀서를 판단하는 파티셔닝
	- 해당 리듀서에 대해 모든 입력키를 정렬

![이미지](https://github.com/soullovers/soullovers.github.io/blob/master/assets/img/docs/hddg_0201.png?raw=true "맵과리듀스")
  
  > 1.키-값 쌍으로 변환되어 입력
  > 2.연도와 기온을 추출하여 출력
  > 3.키를 기준으로 정렬하고 기룹화
  > 4.최고측정값 추출

### 2.3.2 자바 맵리듀스
- 연도와 기온을 추출하는 Mapper

```
import java.io.IOException;
 
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
 
public class MaxTemperatureMapper
    extends Mapper<LongWritable, Text, Text, IntWritable> { // 입력키 , 입력값, 출력키, 출력값 ( 네트워크 직렬화를 위해 기본타입셋 제공)
 
  private static final int MISSING = 9999;
   
  @Override
  public void map(LongWritable key, Text value, Context context) //Context : 사용자 코드와 맵리듀스시스템이 통신할수 있는 객체(JobConf, OutputCollector, Reporter 역할 통합)
      throws IOException, InterruptedException {
     
    String line = value.toString();
    String year = line.substring(15, 19);  // 연도 추출
    int airTemperature;
    if (line.charAt(87) == '+') { // parseInt 함수는 앞에 (+) 기호가 있으면 안된다.
      airTemperature = Integer.parseInt(line.substring(88, 92)); // 기온추출
    } else {
      airTemperature = Integer.parseInt(line.substring(87, 92));
    }
    String quality = line.substring(92, 93);
    if (airTemperature != MISSING && quality.matches("[01459]")) {  // 기온 값이 누락되거나 이상한 문제가 있는 레코드를 제거
      context.write(new Text(year), new IntWritable(airTemperature));
    }
  }
}
```

- 최고기온을 구하는 Reducer
```
import java.io.IOException;
 
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;
 
public class MaxTemperatureReducer
    extends Reducer<Text, IntWritable, Text, IntWritable> { // 입력키 , 입력값, 출력키, 출력값
   
  @Override
  public void reduce(Text key, Iterable<IntWritable> values, Context context)
      throws IOException, InterruptedException {
     
    int maxValue = Integer.MIN_VALUE;
    for (IntWritable value : values) {
      maxValue = Math.max(maxValue, value.get());
    }
    context.write(key, new IntWritable(maxValue));
  }
}
```

- 기상 데이터 셋에서 최고기온을 찾는 애플리케이션

```
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
 
public class MaxTemperature {
 
  public static void main(String[] args) throws Exception {
    if (args.length != 2) {
      System.err.println("Usage: MaxTemperature <input path> <output path>");
      System.exit(-1);
    }
     
    Job job = new Job();
    job.setJarByClass(MaxTemperature.class);  // 해당 클래스를 포함한 Jar파일을 찾아서 클러스터에 배치(명시적으로 지정할수도 있다)
    job.setJobName("Max temperature");
 
    FileInputFormat.addInputPath(job, new Path(args[0]));  // 입력경로( 파일, 디렉토리, 특정 패턴의 파일)
    FileOutputFormat.setOutputPath(job, new Path(args[1])); // 출력경로 ( 지정한 디렉토리가 존재하면 안된다. 파일 덮어쓰기로 인한 데이터 손실방지)
     
    job.setMapperClass(MaxTemperatureMapper.class);
    job.setReducerClass(MaxTemperatureReducer.class);
 
    // 출력타입
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
     
    // map의 출력타입이 reduce와 다를 경우
    job.setMapOutputKeyClass(Text.class);
    job.setMapOutputValueClass(IntWritable.class);
 
    System.exit(job.waitForCompletion(true) ? 0 : 1);   // 중간결과의 생성여부 표시 (true, false) -> 프로그램 종료코드
    //에러가 없이 정상적으로 종료되었을 때는 0, 에러가 있을 때에는 0이 아닌 값
  }
}
```



- 실행

```
$ export HADOOP_CLASSPATH=hadoop-examples.jar           // 애플리케이션의 클래스를 classpath에 추가하기 위해 HADOOP_CLASSPATH에 명시적으로 지정
$ hadoop MaxTemperature input/ncdc/sample.txt output    // hadoop명령은 의존성 있는 모든 하둡 라이브러리를 claspath에 추가하고 하둡 환경설정을 불러온다.
```

- 결과
```
$ cat output/part-r-00000   // 리듀스가 하나이므로 파일 1개 생성
1949    111
1950    22
```


# 2.4 분산형으로 확장하기
- 전체 데이터를 HDFS(분산 파일 시스템)에 저장 - 3장
- YARN(하둡자원관리 시스템) - 4장
	- 데이터의 일부분이 저장된 클러스터의 각 머신에서 맵리듀스 프로그램 실행
	- 스케줄링
	- 태스크 실패시 다른 노드로 자동 재할당


## 2.4.1 데이터 흐름

#### MapReduce Job 
- 클라이언트가 수행하는 작업의 기본단위
- 입력데이터, 맵리듀스 프로그램, 설정정보


#### 스플릿
- 잡의 입력을 고정크기 조각으로 분리
- 각 스플릿마다 하나의 맵태스크를 생성하고 스플릿의 각 레코드를 맵함수로 처리
- 전체 입력을 통채로 처리하는것보다 스플릿으로 분리된 조각을 각각 처리하는 것이 훨씬 빠르다.
- 스플릿이 작을수록, 스플릿이 세분화 될수록 부하분산의 효과가 커진다.
- 너무 작으면 스플릿관리와 맵태스크 생성을 위한 오버헤드 발생
- 적절한 크기
	- HDFS 블록의 기본 크기가 적절하다.(블록크기가 단일노드에 저장된다고 확신할수 있는 가장 큰 입력크기)
	- HDFS블록 크기는 클러스터 설정에 따라 다르다



#### 데이터 지역성 최적화
- HDFS내의 입력데이터가 있는 노드에서 맵태스크를 실행 할때 가장 빠르게 작동한다. (a)
- 맵 태스크의 입력 스플릿에 해당하는 HDFS 블록 복제본이 저장된 세 개의 노드 모두 다른 맵 태스크를 실행하여 여유가 없을 경우 블록 복제본이 저장된 동일 랙에 속한 다른 노드에서 가용한 맵 슬롯을 찾음.(b)
- 데이터 복제본이 저장된 노드가 없는 외부 랙의 노드가 선택될 경우 랙 간에 네트워크 전송이 불가피하게 일어남. (c)
- 리듀스태스크는 모든 맵의 결과를 입력으로 받으므로 데이터 지역성의 장점이 없다.

![이미지](https://github.com/soullovers/soullovers.github.io/blob/master/assets/img/docs/hddg_0202.png?raw=true "데이터 지역성 최적화")



#### 태스크 결과
- Map
	- 로컬에 저장
	- 리듀스가 최종결과를 생성하기 위한 중간 결과물
	- 잡이 완료된 후에는 삭제

- Reduce
	- hdfs에 저장
	- hdfs블록의 첫번째 복제본은 로컬노드에 저장
	- 나머지 복제본은 외부 랙에 저장	


#### 단일 리듀스 테스크의 맵 리듀스 데이터 흐름

![이미지](https://github.com/soullovers/soullovers.github.io/blob/master/assets/img/docs/hddg_0203.png?raw=true "단일 리듀스 테스크의 맵 리듀스 데이터 흐름")

#### 다수의 리듀스 태스크의 맵 리듀스 데이터 흐름
![이미지](https://github.com/soullovers/soullovers.github.io/blob/master/assets/img/docs/hddg_0204.png?raw=true "다수의 리듀스 태스크의 맵 리듀스 데이터 흐름")
- 리듀스 수만큼 파티션을 생성하고 맵의 결과를 각 파티션에 분배한다.
- 같은 키에 속한 모든 레코드는 같은 파티션에 배치
- 파티셔닝
	- 해시 함수로 키를 분배하는 기본 파티셔너를 주로 사용
	- 사용자 정의 파티셔닝 함수 사용가능
- 리듀스 수를 선택하는 것은 잡의 실행시간에 미치는 영향이 크므로 튜닝이 필요하다.(7.3절)


## 2.4.2 컴바이너 함수
- 맵과 리듀스 사이의 셔플 단계에서 전송되는 데이터 양을 최소화하여 최적화
- 맵의 결과를 처리하여 리듀스함수로 전달된다.
- 리듀스 함수를 컴파이너 함수로 재사용할수 있다.
- 모든 함수에 적용은 불가능(ex:mean())

![이미지](hhttps://github.com/soullovers/soullovers.github.io/blob/master/assets/img/docs/combiner2.png?raw=true "컴바이너")


- 컴바이너
```
public class MaxTemperatureWithCombiner {
 
  public static void main(String[] args) throws Exception {
    if (args.length != 2) {
      System.err.println("Usage: MaxTemperatureWithCombiner <input path> " +
          "<output path>");
      System.exit(-1);
    }
     
    Job job = new Job();
    job.setJarByClass(MaxTemperatureWithCombiner.class);
    job.setJobName("Max temperature");
 
    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
     
    job.setMapperClass(MaxTemperatureMapper.class);
    job.setCombinerClass(MaxTemperatureReducer.class); // 리듀서클래스를 컴바이너클래스로 지정
    job.setReducerClass(MaxTemperatureReducer.class);
 
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
     
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}
```




### 2.4.3 분산 맵 리듀스 잡 실행하기
- 분산 맵리듀스 잡 실행결과
> - 실행환경 : EC2 클러스터의 10개노드(고성능 CPU XL 인스턴스)
> - 실행시간 : 6분


# 2.5 하둡스트리밍
- 유닉스 표준 스트리밍 사용
- 표준입력을 읽고 표준 출력을 쓸수있는 모든 프로그래밍 언어를 지원
- 텍스트 처리에 적합 

```
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-*.jar   //스트리밍 옵션X, 스트리밍 JAR파일을 jar옵션으로 지정
-files ch02-mr-intro/src/main/ruby/max_temperature_map.rb,              //클러스로 전송
ch02-mr-intro/src/main/ruby/max_temperature_reduce.rb
-input input/ncdc/all                                                   // input 경로
-output output                                                          // output 경로
-mapper ch02-mr-intro/src/main/ruby/max_temperature_map.rb              //Mapper 지정
-combiner ch02-mr-intro/src/main/ruby/max_temperature_reduce.rb         //Combiner 지정
-reducer ch02-mr-intro/src/main/ruby/max_temperature_reduce.rb          //Reducer 지정

```






