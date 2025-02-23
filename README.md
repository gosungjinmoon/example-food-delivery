# 예제 - 음식배달

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.

# 서비스 시나리오

기능적 요구사항
1. 고객이 메뉴를 선택하여 주문한다.
1. (추가) 회원이 아니면 신규 가입을 하고 가입내역을 이메일로 전달 받는다.
1. (추가) 결제 수단을 등록시 카드 유효성을 확인한다. 
1. 주문이 되면 주문 내역이 입점상점주인에게 주문정보가 전달된다
1. 상점주는 주문을 수락하거나 거절할 수 있다
1. 상점주는 요리시작때와 완료 시점에 시스템에 상태를 입력한다
1. 고객은 아직 요리가 시작되지 않은 주문은 취소할 수 있다
1. 요리가 완료되면 고객의 지역 인근의 라이더들에 의해 배송건 조회가 가능하다
1. 라이더가 해당 요리를 Pick한 후, 앱을 통해 통보한다.
1. 고객이 주문상태를 중간중간 조회한다
1. 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다
1. 고객이 요리를 배달 받으면 배송확인 버튼을 탭하여, 모든 거래가 완료된다


비기능적 요구사항
1. 장애격리
    1. 상점관리 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 고객이 자주 상점관리에서 확인할 수 있는 배달상태를 주문시스템(프론트엔드)에서 확인할 수 있어야 한다  CQRS
    1. 배달상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다  Event driven


## Model
![image](https://user-images.githubusercontent.com/55925545/206178264-da04e6e1-c4f6-4a04-8866-5d7eb8c51bbd.png)


요구사항을 커버하는지 검증

![image](https://user-images.githubusercontent.com/55925545/206178494-38a9bd52-c49c-45ff-aa0a-8791c98adca0.png)  
1. 고객이 메뉴를 선택하여 주문한다. (ok)
2. (추가) 회원이 아니면 신규 가입을 하고 가입내역을 이메일로 전달 받는다. (ok)
3. (추가) 결제 수단을 등록시 카드 유효성을 확인한다. (ok)
4. 주문이 되면 주문 내역이 입점상점주인에게 주문정보가 전달된다. (ok)
5. 상점주는 주문을 수락하거나 거절할 수 있다. (ok)
6. 상점주는 요리시작때와 완료 시점에 시스템에 상태를 입력한다. (ok)
    
![image](https://user-images.githubusercontent.com/55925545/206179237-3a7ccae9-5ffd-4958-a028-0b39f01f7a2e.png)

7. 고객은 아직 요리가 시작되지 않은 주문은 취소할 수 있다. (ok)
8. 요리가 완료되면 고객의 지역 인근의 라이더들에 의해 배송건 조회가 가능하다. (ok)
9. 라이더가 해당 요리를 Pick한 후, 앱을 통해 통보한다. (ok)
10. 고객이 주문상태를 중간중간 조회한다. (ok)
11. 주문상태가 바뀔 때 마다 카톡으로 알림을 보낸다. (ok)
12. 고객이 요리를 배달 받으면 배송확인 버튼을 탭하여, 모든 거래가 완료된다. (ok)

# 체크포인트
## 1. Saga(Pub/Sub)
orders로 post 요청을 보내면 OrderPlaced에서 pay에 있는 pay커맨드로 요청을 전달한다.(req/res : 동기)
그 후에 pay에서 PaymentApproved이벤트를 거쳐 store에 있는 receipt정책으로 이벤트를 전달한다.(Pub/Sub : 비동기)
아래는 orders post요청으로 3개의 테이블에 데이터가 들어간 것을 확인한 증적이다.

### 요청
![image](https://user-images.githubusercontent.com/55925545/206179718-777ffab6-72db-4acb-ad27-6a2ab68d943f.png)

### 결과
![image](https://user-images.githubusercontent.com/55925545/206179988-becbe410-b982-462a-be9b-cd4fd17efa14.png)

## 2. CQRS 
![image](https://user-images.githubusercontent.com/55925545/206180485-7a59af7c-5d2c-4435-94bf-ce676d3f29fe.png)

![image](https://user-images.githubusercontent.com/55925545/206181763-ba18bc1d-e5a9-4ded-b873-40b7e78b90ac.png)

## 3. Request / Response

![image](https://user-images.githubusercontent.com/55925545/206180164-76662dda-dbe2-4441-8f10-df382a595659.png)

## 4. Gateway
![image](https://user-images.githubusercontent.com/55925545/206180772-ee5ffd99-679f-48be-9c77-35278247b1c7.png)

