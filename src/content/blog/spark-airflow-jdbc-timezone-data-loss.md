---
title: "Spark 배치에서 KST와 UTC가 섞이면 왜 00시 데이터가 사라지는가"
description: "Spark batch job에서 JDBC serverTimezone, JVM timezone, Hive SerDe, Airflow schedule 기준이 어긋나며 특정 시간대 데이터가 조용히 누락된 문제를 정리했습니다."
category: "Data Engineering"
pubDate: "2026-03-31T00:00:00+09:00"
---

## TL;DR

Spark batch job은 성공했지만, 특정 시간대 데이터가 빠질 수 있습니다.

이번 문제는 job failure가 아니라 **silent data loss**에 가까웠습니다.

겉으로 보이는 현상은 다음과 같았습니다.

```text
09시~23시 데이터는 정상 적재
00시~08시 데이터는 누락
```

job 자체는 성공했습니다.

그래서 더 위험했습니다.

```text
Airflow task: success
Spark job: success
Hive table: 일부 시간대 누락
```

원인은 단일 설정 하나가 아니었습니다.

여러 계층의 timezone 기준이 서로 달랐습니다.

```text
Airflow schedule timezone
JVM timezone
JDBC serverTimezone
Timestamp 생성 방식
Hive SerDe timezone
business date 기준
```

이 글의 주장은 하나입니다.

> Batch job에서 날짜 필터가 `now()`와 runtime timezone에 의존하면, job은 성공해도 특정 시간대 데이터가 조용히 누락될 수 있다. schedule 기준 날짜는 Airflow에서 명시적으로 주입하고, 저장 timezone과 비즈니스 timezone을 분리해서 다뤄야 한다.

## 1. 문제 상황

어떤 hourly snapshot job이 있었습니다.

이 job은 갱신된 데이터를 읽어서 시간 단위 snapshot table에 적재합니다.

단순화하면 다음과 같은 구조였습니다.

```text
source database
↓
Spark JDBC read
↓
filter by updated timestamp
↓
Hive snapshot table
```

필터의 의도는 단순했습니다.

```text
오늘 00시 이후 갱신된 데이터만 snapshot에 포함한다.
```

하지만 실제 결과는 이상했습니다.

```text
09시~23시 구간: 정상
00시~08시 구간: 누락
```

특정 시간대만 빠지는 패턴은 대개 우연이 아닙니다.

특히 9시간 차이가 보인다면 KST와 UTC가 섞였을 가능성을 먼저 의심해야 합니다.

## 2. 왜 job failure보다 위험한가

Airflow task가 실패했다면 차라리 빨리 알 수 있습니다.

```text
task failed
alert fired
operator checks logs
```

하지만 이번 문제는 달랐습니다.

```text
task succeeded
data written
only some hours missing
```

이런 문제는 늦게 발견됩니다.

downstream job도 "테이블이 존재한다"는 이유로 계속 진행할 수 있습니다.

운영자 입장에서는 더 나쁩니다.

```text
pipeline success
dashboard partial data
business logic wrong
alert may not fire
```

그래서 timezone 문제는 단순 버그라기보다 데이터 품질 문제에 가깝습니다.

## 3. 문제를 만든 timezone 레이어

이번 문제를 이해하려면 여러 계층을 나눠 봐야 합니다.

### 3.1 Airflow schedule timezone

Airflow는 job을 언제 실행할지 결정합니다.

하지만 "언제 실행했는가"와 "어느 날짜 데이터를 처리해야 하는가"는 다릅니다.

```text
execution time
≠
business date
```

예를 들어 KST 기준 하루 데이터를 처리해야 한다면 기준은 KST 자정입니다.

```text
KST 00:00
=
UTC previous day 15:00
```

이 값을 job 내부에서 `now()`로 계산하기 시작하면 재처리와 backfill도 어려워집니다.

### 3.2 JVM timezone

Spark job은 JVM 위에서 동작합니다.

따라서 Java/Scala의 시간 API가 JVM timezone 영향을 받을 수 있습니다.

예를 들면 다음 코드는 보기보다 위험합니다.

```java
Timestamp.valueOf(localDateTime)
```

`LocalDateTime` 자체는 timezone을 갖지 않습니다.

그런데 `Timestamp`로 변환되는 순간 runtime timezone 해석이 끼어들 수 있습니다.

반면 다음 방식은 의도를 더 명확하게 드러냅니다.

```java
Timestamp.from(instant)
```

`Instant`는 UTC 기준의 한 시점을 표현합니다.

따라서 timezone 변환이 필요한 지점을 코드에서 더 명시적으로 관리할 수 있습니다.

### 3.3 JDBC serverTimezone

JDBC URL의 `serverTimezone`도 중요합니다.

