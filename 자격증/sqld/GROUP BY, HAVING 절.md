그룹화 WHERE vs HAVING

WHERE vs HAVING
실행순서: FROM->WHERE->GROUP BY-> HAVING->SELECT->ORDER BY

GROUP BY 핵심 규칙
GROUP BY에 명시되지 않은 컬럼은 SELECT 절에 그대로 쓸 수 없다. 집계함수로만 사용 가능
```sql
-- 잘못된 쿼리
SELECT DEPT_ID, NAME, AVG(SAL)   -- NAME이 GROUP BY에 없음
FROM   EMP
GROUP BY DEPT_ID;

-- 옳은 쿼리
SELECT DEPT_ID, AVG(SAL)
FROM   EMP
GROUP BY DEPT_ID;
```

### WHERE vs HAVING
WHERE 행 단위 필터 적용시점: 그룹화 전 집계함수 불가
HAVING 그룹 단위 필터 적용시점: 그룹화 후 집계함수 가능
