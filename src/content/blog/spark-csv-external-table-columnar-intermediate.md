---
title: "CSV External Table을 직접 집계하면 왜 Spark가 느려지는가"
description: "Spark aggregation job에서 CSV external table full scan 병목을 Parquet 중간 테이블로 줄이며 2시간 이상 걸리던 처리를 약 20분으로 줄인 과정을 정리했습니다."
category: "Data Engineering"
pubDate: "2026-06-30T00:00:00+09:00"
---

## TL;DR

Spark job이 느려졌을 때 가장 먼저 떠올리기 쉬운 대응은 executor를 늘리거나 timeout을 늘리는 것입니다.

하지만 이번 문제의 핵심은 compute가 아니라 **데이터 레이아웃과 scan 방식**이었습니다.

문제 상황은 단순했습니다.

```text
기존 실행 시간
1시간 timeout 안에 완료

변경 후 실행 시간
2시간 이상 소요
```

처음에는 Airflow task timeout을 늘리고 executor 수를 늘리는 방향도 검토했습니다.

하지만 실행 계획을 보면 병목은 더 근본적인 곳에 있었습니다.

```text
CSV external table
→ Spark aggregation
→ full scan 중심 실행
```

CSV 기반 external table을 직접 집계하면서 Spark가 필요한 컬럼과 조건만 효율적으로 읽지 못했고, aggregation query가 큰 scan 비용을 계속 떠안는 구조였습니다.

최종 방향은 원천 CSV external table을 바로 집계하지 않는 것이었습니다.

```text
CSV external table
→ Parquet intermediate table
→ aggregation / feature ingestion
```

이후 실행 시간은 다음처럼 줄었습니다.

```text
변경 전: 2시간 이상
변경 후: 약 20분
```

이 글의 주장은 하나입니다.

> Spark 성능 문제는 executor 부족이 아니라 데이터 레이아웃 문제일 수 있다. 특히 CSV external table을 대규모 aggregation의 직접 입력으로 쓰고 있다면, compute를 늘리기 전에 scan plan과 저장 포맷부터 봐야 한다.

## 1. 문제 상황

어떤 feature aggregation pipeline에서 upstream 데이터 소스가 변경된 뒤 Spark job 실행 시간이 급격히 늘었습니다.

기존에는 Airflow task가 1시간 timeout 안에 완료되었습니다. 그런데 변경 이후에는 같은 job이 2시간을 넘기기 시작했습니다.

운영 관점에서는 단순히 느려진 문제가 아니었습니다.

```text
Airflow task timeout 발생
↓
feature ingestion 지연
↓
downstream pipeline 대기
↓
SLA 불안정
```

그래서 먼저 필요한 것은 "왜 느려졌는가"를 분리하는 일이었습니다.

Spark job이 느려지는 이유는 여러 가지가 있습니다.

```text
데이터 양 증가
executor 부족
shuffle 증가
skew 발생
partition 설계 문제
file format 문제
predicate pushdown 실패
불필요한 full scan
```

겉으로 보이는 현상은 모두 비슷합니다.

```text
job이 오래 돈다
timeout이 난다
```

하지만 원인에 따라 대응은 완전히 달라집니다.

## 2. 처음 떠올린 대응: timeout과 executor 증가

가장 빠른 완화책은 timeout을 늘리는 것입니다.

```text
기존 timeout: 1시간
검토 timeout: 2시간
```

이 방식은 당장 Airflow failure를 줄일 수 있습니다. 하지만 job이 2시간 걸리는 구조가 그대로라면 문제를 해결한 것은 아닙니다.

실패를 늦춘 것에 가깝습니다.

두 번째로 검토한 것은 compute 증설이었습니다.

```text
num executors 증가
executor memory 증가
```

실제로 executor 수를 늘리는 방향도 테스트했습니다.

이 방식 역시 일부 상황에서는 효과가 있습니다. CPU나 memory가 진짜 병목이라면 당연히 compute 증설이 맞습니다.

하지만 이번 경우에는 애매했습니다.

