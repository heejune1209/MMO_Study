## Sorting

```sql
USE BaseballData;

-- 1) 생략
-- 2) ORDER BY

-- 정렬이 되는 원인) ORDER BY 순서대로 정렬을 해야 하니깐
SELECT *
FROM players
ORDER BY college;

-- INDEX가 잡혀 있는 순서대로 정렬을 하면 실제로는 Sorting을 하지 않는다
SELECT *
FROM batting
ORDER BY playerID, yearID; 

-- 3) GROUP BY
-- 원인) 집계를 하기 위해 정렬을 먼저 실행
SELECT college, COUNT(college)
FROM players
WHERE college LIKE 'C%'
GROUP BY college;

SELECT playerID, COUNT(playerID)
FROM players
WHERE playerID LIKE 'C%'
GROUP BY playerID; -- playerID가 INDEX로 잡혀 있기 때문에 Sorting을 안한다.

-- 4) DISTINCT
-- 원인) 중복을 제거하기 위해서 정렬
-- 정말 중복 제거가 필요 없는 경우에는 굳이 쓸 필요가 없다.
SELECT DISTINCT college
FROM players
WHERE college LIKE 'C%';

-- 5) UNION
-- 원인) 두 데이터를 합칠 때 중복된 데이터가 있을 때는 한 번만 나오게 하기 위해서 
-- => 중복 제거 DISTINCT
-- UNION의 경우 중복이 일어나지 않을 것이란 확신이 있다면(ex. 아래 경우와 같이 B, C로 시작하면 중복x)
-- UNION ALL을 사용해서 Sorting을 안할 수도 있다.
-- 즉 중복이 없다면 UNION ALL을 사용하면 성능을 향상 시킬 수 있다.
SELECT college
FROM players
WHERE college LIKE 'B%'
UNION ALL
SELECT college
FROM players
WHERE college LIKE 'C%';

-- 6) 순위 윈도우 함수
-- 원인) 집게를 하기 위해서
SELECT ROW_NUMBER() OVER (ORDER BY college)
FROM players;

-- 인덱스를 활용하면 sort가 사라진것을 알수 있다
SELECT ROW_NUMBER() OVER (ORDER BY playerId)
FROM players;

-- 오늘의 결론 --
-- Sorting (정렬)을 줄이자!

-- O(N * LogN) -> DB는 데이터가 어마어마하게 많다.
-- 너무 용량이 커서 가용 메모리로 커버가 안 되면 -> 디스크까지 찾아간다.
-- 따라서 디스크까지 가서 Sorting을 하게 되면 너무 느리기 때문에
-- Sorting이 언제 일어나는지 파악하고 있어야 함

-- Sorting이 일어날 때
-- 1) SORT MERGE JOIN
		-- 원인) 알고리즘 특성상 Merge하기 전에 Sort를 해야 함
-- 2) ORDER BY
-- 3) GROUP BY
-- 4) DISTINCT
-- 5) UNION
-- 6) RANKING WINDOWS FUNCTION
-- 7) MIN MAX
-- 8) 인덱스가 없어서 정렬된 데이터가 없었을 경우

-- INDEX를 잘 활용하면, Sorting을 굳이 하지 않아도 된다 
-- 또는 Sorting 자체가 정말 필요한지 고민을 해봐야 한다.
```
## 정렬(Sort) 발생 주요 원인
- **SORT MERGE JOIN**  
  - Merge 조인 전 양쪽 테이블 정렬  
- **ORDER BY**  
  - 지정된 컬럼 순서대로 결과 정렬  
- **GROUP BY**  
  - 집계 전 그룹별 정렬  
- **DISTINCT**  
  - 중복 제거를 위한 정렬  
- **UNION**  
  - 중복 제거 시 정렬  
  - 중복 없으면 `UNION ALL` 사용해 정렬 생략 가능  
- **윈도우 함수 (ROW_NUMBER, RANK 등)**  
  - 순위 계산 전 정렬  
- **MIN / MAX**  
  - 인덱스 없으면 정렬 수행  
- **인덱스 부재**  
  - 사전 정렬된 데이터 없으면 임시 정렬 발생  

---

## 인덱스 활용으로 정렬 회피
- **정렬 키에 클러스터드/논클러스터드 인덱스 존재**  
  - 인덱스 순서대로 이미 정렬된 데이터 사용 → Sort 연산 생략  
- **`UNION ALL` 사용**  
  - 중복 없으면 정렬 없이 결과 합병  
- **조인 전략 변경**  
  - Merge Join 대신 Hash/Nested Loop 등으로 Sort 비용 회피  

---

## 최적화 팁
1. **정렬 구문 검토** → 불필요한 `ORDER BY`·`DISTINCT` 제거  
2. **인덱스 커버링**  
   ```sql
   CREATE INDEX IX_Covered
     ON 테이블(정렬컬럼)
     INCLUDE(출력컬럼…);
   ```
3. UNION ALL 활용 → 중복 없으면 정렬 제거
4. 메모리/디스크 I/O 고려
   - 대량 데이터 정렬 시 버퍼 풀·디스크 스페이스 확인
5. 실행 계획 확인
   - Sort (Top N) 연산 아이콘 식별

## 결론
- 정렬은 O(N·logN) → 대규모 데이터에서 비용 급증
- 정렬 발생 원인을 정확히 파악하고,
- 인덱스 활용 또는 쿼리 리팩토링으로 Sort 연산을 최소화하여 성능 최적화
