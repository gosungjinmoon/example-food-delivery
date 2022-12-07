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

```
gitpod /workspace/mall (main) $ http :8081/orders item="치킨" qty=10 price=200 state="주문접수-결재완료"
HTTP/1.1 201 
Connection: keep-alive
Content-Type: application/json
Date: Tue, 06 Dec 2022 06:19:20 GMT
Keep-Alive: timeout=60
Location: http://localhost:8081/orders/1
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/1"
        },
        "self": {
            "href": "http://localhost:8081/orders/1"
        }
    },
    "item": "치킨",
    "price": 200,
    "qty": 10,
    "state": "주문접수-결재완료"
}


gitpod /workspace/mall (main) $ http :8084/payments
HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/hal+json
Date: Tue, 06 Dec 2022 06:19:36 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_embedded": {
        "payments": [
            {
                "_links": {
                    "payment": {
                        "href": "http://localhost:8084/payments/1"
                    },
                    "self": {
                        "href": "http://localhost:8084/payments/1"
                    }
                },
                "action": "progress",
                "amount": 2000,
                "orderId": 1
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8084/profile/payments"
        },
        "self": {
            "href": "http://localhost:8084/payments"
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 1,
        "totalPages": 1
    }
}


gitpod /workspace/mall (main) $ http :8082/orderManagements
HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/hal+json
Date: Tue, 06 Dec 2022 06:19:51 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_embedded": {
        "orderManagements": [
            {
                "_links": {
                    "orderManagement": {
                        "href": "http://localhost:8082/orderManagements/1"
                    },
                    "self": {
                        "href": "http://localhost:8082/orderManagements/1"
                    }
                },
                "address": "test주소",
                "foodType": "한식",
                "orderId": 1,
                "state": null
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8082/profile/orderManagements"
        },
        "self": {
            "href": "http://localhost:8082/orderManagements"
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 1,
        "totalPages": 1
    }
}


gitpod /workspace/mall (main) $ 
```

## 2. CQRS 
읽기 모델을 분리한다.
- app -> OrderStateViewHandler.java에서 이벤트에 따라 Real Model 저장, 업데이트, 삭제를 정의한다. 
```
package mall.infra;

import mall.domain.*;
import mall.config.kafka.KafkaProcessor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

@Service
public class OrderStateViewHandler {

    @Autowired
    private OrderStateRepository orderStateRepository;

    // 생성될 경우 해당 Real Model에 값 저장
    @StreamListener(KafkaProcessor.INPUT)
    public void whenOrderPlaced_then_CREATE_1 (@Payload OrderPlaced orderPlaced) {
        try {

            if (!orderPlaced.validate()) return;

            // view 객체 생성
            OrderState orderState = new OrderState();
            // view 객체에 이벤트의 Value 를 set 함
            orderState.setId(orderPlaced.getId());
            orderState.setItem(orderPlaced.getItem());
            orderState.setState("생성");
            // view 레파지 토리에 save
            orderStateRepository.save(orderState);

        }catch (Exception e){
            e.printStackTrace();
        }
    }
}
```

- 아래의 결과처럼 orderStates에 저장된다.
```
gitpod /workspace/mall (main) $ http :8081/orderStates
HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/hal+json
Date: Tue, 06 Dec 2022 08:22:59 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_embedded": {
        "orderStates": []
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8081/profile/orderStates"
        },
        "self": {
            "href": "http://localhost:8081/orderStates"
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 0,
        "totalPages": 0
    }
}


gitpod /workspace/mall (main) $ http :8081/orders item="치킨" qty=10 price=200 state="주문접수-결재완료"
HTTP/1.1 201 
Connection: keep-alive
Content-Type: application/json
Date: Tue, 06 Dec 2022 08:23:10 GMT
Keep-Alive: timeout=60
Location: http://localhost:8081/orders/1
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/1"
        },
        "self": {
            "href": "http://localhost:8081/orders/1"
        }
    },
    "item": "치킨",
    "price": 200,
    "qty": 10,
    "state": "주문접수-결재완료"
}


gitpod /workspace/mall (main) $ http :8081/orderStates
HTTP/1.1 200 
Connection: keep-alive
Content-Type: application/hal+json
Date: Tue, 06 Dec 2022 08:23:23 GMT
Keep-Alive: timeout=60
Transfer-Encoding: chunked
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers

{
    "_embedded": {
        "orderStates": [
            {
                "_links": {
                    "orderState": {
                        "href": "http://localhost:8081/orderStates/1"
                    },
                    "self": {
                        "href": "http://localhost:8081/orderStates/1"
                    }
                },
                "item": "치킨",
                "state": "생성"
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8081/profile/orderStates"
        },
        "self": {
            "href": "http://localhost:8081/orderStates"
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 1,
        "totalPages": 1
    }
}


gitpod /workspace/mall (main) $ 
```

- 다만 위와 같은 모델로 구현할 경우 다른 도메인에 있는 이벤트에 따른 뷰를 제공하지 못하므로 단독 컨텍스트에 리얼 모델을 구현하여 보완이 필요함.

## 3. Compensation / Correlation
- 작성예정