executor를 늘려도 Spark가 계속 같은 데이터를 넓게 읽고, 같은 방식으로 full scan을 수행한다면 비용이 커질 뿐입니다.

```text
compute 부족 문제
→ executor 증설이 효과적

scan 구조 문제
→ executor 증설은 비용 높은 완화책
```

따라서 timeout과 executor 증설은 임시 대응으로만 두고, 실행 계획과 upstream table 구조를 확인했습니다.

## 3. 원인: CSV external table 직접 aggregation

병목의 핵심은 upstream table의 성격이었습니다.

원천 데이터는 CSV 기반 external table로 노출되어 있었습니다. 그리고 aggregation query는 이 external table을 직접 읽고 있었습니다.

구조를 단순화하면 다음과 같습니다.

```text
S3 objects
↓
CSV external table
↓
Spark SQL aggregation
↓
feature table
```

CSV는 사람이 보기 쉽고, 단순한 데이터 교환에는 편합니다.

하지만 대규모 Spark aggregation의 저장 포맷으로는 불리한 점이 많습니다.

```text
column pruning에 불리함
predicate pushdown에 불리함
schema/type handling 비용 증가
scan 단위가 커지기 쉬움
반복 aggregation에서 같은 비용이 반복됨
```

특히 aggregation query가 필요한 컬럼 일부만 사용하더라도, CSV 파일은 columnar format처럼 필요한 컬럼만 효율적으로 읽기 어렵습니다.

결과적으로 Spark는 필요한 정보보다 훨씬 넓은 범위의 데이터를 읽게 됩니다.

문제는 이것이 일회성 비용이 아니라는 점입니다.

pipeline이 매번 같은 external table을 직접 aggregation하면 같은 scan 비용이 반복됩니다.

```text
한 번 느림
→ 운영 이슈

매번 느림
→ 구조적 병목
```

이 경우 timeout을 늘리는 것은 구조적 병목을 숨기는 조치가 됩니다.

## 4. 왜 Parquet 중간 테이블인가

최종 방향은 CSV external table을 바로 aggregation하지 않는 것이었습니다.

대신 원천 데이터를 먼저 Spark가 다루기 좋은 중간 테이블로 적재합니다.

```text
Before

CSV external table
→ aggregation
→ feature ingestion
```

```text
After

CSV external table
→ Parquet intermediate table
→ aggregation
→ feature ingestion
```

여기서 핵심은 "중간 테이블을 하나 더 만든다"가 아닙니다.

핵심은 aggregation의 입력을 바꾸는 것입니다.

CSV external table은 원천 ingestion boundary로만 사용하고, 반복적으로 무거운 aggregation을 수행하는 단계에서는 Parquet 같은 columnar format 기반 테이블을 사용합니다.

이렇게 하면 이후 Spark job은 다음 이점을 얻을 수 있습니다.

```text
필요한 컬럼 중심으로 읽기 쉬움
predicate / partition 조건을 활용하기 쉬움
반복 aggregation의 scan 비용 감소
schema와 type handling이 안정적
downstream query 구조가 단순해짐
```

문제의 성격도 바뀝니다.

```text
CSV를 매번 직접 크게 읽는 문제
↓
한 번 정리한 중간 테이블을 기준으로 후처리하는 문제
```

대규모 배치에서는 이 차이가 꽤 큽니다.

## 5. 선택지 비교

이번 문제에서 검토한 선택지는 크게 세 가지였습니다.

| 선택지 | 장점 | 한계 | 판단 |
|---|---|---|---|
| timeout 증가 | 적용이 빠르다 | runtime 자체는 그대로다 | 임시 완화책 |
| executor 증설 | compute 병목이면 효과가 있다 | full scan 구조가 그대로면 비용이 커진다 | 보조 수단 |
| Parquet 중간 테이블 | scan 비용과 query 구조를 바꾼다 | 중간 적재 단계가 추가된다 | 근본 대응 |

처음부터 Parquet 중간 테이블이 항상 정답이라는 뜻은 아닙니다.

데이터가 작거나, 일회성 job이거나, query latency가 중요하지 않다면 CSV external table을 그대로 써도 충분할 수 있습니다.

