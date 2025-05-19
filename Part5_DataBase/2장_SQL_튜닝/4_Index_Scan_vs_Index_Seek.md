## Index Scan vs Index Seek

```sql
USE Northwind;

-- 인덱스 접근 방식 (Access)
-- Index Scan vs Index Seek

CREATE TABLE TestAccess
(
		id INT NOT NULL,
		name NCHAR(50) NOT NULL,
		dummy NCHAR(1000) NULL
);

GO

CREATE CLUSTERED INDEX TestAccess_CI
ON TestAccess(id);
GO

CREATE NONCLUSTERED INDEX TestAccess_NCI
ON TestAccess(name);
GO

DECLARE @i INT;
SET @i = 1;

WHILE (@i <= 500)
BEGIN
		INSERT INTO TestAccess
		VALUES (@i, 'Name' + CONVERT(VARCHAR, @i), 'Hello World ' + CONVERT(VARCHAR, @i));
		SET @i = @i + 1;
END

-- 인덱스 정보
EXEC sp_helpindex 'TestAccess'

-- 인덱스 번호
SELECT index_id, name
FROM sys.indexes
WHERE object_id = object_id('TestAccess');

-- 조회
DBCC IND('Northwind', 'TestAccess', 1);
DBCC IND('Northwind', 'TestAccess', 2);

-- CLUSTERED(1) : id
--				1001
-- 1000 1002 1003 ~ 9215 (167)

-- NONCLUSTERED(2) : name
--				9097
-- 1104 1105 1106 ~ 9103 (13)

-- 논리적 읽기 -> 실제 데이터를 찾기 위해 읽은 페이지 수
SET STATISTICS TIME ON;
SET STATISTICS IO ON;

-- INDEX SCAN => LEAF PAGE 순차적으로 검색
-- 실행 결과 : 논리적 읽기 -> 169, 경과 시간 = 51ms
SELECT * 
FROM TestAccess;

-- INDEX SEEK
-- 실행 결과 : 논리적 읽기 -> 2 , 경과시간 = 0ms
SELECT *
FROM TestAccess
WHERE id = 104;

-- INDEX SEEK + KEY LOOKUP
-- 실행 결과 : 논리적 읽기 -> 4 , 경과시간 = 0ms
SELECT *
FROM TestAccess
WHERE name = 'name5';

-- INDEX SEEK + KEY LOOKUP으로 데이터를 찾는 과정
-- 우선 name으로 찾을 때
-- NONCLUSTERED(2) : name
--				9097
-- 1104 1105 1106 ~ 9103 (13)
-- 위에서 9097에 들어간 다음에(1) 거기서 name5에 해당하는 ID를 찾고(2)
-- CLUSTERED(1) : id
--				1001
-- 1000 1002 1003 ~ 9215 (167)
-- 다시 CLUSTERED INDEX로 저장된 1001 page로 넘어가서(3)
-- 여기서 해당 ID값이 어디에 있는지 찾는 과정(4)을 거치기 때문에 
-- 논리적 읽기가 4이다.
-- 위에 1~2까지가 Index Seek과정이고 3~4까지가 Key Lookup이다.

-- INDEX SCAN + KEY LOOKUP
-- 실행 결과 : 논리적 읽기 -> 16 , 경과시간 = 11ms
SELECT TOP 5 *
FROM TestAccess
ORDER BY name;

-- INDEX SCAN이 뜨면 인덱스를 활용하지 못하는 경우이기 때문에
-- 좋지 않는 방법이라고 했었지만 경우에 따라서는 마냥 안좋은 방법은 아니다.
-- 바로 위 경우가 그러하다.
-- 이유는 바로 ORDER BY와 TOP 5에 있다.
-- ORDER BY를 건 조건이 name(NONCLUSTERED)이기 때문에 
-- 이미 LEAF 데이터는 정렬이 완료가 되어 있고 거기서 5개의 데이터만 뽑아오면 되기 때문에 
-- 효율적으로 데이터를 찾아내게 된 것이다.
```

