# GROKKING THE SYSTEM DESIGN

[시스템 디자인 용어 정리](https://www.notion.so/d5d558bb5a524d0f9b62c6a9c8b79b38)

[System Design Problems](https://www.notion.so/System-Design-Problems-ccdc0ce6425f49ba84fc7bb2e6194159)

[Glossary of System Design Basics](https://www.notion.so/Glossary-of-System-Design-Basics-8b1f78412d4d42d88bc79c20e0689669)

엔지니어들은 SDI(System Design Interview) 에 대해 다음 3가지 이유로 인해, 문제를 겪는다.

- 면접자들은 기존에 답이 정해지지 않은, 열린 시스템 디자인 문제에 대하여 SDI가 갖춰져 있지 않음.
- 면접자들은 복잡한 대용량 시스템에 대해 경험이 부족함.
- SDI에 충분한 시간을 쏟아붇지 않았음

코딩 인터뷰와 마찬가지로, SDI에 대해 근면성실한 준비가 없다면 구글, 넷플릭스, 아마존과 같은 대기업에서 수행능력이 형편 없을 것이다. 평균 이상 수행하지 못하는 면접자들은 제한된 기회를 얻을 것이다.

디자인 문제를 해결하기 위해 다음 차례를 구성했다.

**Step 1. 요구 사항 명료화**

- 문제에 대한 정확한 범위가 필요.
- 대부분 정답이 정해지지 않은 문제들이기 때문에, 모호성을 명료화 하는것이 인터뷰의 핵심이다. 명확하게 시스템의 목적을 정의하는 면접자들은 인터뷰에서 성공적인 결과를 거둘수 있다.
- 트위터 사례를 예로 들어보자.
    - 이용 고객이 다른 사람을 트윗하고 팔로우를 할 수 있는가?
    - 유저의 타임라인을 생성하고 조회할수 있는가?
    - 트윗은 사진과 비디오를 포함할 수 있는가?
    - 백엔드, 프론트엔드 모두 개발할 것인가?
    - 유저가 트윗을 검색할 수 있는가?
    - 핫 토픽을 보여줄 수 있는가?
    - 푸시 알림이나 새로운 트윗 알람이 있는가?
- 이러한 질문들은 디자인이 어떻게 될 지 결정하게끔 도와준다.

**Step 2.  어림잡은 추산**

- 시스템의 규모를 어림잡아 측정하는 것도 필요
- 이것은 후에 스케일링, 파티셔닝, 로드밸런싱, 캐싱에 초점을 둘 때 도움이 됨
    - 얼마나 많은 스토리지를 필요로 하는가?(사진과 비디오에 따라 요구사항이 달라질 것이다.)
    - network bandwidth(대역폭) 기대 사용량은 얼마인가? 이것은 트래픽 관리나 서버의 로드 밸런싱 시에 중요하게 작용할 것이다.
    

**Step 3. 시스템 인터페이스 정의**

시스템에서 예상되는 API를 정의하라. 이것은 우리가 요구사항을 제대로 파악했는지 확인 시켜줄 것이다.  예상되는 트위터 API는 다음과 같다.

```bash
postTweet(user_id, tweet_data, tweet_location, user_location, timestamp, ...)
generateTimeline(user_id, current_time, user_location, ...)
markTweetFavorite(user_id, tweet_id, timestamp)
```

**Step 4. 데이터 모델 정의**

데이터를 면접 초창기에 정의하는 것은 시스템 구성요소 사이에서 데이터가 어떻게 전달될 것인지를 명료하게 해준다. 후에는, 데이터 파티셔닝 및 관리에 도움을 줄 것이다. 면접자는 다양한 시스템 엔티티를 정의해야 하므로, 어떻게 상호작용할 것인지, 다른 양상의 데이터(전송용, 암호화, 등등...)을 어떻게 처리할 것인지 파악해야 한다.

- User
    - UserID, Name, Email, DoB, CreationDate, LastLogin etc...
- Tweet
    - TweetID, Content, TweetLocation, NumberOfLikes, TimeStamp etc...
- UserFollow
    - UserID1, UserID2
- FavoriteTweets
    - UserID, TweetID, Timestamp
    

어느 DB를 쓸 지도 정해야 한다. 카산드라 같은 NoSQL을 사용할 지 , MySQL 솔루션을 사용해야 할지 정하고, 블록 스토리지는 무엇을 사용해야 할지 정해야 한다.

**Step 5. 고레벨 디자인**

5-6개 정도의 코어 컴포넌트가 있는 블록 다이어그램을 그려보아라.

우리는 실제 문제를 해결할 만큼의 충분한 컴포넌트들을 정의할 필요가 있다.

예를 들어, 트위터는 로드밸런서 사용을 통한 IO 분산을 위해 여러개의 어플리케이션 서버가 필요할 것이다. 만약 조회 트래픽이 더 많다고 예상되면, 우리는 이 시나리오를 만족시키기 위해 서버를 분리해야 할 것이다.

백엔드 단에서는, 많은 양의 트윗과 많은 수의 조회를 감당할 수 있는 충분한 DB가 필요할 것이다.

또한 사진과 비디오를 위한 분산 파일 스토리지 시스템도 필요할 것이다.

그림으로 나타내면 다음과 같다.

![Untitled](GROKKING%20THE%20SYSTEM%20DESIGN%20d8a8c8de4c1443ac9c35a4fc8f5f4c28/Untitled.png)

 

**Step 6. 세부 디자인**

여러 병목지점들을 예상해야 한다.

- 시스템의 실패 지점 예측
- 데이터 유실을 대비한 복제
- 시스템 셧다운을 방지하기 위한 여러개의 replica 서비스들
- 성능에 대한 서비스 모니터링 및 알림

**시스템 디자인 면접을 준비할때 주의사항**

1. Do not go into details prematurely(깊게 파고들지 않기)
2. Don't have a set architecture in mind (avoid silver bullets) - 은탄환 신드롬.정해진 답은 없다
3. KISS (Keep In Short and Simple, 오컴의 면도날, 논리적 비약 금지)
4. Don't make points without justification (tell why, form your thoughts) - 근거 없이 주장하지 않기
5. Be aware of current technologies (existing off-the-shelf solutions)