## 4. Request / Response

![image](https://user-images.githubusercontent.com/53729857/205790895-04938551-3ad8-471c-b8c6-676527a106a7.png)

Reqeust / Response : Pay쪽 서버가 올라오지 않을경우 에러발생.


![image](https://user-images.githubusercontent.com/53729857/205791167-d9b4359b-e816-482a-90bc-3ba348ab5a67.png)

Reqeust / Response : Pay쪽 서버가 정상일 경우만 정상 작동

Pub / Sub : Store쪽 서버가 올라오지 않을 경우에도 올라온 서버안에서 정상작동


## 5. Circuit Breaker
- pay -> Payment.java에 추가
```
    @PrePersist
    public void onPrePersist(){
        if(action.equals("canceled")){
            PaymentCanceled paymentCanceled = new PaymentCanceled();
            BeanUtils.copyProperties(this, paymentCanceled);
            paymentCanceled.publish();
        }else if(action.equals("progress")){
            PaymentApproved paymentApproved = new PaymentApproved();
            BeanUtils.copyProperties(this, paymentApproved);

            // 주문 정보가 커밋된 후에 이벤트 발생시켜야 한다.
            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
                @Override
                public void beforeCommit(boolean readOnly) {
                    paymentApproved.publish();
                }
            });
            
            // 강제 지연
            try {
                Thread.currentThread().sleep((long) (1000 + Math.random() * 220));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }else{
            System.out.println("알 수 없는 action");
        }

    }
```
- app -> pom.xml에 추가
```
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
		</dependency>
```
- app -> application.yml에 추가
```
feign:
  hystrix:
    enabled: true

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 10
```
- app -> PaymentServiceFallBack.java 
```
package mall.external;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;



@Service
public class PaymentServiceFallBack implements PaymentService {
    private static Logger logger = LoggerFactory.getLogger(PaymentServiceFallBack.class);

    @Override
    public void pay(Payment payment) {
        logger.error("Circuit breaker has been opened. Fallback returned instead.");
    }
}
```
- 실행 시 아래의 에러 로그를 app쪽 로그에서 확인가능
![image](https://user-images.githubusercontent.com/53729857/205817063-abf4008f-a96b-4f6f-8c28-b493878baf36.png)

- 과부하 테스트 명령어
![image](https://user-images.githubusercontent.com/53729857/205828449-8838f7cb-3342-48d6-b1e2-de4d36ccc191.png)

- 초기상태
![image](https://user-images.githubusercontent.com/53729857/205828846-d13262ad-8136-46d6-be86-53db3531fde9.png)

- 요청이 점점 밀리는 것을 확인 가능
![image](https://user-images.githubusercontent.com/53729857/205829022-646faa3b-1707-4a8e-9987-85f6d08a9cd3.png)

- 종료
![image](https://user-images.githubusercontent.com/53729857/205829348-11600e6b-6307-4586-9939-695e9e5f56d3.png)

- WAS 로그 - 중간중간 FallBack 로그가 있음
![image](https://user-images.githubusercontent.com/53729857/205829830-ea1edeac-a025-41fe-9d50-92d15ed502d7.png)

## 6. Gateway / Ingress
gateway의 라우터 설정으로 :8081/orders 요청과 :8088/orders 요청이 같은 서비스를 제공한다.
```
spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: app
          uri: http://localhost:8081
          predicates:
            - Path=/orders/**, /menus/**, /orderStates/**
```
![image](https://user-images.githubusercontent.com/53729857/205836882-b940a8ac-e567-43c0-b569-4b6adc0aa981.png)
![image](https://user-images.githubusercontent.com/53729857/205836894-5858d3c0-08d5-4bd7-b8b0-7dba84c23b8a.png)



# 기타
## Before Running Services
### Make sure there is a Kafka server running
```
cd kafka
docker-compose up
```
- Check the Kafka messages:
```
cd kafka
docker-compose exec -it kafka /bin/bash
cd /bin
./kafka-console-consumer --bootstrap-server localhost:9092 --topic
```

## Run the backend micro-services
See the README.md files inside the each microservices directory:

- app
- store
- customer
- pay
- rider


## Run API Gateway (Spring Gateway)
```
cd gateway
mvn spring-boot:run
```

## Test by API
- app
```
 http :8088/orders id="id" item="item" qty="qty" price="price" state="state" 
```
- store
```
 http :8088/orderManagements id="id" orderId="orderId" address="address" foodType="foodType" state="state" 
```
- customer
```
```
- pay
```
 http :8088/payments id="id" orderId="orderId" amount="amount" action="action" 
```
- rider
```
 http :8088/deliveries id="id" orderId="orderId" state="state" address="address" 
```


## Run the frontend
```
cd frontend
npm i
npm run serve
```

## Test by UI
Open a browser to localhost:8088

## Required Utilities

- httpie (alternative for curl / POSTMAN) and network utils
```
sudo apt-get update
sudo apt-get install net-tools
sudo apt install iputils-ping
pip install httpie
```

- kubernetes utilities (kubectl)
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

- aws cli (aws)
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

- eksctl 
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```
