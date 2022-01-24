# Designing Instagram

1. 인스타그램이란?

소셜 네트워킹 서비스이며, 유저들 간 업로드 및 사진 비디오를 공유할 수 있다.

인스타그램 유저는 게시물을 사적으로 혹은 공적으로 공유할 수 있다.(사적인 게시물은 팔로워들만 확인할 수 있다.)

기능적 요구사항

1. 유저는 사진을 업로드/다운로드/조회 할 수 있다.
2. 유저는 사진/비디오의 제목에 기반하여 검색할 수 있다.
3. 유저는 다른 유저를 팔로우 할 수 있다.
4. 시스템은 유저에게 자신을 팔로우한 유저들에게 최근 게시물인 뉴스 피드를 제공해야 한다.

비기능적 요구사항

- 고가용성 이어야함.(항상 사용가능 상태여야 함 = 24시간)
- 지연시간 2초
- 신뢰성이 있어야 함.(사진 비디오가 유실되면 안됨)

1. 디자인 고민
- 시스템은 조회가 많을 것이며, 따라서 사진을 빠르게 반환할 수 있는 시스템이여야 함.
    - 실제로, 유저들은 많은 양의 사진들을 업로드할 것이기 때문에 이것을 디자인의 핵심 요소로 반영해야 한다.
    - 사진을 보여줄 때 지연 최소화(=low latency)가 필요하다.
    - 데이터는 100% 신뢰 가능하여야 한다. 만약 유저가 사진을 업로드 하면, 시스템은 사진이 유실되지 않음을 보장해야 한다.
    
1. 용량 산정과 제한 요소
- 500M의 총 유저와 1M의 일일접속 유저가 있다고 가정하자.
- 2M의 새로운 사진이 매일 업로드된다고 가정하면, 평균적으로 초마다 23개의 사진이 처리되어야 한다.
- 사진의 평균 사이즈 : 200KB
- 하루에 업로드 되는 사진의 총 용량 : 400GB
- 10년간 업로드 되는 사진의 총 용량 : 400GB * 365 * 10 ~= 1425TB

