두개 이상의 테이블을 키로 연결해서 하나의 결과로 합치는 작업

#### 조인의 종류
동등 조인 - =연산자로 연결
비동등 조인 - =외 연산자로 연결
셀프 조인 - 같은 테이블끼리
외부 조인 - LEFT RIGHT FULL 없는 쪽은 NULL
교차 조인 - 카티션 곱

외부조인
```sql
-- LEFT OUTER (왼쪽 다)
SELECT M.NAME, O.ORDER_ID
FROM   MEMBER M LEFT OUTER JOIN ORDERS O
       ON M.MEMBER_ID = O.MEMBER_ID;

-- Oracle 전통 방식 (+) 기호
SELECT M.NAME, O.ORDER_ID
FROM   MEMBER M, ORDERS O
WHERE  M.MEMBER_ID = O.MEMBER_ID(+);   -- (+)는 매칭 안 될 때 NULL로 보충될 쪽(없을 수 있는 쪽)에 붙임
```

NAME이랑 ORDER_ID랑 연결하는거임
OUTER는 생각하지 말고 LEFT니까 NAME을 기준으로 조인함.

교차조인
```sql
SELECT * FROM A CROSS JOIN B;
-- A 3행, B 2행 → 6행 (3×2)
```

주의사항
조인 조건은 누락하면 카티션 곱 발생
조인 조건은 ON으로 해야함
N+! 테이블에 조인은 N개의 조인 조건이 필요