예를 들어 source database를 JDBC로 읽을 때 다음 설정이 있다고 가정합니다.

```text
serverTimezone=Asia/Seoul
```

그런데 Spark job의 JVM은 UTC로 실행됩니다.

```text
-Duser.timezone=UTC
```

이 조합에서는 timestamp filter 기준과 JDBC가 timestamp 값을 해석하는 기준이 달라질 수 있습니다.

단순화하면 이런 상태입니다.

```text
filter 기준: UTC JVM에서 계산
JDBC timestamp 해석: Asia/Seoul 기준
```

결과적으로 비교 기준이 9시간 밀릴 수 있습니다.

다만 이 부분은 database와 JDBC driver마다 다르게 봐야 합니다.

MySQL의 `serverTimezone` 문제와 PostgreSQL, Oracle의 timestamp 처리 방식은 같지 않습니다.

따라서 이 글의 핵심은 "모든 JDBC driver가 같은 방식으로 동작한다"가 아닙니다.

핵심은 source database, JDBC driver, Spark session, JVM timezone이 timestamp 값을 어느 기준으로 해석하는지 분리해서 확인해야 한다는 점입니다.

운영에서는 최소한 다음을 명시적으로 남기는 편이 안전합니다.

```text
database product / version
JDBC driver artifact / version
timestamp column type
JDBC URL timezone 관련 옵션
Spark spark.sql.session.timeZone
driver/executor JVM user.timezone
```

### 3.4 Hive SerDe timezone

Spark SQL 설정만으로 끝나지 않는 경우도 있습니다.

예를 들어 다음 설정을 했다고 해서 모든 timestamp serialization이 해결되는 것은 아닙니다.

```text
spark.sql.session.timeZone=UTC
```

Hive SerDe가 timestamp를 처리할 때 JVM timezone을 사용한다면, Spark SQL session timezone과 다른 경로로 값이 직렬화될 수 있습니다.

이 문장은 버전과 table format에 따라 확인이 필요합니다.

Hive SerDe, Parquet/ORC datasource, Spark SQL native datasource는 timestamp를 통과시키는 경로가 다를 수 있습니다.

따라서 "Hive SerDe는 항상 JVM timezone을 쓴다"가 아니라, 실제 job이 사용하는 read/write path에서 timestamp가 어디서 변환되는지 확인해야 합니다.

확인 지점은 다음입니다.

```text
table provider
SerDe class
input/output format
Spark SQL session timezone
driver/executor JVM timezone
read/write 양쪽의 timestamp round-trip 결과
```

이 경우 driver와 executor 모두에 JVM timezone을 맞춰야 합니다.

```text
spark.driver.extraJavaOptions=-Duser.timezone=UTC
spark.executor.extraJavaOptions=-Duser.timezone=UTC
```

`TimeZone.setDefault(...)` 같은 방식도 조심해야 합니다.

driver에서만 적용되고 executor에는 적용되지 않으면 distributed job에서 일관성이 깨집니다.

## 4. 잘못된 해결: 그냥 UTC 자정으로 바꾸기

처음 보기에는 모든 것을 UTC로 통일하면 해결될 것처럼 보입니다.

```text
JVM timezone = UTC
JDBC serverTimezone = UTC
filter 기준 = UTC 자정
```

하지만 비즈니스 기준일이 KST라면 이 방식도 틀릴 수 있습니다.

KST 기준 오늘 00시는 UTC로 전날 15시입니다.

```text
KST today 00:00
=
UTC previous day 15:00
```

그런데 필터 기준을 UTC 오늘 00시로 잡으면 어떻게 될까요?

```text
UTC today 00:00
=
KST today 09:00
```

그러면 KST 00:00~09:00에 갱신된 데이터가 필터 밖으로 밀릴 수 있습니다.

```text
원하는 기준
KST 00:00 이후

잘못 잡은 기준
UTC 00:00 이후 = KST 09:00 이후
```

이러면 바로 00시~08시 데이터 공백이 생깁니다.

UTC로 저장하고 처리하는 것은 좋습니다.

하지만 비즈니스 날짜가 KST라면 "KST 자정이라는 business boundary를 UTC instant로 변환한 값"을 사용해야 합니다.

## 5. 더 나은 방향: 기준 날짜를 외부에서 주입하기

이 문제에서 가장 아쉬웠던 부분은 job이 내부에서 현재 시각을 계산했다는 점입니다.

```java
Instant.now()
LocalDate.now()
```

이 방식은 간단하지만 batch job에는 약합니다.

왜냐하면 batch job은 "지금"보다 "어느 schedule interval을 처리하는가"가 더 중요하기 때문입니다.

더 나은 방향은 Airflow가 schedule 기준 날짜를 Spark job에 넘기는 것입니다.

