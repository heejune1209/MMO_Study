## Merge 조인

Merge(병합) = Sort Merge(정렬 병합) 조인

실제로 구현 되는 방식이 정렬 후 병합이기 때문에 참고 삼아 Sort Merge로 기억하는 것이 좋다.

```csharp
using System;
using System.Collections.Generic;

class Program
{
    class Player : IComparable<Player>
    {
        public int playerId;

        public int CompareTo(Player other)
        {
            if (playerId == other.playerId)
                return 0;
            return (playerId > other.playerId) ? 1 : -1;
        }
    }
    class Salary : IComparable<Salary>
    {
        public int playerId;
        // ...
        public int CompareTo(Salary other)
        {
            if (playerId == other.playerId)
                return 0;
            return (playerId > other.playerId) ? 1 : -1;
        }
    }
    static void Main()
    {
        List<Player> players = new List<Player>();
        players.Add(new Player() { playerId = 0 });
        players.Add(new Player() { playerId = 9 });
        players.Add(new Player() { playerId = 1 });
        players.Add(new Player() { playerId = 3 });
        players.Add(new Player() { playerId = 4 });

        List<Salary> salaries = new List<Salary>();
        salaries.Add(new Salary() { playerId = 0 });
        salaries.Add(new Salary() { playerId = 5 });
        salaries.Add(new Salary() { playerId = 0 });
        salaries.Add(new Salary() { playerId = 2 });
        salaries.Add(new Salary() { playerId = 9 });

        // 1단계) Sort (이미 정렬되어 있으면 Skip)
        // O(N * Log(N))
        players.Sort();
        salaries.Sort();

        // One-To-Many(outer(players)는 중복이 없다)
        // 2단계) Merge
        // outer players [0, 1, 3, 4, 9] -> N
        // inner salaries [0, 0, 2, 5, 9] -> M
        
        // players와 salaries의 커서 => 현재 위치 추적용
        int p = 0;
        int s = 0;

        List<int> result = new List<int>();

        // O(N + M) 
        // => 시간 복잡도에 가장 영향을 미치는 연산은 결국 while문이고
        // 가장 최악의 경우를 생각해봤을 때 players의 개수와 salaries의 개수만큼 돌기 때문에
        // 위와 같은 시간 복잡도가 나오게 된다.
        // 각 커서가 데이터 밖으로 벗어나면 끝낸다
        while(p < players.Count && s < salaries.Count)
        {
            if (players[p].playerId == salaries[s].playerId)
            {
                result.Add(players[p].playerId); // 성공!
                s++;
            }
            else if (players[p].playerId < salaries[s].playerId)
            {
                // outer가 더 작으면 players의 커서를 이동
                p++;
            }
            else
            {
                // inner가 더 작으면 salaries의 커서를 이동
                s++;
            }
        }

        // Many-To-Many(outer(players)는 중복이 있다)
        // 시간 복잡도 => O(N * M)
        // 최악의 경우는?
        // outer players [0, 0, 0, 0, 0] -> N
        // inner salaries [0, 0, 0, 0, 0] -> M
        // 즉 outer에 중복이 가능해지면 많이 느려지게 된다.
    }
}
```

 

Merge가 어떻게 되는지 확인하는 방법(⇒ One-To-Many or Many-To-Many)

1. 실행 계획에서 확인
    
    ![image](https://user-images.githubusercontent.com/75019048/138374206-1e5ef20e-5dd3-44a0-94f4-658e1739d4b1.png)    
    
2. PROFILE에서 확인
    
    ![image](https://user-images.githubusercontent.com/75019048/138374231-ee14d018-91df-4875-8300-d3fb71505aca.png)
    

```sql
USE BaseballData;

SET STATISTICS TIME ON;
SET STATISTICS IO ON;
SET STATISTICS PROFILE ON;

-- NonClustered
--   1
-- 2 3 4

-- Clustered 
--   1
-- 2 3 4

-- Heap Table[{Page} {Page} {Page}]

-- Merge(병합) = Sort Merge(정렬 병합) 조인

 SELECT *
 FROM players AS p
		INNER JOIN salaries AS s
		ON p.playerID = s.playerID;

-- One-To-Many(outer가 unique해야 함 => PK, Unique)
-- Merge 조인도 조건이 붙는다 
-- 일일히 Random Access -> Clustered Scan 후 정렬

SELECT *
FROM schools AS s
		INNER JOIN schoolsplayers AS p
		ON s.schoolID = p.schoolID;

-- 오늘의 결론 --
-- Merge -> Sort Merge
-- 1) 양쪽 집합을 Sort(정렬)하고 Merge(병합)한다.
		-- 이미 정렬된 상태라면 Sort는 생략 (특히, Clustered로 물리적으로 정렬된 상태라면 Best)
		-- 정렬할 데이터가 너무 많으면 GG -> Hash 사용이 더 좋음
-- 2) Random Access 위주로 수행되진 않는다.
-- 3) Many-To-Many 보다는 One-To-Many 조인에 효과적 => One: outer쪽 정보
		-- Join을 하는 조건에 Primary Key, UNIQUE Constraint가 붙었을 때 효과적
```
# Merge(정렬 병합) 조인 핵심 요약

## 1. 개념
- **Merge 조인(Sort Merge)**  
  - 양쪽 테이블을 **정렬(Sort)** 후, 두 커서를 이동하며 **병합(Merge)**  
  - `IComparable` 인터페이스처럼 비교 로직에 따라 작동  

## 2. 동작 단계
1. **정렬 단계**  
   - 양쪽 입력을 `O(N·logN)` 또는 `O(M·logM)`으로 정렬  
   - 이미 정렬된 상태(클러스터드 인덱스)면 생략 가능  
2. **병합 단계**  
   - 두 리스트의 커서를 `O(N+M)`만큼 이동  
   - `playerId`가 같으면 결과 추가, 작으면 작은 쪽 커서를 이동  

## 3. 성능 포인트
- **One-To-Many(1:N)**  
  - OUTER 쪽이 고유(Primary Key/Unique)일 때 효율적  
- **Many-To-Many(N:M)**  
  - 중복 많으면 최악 **O(N·M)** → 성능 저하  
- **정렬 비용**  
  - 입력이 크면 **Hash Join** 고려  
- **랜덤 액세스 최소화**  
  - 키 룩업 없이 순차 스캔 위주  

## 4. SQL 예시
```sql
USE BaseballData;
SET STATISTICS IO, TIME, PROFILE ON;

-- 기본 Merge 조인
SELECT *
FROM players AS p
INNER JOIN salaries AS s
  ON p.playerID = s.playerID;

-- One-To-Many 조인 예시 (schools.pk = schoolsplayers.fk)
SELECT *
FROM schools AS s
INNER JOIN schoolsplayers AS p
  ON s.schoolID = p.schoolID;
```
## 결론
- Merge 조인은 정렬(Sort) + 병합(Merge) 방식
- One-To-Many 상황에서 최적, Many-To-Many일 때는 성능 저하
- 클러스터드 인덱스로 미리 정렬된 테이블 활용 시 비용 절감
- 대용량 정렬 비용이 크면 Hash Join 고려
