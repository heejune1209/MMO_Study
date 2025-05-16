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

# Index Access: Scan vs Seek 사고 흐름 정립용 키워드

1. **통계 설정**  
   - `SET STATISTICS IO ON` → **Logical Reads** 확인  
   - `SET STATISTICS TIME ON` → **Elapsed Time** 확인

2. **Index Scan**  
   - **개념**: Leaf 페이지를 순차(Full) 스캔  
   - **키워드**: LEAF Page 순차 검색, 논리적 읽기(높음), 전체 스캔  
   - **발생 조건**:  
     - 필터 조건 없음 (`SELECT * FROM TestAccess`)  
     - `ORDER BY name` + `TOP N` on Non-Clustered Index → 이미 정렬된 NCI를 활용해 효율적  

3. **Index Seek**  
   - **개념**: B-Tree 탐색으로 해당 키에 바로 점프  
   - **키워드**: 인덱스 키 탐색, 논리적 읽기(낮음), 특정 값 조회  
   - **발생 조건**:  
     - 직접 비교(=) 필터 (`WHERE id = 104`)

4. **Key Lookup**  
   - **개념**: Non-Clustered Seek 후, 실제 데이터(Clustered) 조회  
   - **키워드**: Seek + Bookmark Lookup, 추가 페이지 접근  
   - **발생 조건**:  
     - Non-Clustered Index on `name` + SELECT * (추가 컬럼 필요)  
     - 순서:  
       1. NCI Seek → `name='name5'` 의 Leaf Page (PagePID=9097) 접근  
       2. 거기서 **Clustered Key(id)** 획득  
       3. CI B-Tree Seek → Data Page 접근 (PagePID=1001)  
       4. 해당 슬롯에 있는 행 읽기

5. **사고 흐름 연결 예시**  
   1. **필터 유무 확인** → Scan vs Seek 판단  
   2. **Seek 발생 시** → NCI인지 CI인지 확인 → 필요시 Key Lookup  
   3. **ORDER BY/ TOP N** → 이미 정렬된 인덱스 활용하면 Scan도 효율적  
   4. **통계 결과** → Logical Reads & Elapsed Time으로 계획 평가  

각 단계에서 위 **키워드**를 떠올리며, “통계 설정 → Scan vs Seek → Key Lookup → 계획 평가” 순으로 머릿속에 그려보세요.  

