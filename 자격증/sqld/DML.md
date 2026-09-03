INSERT / UPDATE / DELETE / MERGE

데이블 안의 데이터를 입력 수정 삭제 조회하는 명령어들이다.

INSERT 노트에 새 줄 적기
UPDATE 기존 행 수정
DELETE 행 삭제
MERGE 있으면 UPDATE 없으면 INSERT

UPDATE 테이블 SET 컬럼 = 값
DELETE FROM 테이블 ; 모든 행 삭제(구조는 유지)

TRUNCATE
모든 데이터 삭제 테이블 구조 유지 자동 COMMIT

DELETE 모든 데이터 삭제 행단위로 삭제해서 느림

DROP 테이블 자체를 삭제