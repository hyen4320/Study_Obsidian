https://news.hada.io/topic?id=33576
Pandas는 느림
Pandas->Polars,DockDB->Spark,DataBricks,snowflaske,Dask
이 순서로 대응해야함. 

Apache Arraow를 사용하면 Pandas, DuckDB, Polars를 섞어서 쓸 수 있음

100GB가 넘어가면 Pandas를 버리고 넘어가야함. 그 전에 넘어가도 좋고

이는 Pandas의 실행 매커니즘에 있음.

Pandas:

Polars:

DuckDB:





아마존의 논문-쿼리 실행 시간, 테이블 크기 통계를 공개
https://www.amazon.science/publications/why-tpc-is-not-enough-an-analysis-of-the-amazon-redshift-fleet