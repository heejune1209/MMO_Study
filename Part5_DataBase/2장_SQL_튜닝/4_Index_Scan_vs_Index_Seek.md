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

| 구분     | Index Scan                                             | Index Seek                                                        |
| ------ | ------------------------------------------------------ | ----------------------------------------------------------------- |
| 정의     | 인덱스(또는 클러스터드 테이블)의 **모든(또는 상당 부분) 페이지**를 순차적으로 읽음      | 인덱스에서 **특정 키 값(범위)** 에 해당하는 페이지만 직접 찾아 읽음                         |
| URL    | ☑️ 전체 스캔<br>100% 순차 I/O                                | ☑️ 포인트/범위 탐색<br>순차+랜덤 I/O                                         |
| 적합 상황  | • 결과 행이 많거나 정렬 후 일부 TOP N 조회<br>• 통계상 Scan이 비용이 더 낮을 때 | • 결과 행이 적고, 인덱스 선택도가 높을 때 (선택도(selectivity) 높음)                   |
| I/O 비용 | 높음 (읽은 페이지 수 ≒ 전체 페이지 수)                               | 낮음 (읽은 페이지 수 ≒ 결과와 키 룩업 수)                                        |
| 추가 비용  | –                                                      | • NonClustered → **Key Lookup** 발생 가능<br>• Bookmark Lookup 추가 I/O |
