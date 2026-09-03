행 필터, AND/OR 우선순위

#### 연산자
- 비교 연산자
- 논리 연산자
- SQL 연산자
- LIKE 연산자와 와일드카드

SQL 연산자 4종
- BETWEEN A AND B
	- A<=X<=B
- IN (a,b,c) a,b,c를 포함, 이게 x=a orx=b 이런 식으로 체인이 풀림
	- 값 목록 또는 서브쿼리 가능
	- NOT 은 and로 풀림

LIKE 연산자의 와일드카드
- %는 0개 이상의 문자
- _ 는 정확히 한 개의 문자

NULL 과의 비교
NULL을 =이나 !=으로 비교하면 UNKNOWN이 됨
항상 IS 나 IS NOT으로 비교해줘야 함