하지만 다음 조건이 겹치면 이야기가 달라집니다.

```text
반복 실행되는 batch job
대규모 aggregation
1시간 이상 runtime
Airflow SLA가 있음
CSV external table 직접 scan
timeout 증가가 반복됨
```

이 경우에는 compute를 늘리기 전에 저장 포맷과 scan plan을 먼저 의심해야 합니다.

## 6. 변경 후 결과

구조 변경 후 실행 시간은 크게 줄었습니다.

```text
변경 전: 2시간 이상
변경 후: 약 20분
```

이 수치를 해석할 때 중요한 것은 변경 범위입니다.

이번 사례에서는 executor 수나 memory를 늘려서 얻은 결과가 아니었습니다.

aggregation logic은 유지하고, query step의 입력만 바꿨습니다.

```text
Before
CSV external table 직접 aggregation

After
Parquet intermediate table 기준 aggregation
```

따라서 이 개선은 CSV external table을 직접 읽던 query step을 Parquet intermediate table 기준으로 바꾼 효과로 보는 것이 맞습니다.

다만 이런 성능 비교를 글로 남길 때는 다음 조건도 함께 기록하는 편이 좋습니다.

```text
executor 수
executor memory / core
input data 기간과 row count
source file 개수와 크기
shuffle partition 수
동일한 aggregation logic
동일한 cluster 부하 수준
```

즉, 결론은 "Parquet이면 항상 6배 빨라진다"가 아닙니다.

결론은 "반복 aggregation의 입력을 row-oriented raw boundary에서 columnar intermediate boundary로 옮기면 scan 비용을 줄일 수 있다"에 가깝습니다.

단순히 빠르게 만든 것보다 더 중요한 변화는 실패 방식이 바뀐 것입니다.

기존에는 job이 timeout에 걸릴 가능성이 있었고, timeout을 늘려도 같은 병목이 반복될 수 있었습니다.

변경 후에는 aggregation 단계가 원천 CSV external table의 full scan 비용에 직접 묶이지 않게 되었습니다.

즉, 개선의 성격은 다음에 가깝습니다.

```text
runtime tuning
보다는
pipeline shape 변경
```

Spark job을 빠르게 만드는 방법은 여러 가지가 있지만, 이번 경우에는 parameter tuning보다 data layout 변경이 더 큰 효과를 냈습니다.

다만 Parquet 중간 테이블도 공짜는 아닙니다.

원천 CSV를 읽는 비용은 사라지는 것이 아니라 중간 테이블 생성 단계로 이동합니다.

```text
CSV external table scan
→ Parquet intermediate table 생성
→ 반복 aggregation에서 재사용
```

따라서 이 방식이 이득이 되려면 중간 테이블 생성 비용보다 downstream에서 반복 절감되는 scan 비용이 커야 합니다.

확인해야 할 값은 다음입니다.

```text
intermediate 생성 시간
intermediate 생성 주기
하루 downstream aggregation 실행 횟수
각 aggregation의 input bytes 감소량
중간 테이블 저장 비용
```

만약 aggregation이 하루 한 번뿐이고 원천 CSV도 작다면, 중간 테이블을 만드는 비용이 더 클 수 있습니다.

## 7. 이 사례에서 얻은 체크리스트

비슷한 문제를 다시 만나면 다음 순서로 볼 것 같습니다.

### 7.1 timeout부터 늘리지 않는다

timeout 증가는 운영 실패를 줄이는 데 도움이 됩니다.

하지만 그것이 원인 분석을 대체하면 안 됩니다.

```text
timeout 증가 전 확인할 것

실행 시간이 갑자기 늘어난 시점
upstream 데이터 소스 변경 여부
Spark UI의 scan/shuffle 비중
input file format
partition pruning 동작 여부
predicate pushdown 가능 여부
```

Spark UI나 `explain`에서는 다음 지표를 같이 봐야 합니다.

```text
Scan 단계의 input size / records
읽은 column 수
PushedFilters 또는 PartitionFilters
shuffle read/write size
task duration 분포
특정 task만 긴 skew 패턴
```

