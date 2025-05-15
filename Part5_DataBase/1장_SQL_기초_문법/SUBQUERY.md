## SUBQUERY

```sql
USE BaseballData;

-- SubQuery (서브 쿼리/하위쿼리)
-- SQL 명령문 안에 지정하는 하부 SELECT

-- 연봉이 역대급으로 높은 선수의 정보를 추출
SELECT TOP 1 *
FROM salaries
ORDER BY salary DESC;

-- rodrial01
SELECT *
FROM players
WHERE playerID = 'rodrial01'

-- 이것을 한번에 하려면?
-- 단일행 서브쿼리
SELECT *
FROM players
WHERE playerID = (SELECT TOP 1 playerID FROM salaries ORDER BY salary DESC);
-- 내부에서 하위 쿼리가 플레이어 아이디를 추출한 다음에 바로 WHERE문에다 집어 넣어서, 
-- 바깥에 있는 셀렉트 구문이 실행이 되면서 이렇게 우리가 두 단계에 거쳐서 했던 연산을 한번에 처리할수 있다

-- 다중행
-- playerID = 단일값
SELECT *
FROM players
--WHERE playerID = (SELECT TOP 20 playerID FROM salaries ORDER BY salary DESC); -- error, 여러개를 뽑아내고 있는데 어떤애랑 같아야되는지 모호하니까 에러.
WHERE playerID in (SELECT TOP 20 playerID FROM salaries ORDER BY salary DESC); -- 이럴때 in을 사용한다.
-- in이라는 것은 말 그대로 오른쪽 조건에 있는 들어가 있는 값 중에 아무거나 라는 의미를 갖게 됨

-- 서브쿼리는 WHERE에서 많이 사용되지만, 나머지 구문에서도 사용가능.
SELECT (SELECT COUNT(*) FROM players) AS playerCount, (SELECT COUNT(*) FROM batting) AS battingCount; 

-- INSERT에서도 사용 가능
SELECT *
FROM salaries
ORDER BY yearID DESC;

-- INSERT INTO
INSERT INTO salaries
VALUES (2020, 'KOR', 'NL', 'rookiss', (SELECT MAX(salary) FROM salaries))

-- INSERT SELECT 
INSERT INTO salaries
SELECT 2020, 'KOR', 'NL', 'rookiss2', (SELECT MAX(salary) FROM salaries);

-- 다른 테이블에 있는 정보를 가져와서 복사(자주 사용됨.)
INSERT INTO salaries_temp
SELECT yearID, playerID, salary FROM salaries;
 
SELECT *
FROM salaries_temp;

-- 상관관계 서브쿼리
-- EXISTS, NOT EXISTS
-- 당장 자유자재로 사용은 못해도 되니까 -> 기억만 하면 된다.

-- 포스트 시즌 타격에 참여한 선수들 목록
SELECT *
FROM players
WHERE playerID IN (SELECT playerID FROM battingpost);

SELECT *
FROM players
WHERE playerID IN (SELECT playerID FROM battingpost WHERE players.playerID = battingpost.playerID);
-- SELECT playerID FROM battingpost WHERE players.playerID = battingpost.playerID 이제 이부분만 따로 드래그해서 실행할수 없다.
-- 상관관계를 이용했기 때문 

-- 그래서 데이터가 실제로 있냐 없냐를 확인하고 싶을때 유용하게 사용되는게 EXIST, NOT EXISTS이다.
-- 는 있으면 SELECT해서 가져오고, 없으면 스킵

SELECT *
FROM players
WHERE EXISTS (SELECT playerID FROM battingpost WHERE players.playerID = battingpost.playerID);
-- 이렇게 하면 battingpost에 있는 플레이어 아이디와 players에 플레이어의 아이디와 같은 애들이면 SELECT해서 가져온다
-- EXISTS이 IN보다 확장성 높게 사용가능

-- 그리고 원하는 부분을 드래그해서 Ctrl + L을 누르면 
-- sql 서버가 어떤식으로 실행할지 실행 계획을 보여준다
```
![Image](https://github.com/user-attachments/assets/75bdc69b-6ea5-40bd-9383-6be396dad72a)
