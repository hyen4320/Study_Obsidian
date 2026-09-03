ORDER BY는 결과를 어떤 순서로 보여줄지 정렬하는 절 SELECT 문에서 가장 마지막에 실행됨.

정렬기준3가지
- 컬럼명
- 별칭
- N번째 컬럼(정수)

다중정렬가능

```sql
-- 부서별로 묶고, 같은 부서 안에서는 급여 높은 순
SELECT NAME, DEPT_ID, SAL
FROM   EMP
ORDER BY DEPT_ID ASC, SAL DESC;
```

ORDER BY는 NULL이 DBMS마다 위치가 다름
ORACLE-NULL마지막
MS_SQL-NULL 처음