1. 고 수준의 시스템 디자인
- 고 수준에서는, 사진 업로드, 사진 조회 2가지 시나리오 모두 생각해봐야 한다. 현재 서비스는 사진을 저장하기 위한 객체 스토리지 서버와 사진의 메타데이터를 저장할 db 서버 2개가 필요하다.
    
    ![스크린샷 2021-09-25 오후 3.50.36.png](https://raw.githubusercontent.com/rottoy/TIL/main/assets/designing-instagram-01.png)
    
1. **Db 스키마**
- 면접 초기에 DB 스키마를 정의하는 것은 여러 요소들간의 데이터 흐름을 파악하는것은 물론 데이터 파티셔닝에도 도움을 준다.
- 우리는 유저, 사진, 팔로우에 대한 데이터를 저장해야 한다. 사진 테이블은 사진과 관련된 데이터를 저장한다; 최신 사진을 가져오기 위해 **인덱스를 적용**해야 한다.(PhotoID, CreationDate)
    - 최신과 인덱스는 무슨 연관?
- RDBMS 와 NoSQL 중에 무엇을 써야 할지도 결정해야 한다.
- RDBMS(MySQL)
    - RDBMS를 쓰기엔 약간의 어려움이 있다.

![스크린샷 2021-09-25 오후 4.15.49.png](https://raw.githubusercontent.com/rottoy/TIL/main/assets/designing-instagram-02.png)

- NoSQL
    
    **Photo**
    
    key : PhotoID
    
    value : Object(PhotoLocation, UserLocation, CreationTimestamp ...)
    
    **UserPhoto**
    
    key : UserID
    
    value : PhotoIDs (유저가 소유한 사진)
    
    **UserFollow**
    
    key : UserID
    
    value : UserIDs(유저가 팔로우한 유저들)
    
    UserPhoto 와 UserFollow 테이블의 경우, Cassandra와 같은 wide-column DB를 사용하면 좋다.
    
    Cassandra 혹은 key-value 저장소는 신뢰성을 보장하기 위해 여러 복제본을 유지한다(why?).
    
    또한, 데이터 삭제가 즉시 적용되지 않는다.
    
    NoSQL의 경우 update에 관한 작업은 어렵지 않나?
    
1. DB 데이터 용량 산정
- 10년간 사용될 데이터 총량을 계산해보자.
- User
    - UserID(4) + Name(20) + Email(32) + DateOfBirth(4) + CreationDate(4) + LastLogin(4) = 68 (Bytes)
    - 10년 : 68(Bytes) * 5억명 = 32(GB)
- Photo
    - PhotoID (4 bytes) + UserID (4 bytes) + PhotoPath (256 bytes) + PhotoLatitude (4 bytes) + PhotoLongitude(4 bytes) + UserLatitude (4 bytes) + UserLongitude (4 bytes) + CreationDate (4 bytes) = 284 bytes
    - 10년 : 284(Bytes) * 2M * 10 * 365 ~= 1.88TB
- UserFollow
    - 8 bytes
    - 5억명 * 500 * 8 bytes ~=1.82TB
- Total : 32GB + 1.88TB + 1.82TB ~=3.7TB

1. Component Design

1. 신뢰성과 복제성
- 파일 유실의 가능성(고가용성)을 염두하여, storage server와 db 서버는 여러 개의 복제(replication)이 필요함.
- 복제품을 생성하는 것은 시스템의 실패 가능성을 제거한다.
    - 예를 들어, 한 서비스에 2가지 인스턴스가 있다면, 한 인스턴스가 다운되었을 때 복제 인스턴스를 사용하면 된다.
    - Failover(장애 극복 기능)은 자동적이거나 수동 조작을 필요로 한다.
    
    ![Untitled](https://raw.githubusercontent.com/rottoy/TIL/main/assets/designing-instagram-03.png)
    
1. Data Sharding([Data Sharding 이란?](https://github.com/rottoy/TIL/blob/main/courses/GROKKING%20THE%20SYSTEM%20DESIGN/data-sharding.md))
    1. 유저ID를 기반으로한 파티셔닝
        1. 유저ID를 기반으로 하기에, **한 유저의 모든 사진은 같은 샤드에 들어간다 가정.**
        2. 한 DB 샤드당 1TB(scale-in의 한계로 샤딩을 적용하는 듯)라고 하면 3.7TB를 수용하려면 최소 4개의 샤드가 필요함.(확장성과 성능을 위해 10개의 샤드라고 가정하자.)
        3. PhotoID를 고유하게 유지하기 위해선 어떻게 해야 하는가?
            1. shard number를 각 photoID 뒤에 붙여준다.
            2. auto-increment여도 shardID가 뒤에 붙기 때문에, 고유한 값이 매겨질 것이다.
        4. 문제점
            1. 많은 팔로워를 가진 user(조회 문제)
            2. 많은 사진을 가진 user(업로드 용량 문제)
            3. 한 유저에 대한 사진을 한 샤드에 보관하지 못하는 문제가 발생한다면? 그로 인해 분산하면 야기되는 또 다른 문제는?
            4. 고가용성의 문제(유저의 데이터 손실 등...)
    2. PhotoID를 기반으로 한 파티셔닝(**정답**)
        1. 위의 문제점들을 모두 해결할 수 있는 방법이다.(정말?)
        2. id의 uniqueness는 dedicate DB(key generating DB)로 키를 생성하는 DB를 통해 생성한다.
            1. **단일 실패 지점을 예방하기 위해, key generating DB를 2개 사용한다.**
            2. 각 2개의 DB는 서로 async 하여도 상관없다(rule에 의한 auto-increment이므로 독립적임)
            3. 각 ID 생성 DB를 분리함으로써 시스템을 확장 시킬 수도 있다.
                
                ```yaml
                KeyGeneratingServer1:
                auto-increment-increment = 2
                auto-increment-offset = 1
                
                KeyGeneratingServer2:
                auto-increment-increment = 2
                auto-increment-offset = 2
                ```
                
        3. 어떻게 시스템의 확장성을 열어 둘 수 있을까?
            1. 시스템 초기에, **논리적 파티션으로 데이터를 분리**하여 보관하면 된다.(물리 db는 같을 지라도)
            2. 그렇게 설계한다면, 아무 서버에서나 논리적 파티션을 이주하여 사용할 수 있다.
            3. 옮길 때마다, 설정 파일을 업데이트 하기만 하면 된다.
            
2. 랭킹과 뉴스 피드 생성
    - 모든 사용자에게 뉴스 피드를 제공하려면, 가장 최신의, 인기있는, 상관있는 팔로우한 사람들의 뉴스 피드를 제공해야 한다.
    - 100개의 뉴스피드를 가져온다고 가정하자.
        1. 팔로워를 조인하여 100개의 최근 게시글에 관한 메타 데이터를 가져올 것이다.
        2. 랭킹 알고리즘에 의하여, 순서를 결정하고 유저에게 데이터를 반환할 것이다.
        3. 문제는 **높은 지연**
            1. 여러 개의 테이블에 쿼리
            2. 결과 이전 sorting, merging, ranking 수행
            3. 이를 방지하기 위해, **미리** 뉴스피드를 생성하는 방법이 있다.
    - 뉴스피드 미리 생성하기
        - 시간 간격을 정해 UserNewsFeed 테이블에 미리 뉴스피드 결과값을 저장한다.
            - 유저가 뉴스피드를 요청할 때마다, UserNewsFeed 테이블에 쿼리한다.
            - 서버가 뉴스피드를 생성할 때마다, UserNewsFeed 테이블의 마지막 생성 뉴스피드를 조회한다.
    - 다른 관점에서의 뉴스피드
        - Push
            - 클라이언트는 주기적으로 혹은 필요할때마다 뉴스피드를 요청할 수 있다.
                1. 클라이언트가 pull 하기 전까지, 최신 뉴스피드가 보이지 않을 수 있다.
                2. 대부분의 경우, pull request는 새 데이터가 없다면 빈 데이터일 것이다.
        - Pull
            - 서버는 가용 가능한 상태일 때 유저에게 데이터를 Push 할 수 있다. 따라서 유저는 업데이트를 받기  위해 Long Poll request를 기다려야 한다.
            - 서버는 많은 팔로워를 가진 유저의 뉴스피드를 빈번하게 갱신해줘야 한다.
        - Hybrid
            - 많은 팔로워(수백)를 가진 유저에 대해서만 데이터를 push 하기
            - 모든 유저에게 적은 빈도로 데이터를 push하고, pull request를 많이 받기
            
        
3. 샤드된 데이터로 뉴스 피드 생성하기
    1. 뉴스피드의 가장 중요한 목적은, 가장 최신의 데이터를 반환하는 것이다. 그렇다면 샤드된 데이터로 뉴스 피드를 생성하는데 문제는 없을까?
        1. 사진 id에는 2가지의 의미가 있게끔 만들어야 한다.
            1. epoch time(발생 시간)
            2. unique(고유함)
            
            발생 시간에 auto-increment key 값을 더하여 사진id를 생성한다면 샤드된 데이터에서도 뉴스 피드를 찾는데 문제 없을것이다. 
            
    2. 뉴스 id의 사이즈는 얼마가 되어야 할까? 이전에 2가지 사항이 만족되어야 한다고 했을 때,
        1. epoch time
            1. 86400(sec) * 365 * 50 ~= 16억초
            2. 31bit로 표현가능.
        2. unique
            1. 초당 23개의 사진이 저장된다고 예상했음.
            2. 넉넉하게 추가적으로 9bit를 할당한다고 하자.(동시성으로 최대 512개의 사진을 저장가능)
        
        31 + 9 = 40bit(=5 bytes)가 필요하다.
        
        또한, auto-increment를 매 초마다 초기화 해도 된다.
        
4. 캐시와 로드 밸런싱(캐싱)
    1. 세계적으로 분산된 데이터를 유저에게 빠르게 전달하기 위해 **CDN**이 필요하다.
    2. hot DB row를 관리하는 서버 cache 구성도 필요하다.
        1. 서버의 memcache를 이용하여, DB에 쿼리하기 전에 캐시에 row가 있는지 확인한다.
        2. 시스템 용도에 맞게 LRU 알고리즘이 적합할 것이다.
    3. 효율적인 캐시 빌드법