이 지표를 봐야 CSV scan이 진짜 병목인지, 아니면 aggregation 이후의 shuffle이나 group key skew가 병목인지 분리할 수 있습니다.

### 7.2 executor 증가가 맞는 문제인지 확인한다

executor 증설은 강력하지만 비쌉니다.

그리고 모든 병목에 맞는 답은 아닙니다.

```text
executor 증설이 잘 맞는 경우
CPU 부족
memory 부족
parallelism 부족

executor 증설만으로 부족한 경우
불필요한 full scan
비효율적인 file format
잘못된 partition 설계
반복되는 upstream scan
```

### 7.3 CSV external table을 반복 집계하지 않는다

CSV external table은 ingestion boundary로는 괜찮을 수 있습니다.

하지만 반복적인 대규모 aggregation의 입력으로 직접 쓰기 시작하면 문제가 커질 수 있습니다.

```text
CSV external table
→ raw landing / compatibility boundary

Parquet table
→ repeated aggregation / analytics / feature pipeline
```

### 7.4 중간 테이블은 복잡도가 아니라 비용 분리 장치일 수 있다

중간 테이블을 만들면 pipeline 단계는 하나 늘어납니다.

그래서 처음에는 복잡해 보일 수 있습니다.

하지만 그 단계가 다음 비용을 분리해 준다면 충분히 가치가 있습니다.

```text
원천 포맷 해석 비용
schema 정리 비용
scan 비용
aggregation 비용
downstream ingestion 비용
```

중간 테이블의 목적은 "테이블 하나 더 만들기"가 아니라, 반복되는 무거운 query가 원천 데이터의 제약을 매번 떠안지 않게 하는 것입니다.

따라서 "비용 분리"는 비용 제거가 아닙니다.

원천 CSV 스캔 비용은 여전히 존재합니다.

다만 그 비용을 매 aggregation마다 반복 지불하지 않고, 통제된 intermediate 생성 단계에서 한 번 지불하도록 옮기는 것입니다.

이 구조가 맞는지는 다음 질문으로 판단할 수 있습니다.

```text
같은 raw data를 여러 downstream job이 반복해서 읽는가
intermediate가 여러 query에서 재사용되는가
생성 주기와 소비 주기가 분리되는가
raw schema/type normalization을 한 번만 수행해도 되는가
```

## 8. 정리

이번 사례는 Spark job 튜닝이라기보다, batch pipeline을 어디서 끊어야 하는지에 대한 문제였습니다.

처음 보이는 증상은 단순했습니다.

```text
Airflow task가 timeout 난다
Spark job이 2시간 넘게 돈다
```

하지만 실제 원인은 timeout 값이 작아서도, executor 수가 항상 부족해서도 아니었습니다.

CSV external table을 대규모 aggregation의 직접 입력으로 사용하면서 Spark가 비싼 scan을 반복하는 구조가 핵심이었습니다.

그래서 해결 방향도 parameter 조정이 아니라 pipeline 구조 변경이었습니다.

```text
CSV external table 직접 aggregation 제거
↓
Parquet intermediate table 적재
↓
중간 테이블 기준 aggregation
```

그 결과 실행 시간은 2시간 이상에서 약 20분으로 줄었습니다.

이 사례에서 남는 교훈은 간단합니다.

> Spark job이 느릴 때 executor를 늘리기 전에, 내가 무엇을 얼마나 읽고 있는지부터 봐야 한다.

성능 문제는 코드 한 줄보다 데이터 레이아웃에서 시작되는 경우가 많습니다.

## References

- [Apache Spark SQL Parquet Files][spark-parquet].
- [Apache Spark SQL Performance Tuning][spark-performance].
- [Mary Shaw, ICSE 2003][shaw-icse-2003].

[spark-parquet]: https://spark.apache.org/docs/latest/sql-data-sources-parquet.html
[spark-performance]: https://spark.apache.org/docs/latest/sql-performance-tuning.html
[shaw-icse-2003]: https://www.cs.cmu.edu/~Compose/shaw-icse03.pdf
