# Data Sharding

Data Sharding 이란?

- 같은 테이블 스키마를 가진 데이터를 다수의 DB에 분산하여 저장하는 방법을 의미(Partitioning과 동일)
- Vertical Sharding(Partitioning)

![Untitled](https://raw.githubusercontent.com/rottoy/TIL/main/assets/data-sharding-01.png)

- Horizontal Sharding(Partitioning)

![Untitled](https://raw.githubusercontent.com/rottoy/TIL/main/assets/data-sharding-02.png)

샤딩을 운영한다는 것은?

- 프로그래밍, 운영적인 복잡도는 더 높아진다는 단점
- 따라서, Sharding 을 적용하기전에 피하거나 지연시킬수 있는 방법을 우선 찾아볼 것
- 회피 방법은 다음과 같음
    - Scale-in(Hardware spec이 더 좋은 컴퓨터를 사용)
    - Read 부하가 크다면?
        - Cache나 Database의 replication 적용
    - Table의 일부 컬럼만 사용한다면?
        - Vertical Partitioning
        - Data를 Hot,Cold,Warm 데이터로 분리

샤딩(Sharding) 방법에 대해

- Shard Key를 어떻게 정의하느냐에 따라 데이터를 효율적으로 분산시키는 것이 결정됩니다.

Hash Sharding

![Untitled](https://raw.githubusercontent.com/rottoy/TIL/main/assets/data-sharding-03.png)

- Shard key : Database id를 Hashing 하여 결정함
    - Hash 크기는 클러스터 안의 노드 개수를 기준으로 함
- 아주 간단한 샤딩 기법
- 단점
    - 클러스터에 포함된 노드 갯수가 변할 경우, **Hash 크기**가 변하게 되고 **Hasy key** 또한 변함
    - 따라서 Hash key에 의한 Data 분산 규칙이 어긋나게 되고 resharding이 필요함.
- 각 공간마다의 부하가 다르게 설정될 수 있다.