![image](https://user-images.githubusercontent.com/75019048/138374107-446958e2-1dae-4e8f-8b36-4d52a8847f46.png)

# Index Scan vs Index Seek의 핵심 개념과 활용 가이드

## 1. 기본 개념 정리

| 구분     | Index Scan                                             | Index Seek                                                        |
| ------ | ------------------------------------------------------ | ----------------------------------------------------------------- |
| 정의     | 인덱스(또는 클러스터드 테이블)의 **모든(또는 상당 부분) 페이지**를 순차적으로 읽음      | 인덱스에서 **특정 키 값(범위)** 에 해당하는 페이지만 직접 찾아 읽음                         |
| URL    | ☑️ 전체 스캔<br>100% 순차 I/O                                | ☑️ 포인트/범위 탐색<br>순차+랜덤 I/O                                         |
| 적합 상황  | • 결과 행이 많거나 정렬 후 일부 TOP N 조회<br>• 통계상 Scan이 비용이 더 낮을 때 | • 결과 행이 적고, 인덱스 선택도가 높을 때 (선택도(selectivity) 높음)                   |
| I/O 비용 | 높음 (읽은 페이지 수 ≒ 전체 페이지 수)                               | 낮음 (읽은 페이지 수 ≒ 결과와 키 룩업 수)                                        |
| 추가 비용  | –                                                      | • NonClustered → **Key Lookup** 발생 가능<br>• Bookmark Lookup 추가 I/O |

## 2. 언제 Scan, 언제 Seek?
- 2.1 Index Scan
  - 전체 읽기가 빠른 경우
    - 작은 테이블, 또는 ORDER BY … TOP N 처럼 정렬된 Leaf에서 바로 일부 읽을 때

  - 통계상 비용이 더 낮은 경우
    - 옵티마이저가 io cost·cpu cost 비교 결과 Scan 선택

  - 인덱스 선택도가 낮을 때
    - WHERE 조건이 포괄적(예: Age > 0), 결과 행이 많음
      
- 2.2 Index Seek
  - 선택도가 높은 조건
    - WHERE id = ?, WHERE name = 'XYZ' 같은 고유(또는 좁은 범위) 검색
  - 범위 검색
    - BETWEEN, >= 등으로 특정 구간만 읽을 때
  - 대용량 테이블
    - 전체 스캔 비용이 클 때 포인트 접근으로 I/O 절감

## 3. 비용(Statistics & Cost) 이해
- 논리적 읽기(Logical Reads)
  - 읽은 메모리 페이지 수. Scan은 전체 페이지, Seek는 결과+키룩업만.
- 물리적 읽기(Physical Reads)
  - 메모리에 없는 페이지를 디스크에서 읽을 때.(디스크에서 읽은 페이지 수)
- CPU 시간
  - Scan은 순차 처리지만, Seek+Key Lookup은 랜덤 I/O로 오버헤드 존재.
- 비용 모델
  - 옵티마이저가 Estimated I/O Cost + CPU Cost 비교 후 플랜 선택

## 4. Key Lookup(Bookmark Lookup) 고려사항
- 발생 조건
  - NonClustered Seek 후, 출력 컬럼이 인덱스에 없을 때
- 해결책
  - Covered Index: INCLUDE로 필요한 모든 컬럼 포함
  - 클러스터드 인덱스 설계: 출력 컬럼을 PK로 포함
  - 쿼리 수정: 필요한 컬럼 최소화
```sql
-- 예: Key Lookup 제거를 위한 COVERED INDEX
CREATE NONCLUSTERED INDEX IX_Cov
  ON TestAccess(name)
  INCLUDE (dummy);
```
## 5. 최적화 전략
1. 통계(Statistics) 최신화
   - UPDATE STATISTICS로 카드INALITY 예측 정확도 개선
2. 필터 컬럼 Selectivity 체크
   - sys.dm_db_index_physical_stats, sys.dm_db_stats_histogram 활용
3. Covered/Include 인덱스 설계
4. 쿼리 힌트
   - 강제 Scan/Seek: OPTION (FORCE ORDER), WITH (INDEX(...))
   - 주의: 운영환경에서 남용 금지
5. TOP N + ORDER BY 최적화
   - 정렬 키로 NonClustered Index 존재 시 Scan + TOP N이 오히려 빠름

## 6. 실전 팁
- 작은 테이블은 Scan이 더 빠른 경우 다수
- 빈번한 단건 조회는 Seek + Covered Index
- 대량 처리 배치는 Scan 후 메모리 정렬이 좋을 수 있음
- 옵티마이저 플랜 캐시 확인: sys.dm_exec_query_plan
- 인덱스 남발 경고: DML(Insert/Update/Delete) 성능 저하 고려

## 7. 요약 및 결론
- Index Scan
  - 전체(또는 대량) 페이지 순차 스캔 → 정렬+TOP N에 강점

- Index Seek
  - 포인트/범위 탐색 → 선택도 높은 검색에 최적

- Key Lookup 비용 관리**: Covered/Include 인덱스 활용
- 통계 기반 비용 모델 이해 → 실행 계획 분석 필수
- 운영 환경 테스트: 실제 I/O·CPU 프로파일링으로 최적화 검증