```text
Airflow data interval
↓
business date 계산
↓
Spark job argument로 전달
↓
Spark job은 전달받은 기준일만 사용
```

예를 들면 다음처럼 실행할 수 있습니다.

```bash
spark-submit \
  --conf spark.driver.extraJavaOptions=-Duser.timezone=UTC \
  --conf spark.executor.extraJavaOptions=-Duser.timezone=UTC \
  job.jar \
  --business-date 2026-07-06 \
  --business-timezone Asia/Seoul
```

이렇게 하면 job 내부에서 `now()`를 호출할 이유가 줄어듭니다.

Airflow 쪽에서도 호환성을 확인해야 합니다.

기존 DAG가 runtime clock 기준으로 움직였다면, `business-date`를 외부에서 주입하는 순간 backfill과 retry 의미가 바뀔 수 있습니다.

확인해야 할 것은 다음입니다.

```text
scheduled run의 business-date 계산
manual run에서 business-date 누락 시 동작
backfill 기간의 각 run이 서로 다른 business-date를 받는지
retry가 같은 business-date로 재실행되는지
이미 적재된 partition을 덮어쓸지 append할지
```

이 조건을 맞춰야 "재실행해도 같은 interval을 다시 처리한다"는 batch job의 장점이 살아납니다.

재처리도 쉬워집니다.

```text
오늘 실행
--business-date 2026-07-06

어제 재처리
--business-date 2026-07-05
```

batch job은 deterministic할수록 운영하기 쉽습니다.

## 6. 수정 방향

이번 문제에서 필요한 수정은 한 줄짜리가 아니었습니다.

여러 계층을 함께 맞춰야 했습니다.

### 6.1 timestamp 생성 방식을 명확하게 바꾸기

timezone 없는 `LocalDateTime`을 바로 `Timestamp.valueOf`로 넘기는 패턴은 줄이는 편이 좋습니다.

대신 기준 시점을 `Instant`로 만들고, 그 값을 `Timestamp.from`으로 변환합니다.

```java
ZonedDateTime startOfDay = businessDate
    .atStartOfDay(ZoneId.of("Asia/Seoul"));

Instant startInstant = startOfDay.toInstant();

Timestamp startTimestamp = Timestamp.from(startInstant);
```

이 코드는 의도를 더 명확하게 보여줍니다.

```text
business date: KST 기준
storage/comparison point: UTC instant
```

중요한 점은 이 변경이 한 군데에만 들어가면 부족할 수 있다는 것입니다.

날짜 필터를 만드는 모든 경로를 같이 찾아야 합니다.

```text
WHERE updated_at >= ?
partition key 생성
snapshot hour 계산
backfill 시작/종료 시각 계산
retry 시 기준 시간 재계산
테스트 fixture timestamp 생성
```

한 곳은 `Instant`로 바꿨지만 다른 곳이 여전히 `LocalDateTime.now()`나 `Timestamp.valueOf(...)`를 쓰면, 같은 문제가 다른 경로에서 다시 생길 수 있습니다.

### 6.2 Date utility 이름에 timezone을 드러내기

다음 이름은 위험합니다.

```java
getStartTimeOfToday()
getTodayString()
getLastBaseHour()
```

어느 timezone의 today인지 알 수 없기 때문입니다.

차라리 이름에 기준을 드러내는 편이 낫습니다.

```java
getStartTimeOfTodayKST()
getStartTimeOfTodayUTC()
getStartTimeOfBusinessDateKSTAsInstant()
getLastBaseHourKST()
```

이름이 길어져도 괜찮습니다.

timezone이 숨어 있는 짧은 이름보다, timezone이 드러나는 긴 이름이 안전합니다.

### 6.3 JDBC와 Spark JVM timezone을 함께 본다

JDBC `serverTimezone`만 바꾸거나, Spark JVM timezone만 바꾸면 반쪽짜리 수정이 될 수 있습니다.

함께 봐야 합니다.

```text
JDBC serverTimezone
Spark driver timezone
Spark executor timezone
Hive SerDe timestamp 처리
business date boundary
```

특히 Spark job에서는 driver와 executor가 나뉩니다.

driver에서만 timezone을 맞췄다고 전체 job이 맞는 것은 아닙니다.

## 7. 검증 방법

timezone 버그는 사람이 눈으로 한두 건 확인해서 끝내면 안 됩니다.

테스트 케이스가 시간 경계를 직접 밟아야 합니다.

### 7.1 00시 경계 테스트

가장 중요한 케이스는 자정 경계입니다.

```text
KST 00:00
KST 00:30
KST 08:59
KST 09:00
```

이 값들이 의도한 필터에 포함되는지 확인해야 합니다.

예를 들어 KST business date가 `2026-07-06`이라면, 필터 하한은 UTC instant로 `2026-07-05T15:00:00Z`가 되어야 합니다.

