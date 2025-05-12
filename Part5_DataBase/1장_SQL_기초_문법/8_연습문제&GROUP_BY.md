## 연습문제

```sql
USE BaseballData

-- playerID (선수 ID)
-- yearID (시즌 년도)
-- teamID (팀 명칭, 'BOS' = 보스턴)
-- G_batting (출전 경기 + 타석)

-- AB (타수)
-- H (안타)
-- R (출루)
-- 2B (2루타)
-- 3B (3루타)
-- HR (홈런)
-- BB (불넷)

SELECT *
FROM batting;

-- 1) 보스턴 소석 선수들의 정보들만 모두 출력
SELECT *
FROM batting
WHERE teamID = 'BOS';

-- 2) 보스턴 소속 선수들의 수는 몇명? (단, 중복은 제거)
SELECT COUNT (DISTINCT playerID)
FROM batting
WHERE teamID = 'BOS';
-- 1655

-- 3) 보스턴 팀이 2004년도에 친 홈런 개수
SELECT SUM(HR)
FROM batting
WHERE teamID = 'BOS' AND yearID = 2004;
-- 222

-- 4) 보스턴 팀 소속으로 단일 년도 최다 홈런을 친 사람의 정보 => 2번 검색
SELECT MAX(HR) -- 54
FROM batting
WHERE teamID = 'BOS'

SELECT *
FROM batting
WHERE teamID = 'BOS' AND HR = 54;

-- 알려주신 정답
SELECT TOP 1 *
FROM batting
WHERE teamID = 'BOS'
ORDER BY HR DESC;

SELECT *
FROM players
WHERE playerID = 'ortizda01';

-- GROUP BY

-- Q 2004년도에 가장 많은 홈런을 날린 팀은?
SELECT *
FROM batting
WHERE yearID = 2004
ORDER BY teamID

-- 팀별로 묶어서 뭔가를 분석하고 싶다 -> Grouping
SELECT TOP 1 teamID, COUNT(teamID) AS playercount, SUM(HR) AS homeruns
FROM batting
WHERE yearID = 2004
GROUP BY teamID
ORDER BY homeruns DESC;

-- 2004년도에 200홈런 이상을 날린 팀의 목록?
SELECT teamID, SUM(HR) AS homeruns
FROM batting
WHERE yearID = 2004 
GROUP BY teamID
HAVING SUM(HR) >= 200 -- WHERE랑 굉장히 비슷한 느낌이지만 그룹핑을 한다음에 추가적인 조건을 건다는 느낌
ORDER BY homeruns DESC;

-- GROUP BY를 한 애들은 SELECT문에도 똑같이 그대로 올려서 사용을 할 수 있다

-- 논리적인 순서(실행 순서)
-- FROM  책상에서
-- WHERE 공을(조건)
-- GROUP BY 색상별로 분류해서 
-- HAVING 분류한 다음에 빨간색은 제외하고
-- SELECT  갖고와서 
-- ORDER BY 크기별로 나열해주세요

-- 작성 순서와 논리적인 순서 둘다 암기.

-- Q) 단일년도에 200홈런 이상을 날린 팀?
SELECT teamID, yearID, SUM(HR) AS homeruns
FROM batting
GROUP BY teamID, yearID -- 이 둘을 세트로 해서 그룹핑을 하게됨
ORDER BY homeruns DESC;

-- 그룹핑을 한다고 해서 꼭 하나의 열만 기준으로 그룹핑을 하는게 아니라, 여러개의 열을 조합해가지고 그것들을 하나의 키로 사용할 수 있다.

```

실제로는 이렇게 데이터를 찾는 일은 좋은 일로 찾게 되는 것은 아니다

문제가 일어났거나 버그 상황이 의심 될 때 실질적으로 데이터 베이스 샘플을 받아서 분석을 하게 된다.