테스트는 단순히 utility return 값을 보는 데서 끝내지 않고, 실제 filter 조건까지 밟는 편이 좋습니다.

```sql
-- business date: 2026-07-06 Asia/Seoul
-- expected lower bound: 2026-07-05T15:00:00Z

SELECT id
FROM source_rows
WHERE updated_at >= :start_instant_utc
ORDER BY id;
```

예상 assertion은 다음처럼 시간 경계를 직접 드러내야 합니다.

```text
KST 2026-07-05 23:59:59 → excluded
KST 2026-07-06 00:00:00 → included
KST 2026-07-06 08:59:59 → included
KST 2026-07-06 09:00:00 → included
```

이렇게 작성해야 UTC 00:00을 잘못 하한으로 잡아서 KST 00시~08시 데이터가 빠지는 회귀를 잡을 수 있습니다.

### 7.2 UTC 변환 round-trip 테스트

KST business date를 UTC instant로 바꾼 뒤 다시 사람이 이해하는 시간으로 확인합니다.

```text
KST 2026-07-06 00:00
→ UTC 2026-07-05 15:00
```

이 변환이 코드에서 명확하게 드러나야 합니다.

### 7.3 Spark local mode 테스트

단순 utility test만으로는 부족할 수 있습니다.

실제 Spark filter logic이 맞는지 local mode로 확인하는 편이 좋습니다.

```text
input rows with updatedAt
↓
Spark filter
↓
expected rows
```

이렇게 해야 DataFrame filter에서 timestamp가 어떻게 해석되는지까지 볼 수 있습니다.

### 7.4 운영 데이터 검증

마지막은 실제 적재 결과입니다.

```text
00~23시 모든 hour partition이 채워졌는가
특정 시간대만 비어 있지 않은가
전날/당일 경계 데이터가 중복 또는 누락되지 않았는가
```

timezone 문제는 평균 row count보다 hour별 분포를 보는 쪽이 더 빠릅니다.

## 8. 체크리스트

비슷한 문제를 다시 만나면 이 순서로 보겠습니다.

```text
1. 누락 패턴이 8~9시간 단위인가
2. KST/UTC 변환 경계와 맞물리는가
3. job은 성공했지만 특정 hour만 비는가
4. LocalDate.now() 또는 Instant.now()를 batch job 내부에서 호출하는가
5. Timestamp.valueOf(LocalDateTime)을 사용하는가
6. JDBC serverTimezone과 JVM timezone이 다른가
7. driver와 executor timezone이 모두 설정되어 있는가
8. Hive SerDe가 Spark SQL timezone을 따르는지 확인했는가
9. Airflow data interval을 job argument로 넘길 수 있는가
10. 00시~09시 경계 테스트가 있는가
```

이 중 하나라도 걸리면 단순 timezone 설정 문제가 아닐 수 있습니다.

batch 기준일 모델을 다시 봐야 합니다.

## 9. 정리

이번 문제는 "UTC로 통일하자"만으로 해결되지 않았습니다.

핵심은 timezone을 하나로 외치는 것이 아니라, 각 레이어의 책임을 나누는 것이었습니다.

```text
Airflow
→ 어떤 business date를 처리할지 결정

Spark job
→ 전달받은 business date를 기준으로 deterministic하게 처리

JDBC
→ source timestamp를 일관된 기준으로 해석

Hive/Spark
→ 저장과 직렬화 timezone을 일관되게 유지
```

특히 batch job에서는 `now()`가 편해 보이지만, 운영에서는 위험할 수 있습니다.

job이 언제 실행됐는지보다 중요한 것은 **어느 데이터 interval을 처리해야 하는지**입니다.

이번 사례의 교훈은 간단합니다.

> Batch job의 날짜 기준은 runtime clock이 아니라 scheduler가 넘겨준 data interval이어야 한다.

Spark job이 성공했다고 데이터가 맞는 것은 아닙니다.

시간대가 섞인 시스템에서는 성공한 job도 조용히 데이터를 잃을 수 있습니다.

## References

- [Apache Spark SQL SET TIME ZONE][spark-set-time-zone].
- [Apache Spark SQL JDBC Data Source][spark-jdbc].
- [Apache Spark SQL Data Types][spark-data-types].
- [Mary Shaw, ICSE 2003][shaw-icse-2003].

[spark-set-time-zone]: https://spark.apache.org/docs/latest/sql-ref-syntax-aux-conf-mgmt-set-timezone.html
[spark-jdbc]: https://spark.apache.org/docs/latest/sql-data-sources-jdbc.html
[spark-data-types]: https://spark.apache.org/docs/latest/sql-ref-datatypes.html
[shaw-icse-2003]: https://www.cs.cmu.edu/~Compose/shaw-icse03.pdf
