# Book Market

- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [Book Market](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏](#폴리글랏)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
    - [CQRS](#CQRS)
    - [gateway](#gateway)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
    - [Liveness](#Liveness)
    - [Config Map](#Config-Map)

# 서비스 시나리오

기능적 요구사항
1. 고객은 책을 주문한다 ( Core Domain - Order )
1. 고객이 주문을 할때는 반드시 결제가 되어야 한다.(Req/Rep) ( Circuit Breaker(결제 지연))
1. 결제가 완료되면 배송을 시작한다. ( Pub / Sub Event Dirven )
1. 결제완료되면 주문 상태를 변경한다 ( Pub / Sub Event Dirven )
1. 배송이 시작되면 주문 상태를 변경한다 ( Pub / Sub Event Dirven )
1. 고객은 주문을 취소한다.
1. 주문이 취소되면 결제를 취소한다. ( Pub / Sub Event Dirven )
1. 결제가 취소되면 배송을 취소한다. ( Pub / Sub Event Dirven )

1. 배송이 시작되면 배송 시작 메시지를 전송한다 ( Pub / Sub Event Dirven )

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다  Sync 호출 
1. 장애격리
    1. 배송 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 주문시스템이 과중되면 사용자를 잠시동안 받지 않고 주문를 잠시 후에 하도록 유도한다  Circuit Breaker, fallback
    1. 메시지 전송이 수행되지 않더라도 시스템 운영에 이상이 없어야 한다 Async (event-driven), Eventual Consistency
1. 성능
    1. 고객이 주문 상태를 시스템에서 확인할 수 있어야 한다  CQRS
    1. 메시지 전송 현황을 시스템에서 확인할 수 있어야 한다  CQRS


# 체크포인트

- 분석 설계

# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/8PeWNorN2DZmr9eaKXl5ZKjKrJJ3/share/e97a1124efe99dc1edb15f8453d6a125/-MLBdLSL1aMfQCoHSb7u


### 이벤트 도출
![image](https://user-images.githubusercontent.com/70673849/98312340-de8ab880-2014-11eb-98a9-b3c25fba835a.png)

    - 도메인 서열 분리 
        - Core Domain:  Order : bookmarket 핵심 서비스이며, 연간 Up-time SLA 수준을 99.999% 목표, 배포 주기는 Order 의 경우 1주일 1회 미만
        - Supporting Domain:   Delivery : 경쟁력을 내기 위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain:   Payment : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음
        - General Domain:   Kakao : 메시지 서비스로 3rd Party 공인 서비스 사용

## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/20619166/98073892-acf6dd80-1eac-11eb-99ec-0a7521d96aca.PNG)

    - 이벤트 흐름에서 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출 관계에서 Pub/Sub 과 Req/Resp 를 구분함
    - 바운디드 컨텍스트에 서브 도메인을 1 대 1 모델링하고 팀원별 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 Bounded Context 별로 대변되는 마이크로 서비스들을 Spring Boot 로 구현하였다. 
구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다. (포트 넘버는 8081 ~ 8084 이다)

```
cd Order
mvn spring-boot:run

cd Payment
mvn spring-boot:run 

cd Delivery
mvn spring-boot:run  

cd customerview
mvn spring-boot:run 

cd Kakao
mvn spring-boot:run  

cd Kakaomng
mvn spring-boot:run 
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 Order 마이크로 서비스)

```
package bookmarket;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Kakao_table")
public class Kakao {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long orderId;
    private Long customerId;
    private String status;
    private String message;

    @PrePersist
    public void onPrePersist(){
        SentMessage sentMessage = new SentMessage();
        BeanUtils.copyProperties(this, sentMessage);
        sentMessage.publishAfterCommit();
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getOrderId() {
        return orderId;
    }

    public void setOrderId(Long orderId) {
        this.orderId = orderId;
    }
    public Long getCustomerId() {
        return customerId;
    }

    public void setCustomerId(Long customerId) {
        this.customerId = customerId;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

}


```
- Entity / Repository Pattern을 적용하여 JPA를 통하여 다양한 데이터소스 유형 (HSQLDB, H2) 에 대한 데이터 접근 어댑터 자동 생성하여 
Spring Data REST의 RestRepository 를 적용
```
package bookmarket;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface KakaoRepository extends PagingAndSortingRepository<Kakao, Long>{

}
```
- 적용 후 REST API 의 테스트
```
# Order 서비스의 주문 및 배송 이후, 메시지 전송 결과 확인

```
![image](https://user-images.githubusercontent.com/70673849/98313129-8359c580-2016-11eb-99b0-726e0a887a5d.png)

```

# 메시지 전송 확인

```
![image](https://user-images.githubusercontent.com/70673849/98313555-75587480-2017-11eb-9f18-9fc92a74d05a.png)


# 메시지 전송 누적 결과 확인 (CQRS)

root@labs--1345560735:~/scenario# sh 5*
HTTP/1.1 200 OK
Content-Type: application/hal+json;charset=UTF-8
Date: Fri, 06 Nov 2020 01:06:18 GMT
transfer-encoding: chunked

{
    "_embedded": {
        "kakaomsgs": [
            {
                "_links": {
                    "kakaomsg": {
                        "href": "http://kakaomsg:8080/kakaomsgs/1"
                    }, 
                    "self": {
                        "href": "http://kakaomsg:8080/kakaomsgs/1"
                    }
                }, 
                "customerId": 33, 
                "message": "1 33 Shipped...", 
                "orderId": 1, 
                "status": "Shipped"
            }, 
            {
                "_links": {
                    "kakaomsg": {
                        "href": "http://kakaomsg:8080/kakaomsgs/2"
                    }, 
                    "self": {
                        "href": "http://kakaomsg:8080/kakaomsgs/2"
                    }
                }, 
                "customerId": 33, 
                "message": "2 33 Shipped...", 
                "orderId": 2, 
                "status": "Shipped"
            }, 
            {
                "_links": {
                    "kakaomsg": {
                        "href": "http://kakaomsg:8080/kakaomsgs/3"
                    }, 
                    "self": {
                        "href": "http://kakaomsg:8080/kakaomsgs/3"
                    }
                }, 
                "customerId": 33, 
                "message": "3 33 Shipped...", 
                "orderId": 3, 
                "status": "Shipped"
            }, 
            {
                "_links": {
                    "kakaomsg": {
                        "href": "http://kakaomsg:8080/kakaomsgs/4"
                    }, 
                    "self": {
                        "href": "http://kakaomsg:8080/kakaomsgs/4"
                    }
                }, 
                "customerId": 33, 
                "message": "4 33 Shipped...", 
                "orderId": 4, 
                "status": "Shipped"
            }, 
            {
                "_links": {
                    "kakaomsg": {
                        "href": "http://kakaomsg:8080/kakaomsgs/5"
                    }, 
                    "self": {
                        "href": "http://kakaomsg:8080/kakaomsgs/5"
                    }
                }, 
                "customerId": 33, 
                "message": "5 33 Shipped...", 
                "orderId": 5, 
                "status": "Shipped"
            }
        ]
    }, 
    "_links": {
        "profile": {
            "href": "http://kakaomsg:8080/profile/kakaomsgs"
        }, 
        "self": {
            "href": "http://kakaomsg:8080/kakaomsgs"
        }
    }
}
```


## 폴리글랏 퍼시스턴스

Kakao 서비스에는 HSQLDB를 사용하고, Kakaomng 서비스는 H2 사용으로 퍼시스턴스 폴리글랏 구현 확인.

![image](https://user-images.githubusercontent.com/70673849/98313873-252de200-2018-11eb-947c-b077d9422643.png)


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


배송 이후 메시지 서비스로 이를 알려주는 행위는 비동기식으로 처리하여 메시지로 인한 전체 기능이 블로킹 되지 않도록 처리한다.
 
- 이를 위하여 배송 이력에 기록을 남긴 후에 곧바로 도메인 이벤트를 카프카로 송출한다(Publish)
- 메시지 서비스에서는 PolicyHandler를 통해 수신하고 (Subscribe) 메시지 처리 및 이력을 기록
 
```
@Service
public class PolicyHandler{
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @Autowired
    KakaoRepository kakaoRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverShipped_SendMessage(@Payload Shipped shipped){

        if(shipped.isMe()){
            System.out.println("##### listener SendMessage : " + shipped.toJson());

            if(shipped.isMe()){
                System.out.println("##### listener Ship : " + shipped.toJson());
                Kakao kakao = new Kakao();
                kakao.setOrderId(shipped.getOrderId());
                kakao.setCustomerId(shipped.getCustomerId());
                kakao.setStatus("Shipped");
                kakao.setMessage(shipped.getOrderId() + " " + shipped.getCustomerId() + " Shipped...");

                kakaoRepository.save(kakao);
            }
        }
    }

메시지 서비스는 주요 시스템과 완전히 분리되어, 시스템에 영향이 없음.


## gateway
gateway 프로젝트 내 application.yml 에 두 개 서비스 추가 구성 (kakao, kakaomsg)

![image](https://user-images.githubusercontent.com/70673849/98314537-8904da80-2019-11eb-9762-7059bad48b8c.png)


# 운영

## CI/CD 설정

개별 과제 구현을 위해 팀 과제 결과물을 포함, Azure Pipelines 으로 CI/CD 를 구성하였으며, 구성은 아래와 같다. 
Github 소스 변경이 감지되면, CI 후 trigger 에 의해 CD까지 자동으로 이루어진다.

- CI 
![image](https://user-images.githubusercontent.com/70673849/98314676-d5501a80-2019-11eb-86a2-8ee5da59fa38.png)

- CD
![image](https://user-images.githubusercontent.com/70673849/98314767-0a5c6d00-201a-11eb-9dc2-32b495da4c91.png)


## Circuit Breaker 점검

시나리오는 주문(Order)시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 과도한 주문 요청 시 "circuitBreaker.requestVolumeThreshold"의 옵션을 통한 장애격리 구현.

```
## Hystrix 설정
요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정

# application.yml
feign:
  hystrix:
    enabled: true
    
hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610
```
```
호출 서비스(주문:order) 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다 갔다 하게
# Order.java (Entity)

    @PrePersist
    public void onPrePersist(){  // 주문 저장 전 시간 끌기
  
        try {
            Thread.currentThread().sleep((long) (400 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
```

## 부하 발생을 통한 Circuit Breaker 점검
```
root@siege-5c7c46b788-z8jxc:/# siege -c100 -t120S -v --content-type "application/json" 'http://Order:8080/orders POST {"bookId": "10", "qty": "1", "customerId": "1002"}'
** SIEGE 4.0.4
** Preparing 100 concurrent users for battle.
The server is now under siege...
HTTP/1.1 201     5.35 secs:     226 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     5.36 secs:     226 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     5.35 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     5.37 secs:     226 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     5.34 secs:     226 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     5.44 secs:     226 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     5.47 secs:     226 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     5.48 secs:     226 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     5.49 secs:     226 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     5.50 secs:     226 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     8.64 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     8.68 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     8.78 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     8.78 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     3.31 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     8.78 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     8.80 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     8.81 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     8.80 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     8.81 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    11.84 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    11.84 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    11.91 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    11.93 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    11.95 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    11.99 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    12.01 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     3.23 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    12.03 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    12.02 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    15.00 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    15.03 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201     3.25 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    15.19 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    15.21 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    15.21 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    15.23 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    15.23 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    15.28 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    15.25 secs:     228 bytes ==> POST http://Order:8080/orders

* 과도한 요청으로 CB 작동 -> 요청 차단

HTTP/1.1 500     2.26 secs:     248 bytes ==> POST http://Order:8080/orders
HTTP/1.1 500     2.29 secs:     248 bytes ==> POST http://Order:8080/orders
HTTP/1.1 500     2.28 secs:     248 bytes ==> POST http://Order:8080/orders
HTTP/1.1 500     2.47 secs:     248 bytes ==> POST http://Order:8080/orders
HTTP/1.1 500     2.22 secs:     248 bytes ==> POST http://Order:8080/orders
HTTP/1.1 500     2.48 secs:     248 bytes ==> POST http://Order:8080/orders
HTTP/1.1 500     2.29 secs:     248 bytes ==> POST http://Order:8080/orders
HTTP/1.1 500     2.30 secs:     248 bytes ==> POST http://Order:8080/orders
HTTP/1.1 500     2.29 secs:     248 bytes ==> POST http://Order:8080/orders
HTTP/1.1 500     2.27 secs:     248 bytes ==> POST http://Order:8080/orders

* 요청을 어느 정도 차단 후, 기존에 밀린 일들이 처리되었고, 회로를 닫아 요청 처리

HTTP/1.1 201    18.05 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    18.05 secs:     228 bytes ==> POST http://Order:8080/orders

* 다시 요청이 쌓이기 시작하여 건당 처리시간 부하 => 회로 열기 => 요청 실패처리

HTTP/1.1 500     0.66 secs:     248 bytes ==> POST http://Order:8080/orders
HTTP/1.1 500     0.75 secs:     248 bytes ==> POST http://Order:8080/orders
HTTP/1.1 500     0.77 secs:     248 bytes ==> POST http://Order:8080/orders
HTTP/1.1 500     0.76 secs:     248 bytes ==> POST http://Order:8080/orders

* 요청을 어느 정도 차단 후, 기존에 밀린 일들이 처리되었고, 회로를 닫아 요청 처리

HTTP/1.1 201    18.25 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    18.31 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    18.32 secs:     228 bytes ==> POST http://Order:8080/orders

HTTP/1.1 500     0.82 secs:     248 bytes ==> POST http://Order:8080/orders
HTTP/1.1 500     0.83 secs:     248 bytes ==> POST http://Order:8080/orders
HTTP/1.1 500     0.84 secs:     248 bytes ==> POST http://Order:8080/orders

* 건당 (쓰레드당) 처리시간이 610 밀리 미만으로 회복 -> 요청 수락

HTTP/1.1 201    18.35 secs:     228 bytes ==> POST http://Order:8080/orders
HTTP/1.1 201    18.35 secs:     228 bytes ==> POST http://Order:8080/orders

```
- 시스템은 과도한 Data 생성 요청에 대한 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 
하지만 75.5% 가 성공하고 31.4%가 실패했다는 것은 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가, HPA) 을 통하여 시스템을 확장 해주는 후속처리 필요.

### 오토스케일 아웃
Circuite Breaker 는 시스템을 안정되게 운영할 수 있게 해줬지만, 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 

- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 20프로를 넘어서면 replica 를 20개까지 늘려준다:
```
kubectl autoscale deploy payment --cpu-percent=20 --min=1 --max=20 -n books
```
- Circuite Breaker 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
siege -c100 -t120S -v --content-type "application/json" 'http://20.196.153.152:8080/orders POST {"bookId": "10", "qty": "1", "customerId":"1002"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy payment -w
```
- 어느 정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:

![image](https://user-images.githubusercontent.com/70673830/98115066-915df800-1ee9-11eb-9ebf-f2d79112bec9.png)


- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 

![image](https://user-images.githubusercontent.com/70673830/98115651-7f308980-1eea-11eb-833f-d606aaf6d6d9.png)



## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://customerview:8080/mypages POST {"orderId": "10", "qty": "1", "customerId": "1002"}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.00 secs:     269 bytes ==> POST http://customerview:8080/mypages
HTTP/1.1 201     0.01 secs:     269 bytes ==> POST http://customerview:8080/mypages
HTTP/1.1 201     0.01 secs:     269 bytes ==> POST http://customerview:8080/mypages
HTTP/1.1 201     0.01 secs:     269 bytes ==> POST http://customerview:8080/mypages
HTTP/1.1 201     0.01 secs:     269 bytes ==> POST http://customerview:8080/mypages
HTTP/1.1 201     0.01 secs:     269 bytes ==> POST http://customerview:8080/mypages
HTTP/1.1 201     0.01 secs:     269 bytes ==> POST http://customerview:8080/mypages
HTTP/1.1 201     0.02 secs:     269 bytes ==> POST http://customerview:8080/mypages
HTTP/1.1 201     0.03 secs:     269 bytes ==> POST http://customerview:8080/mypages
HTTP/1.1 201     0.00 secs:     269 bytes ==> POST http://customerview:8080/mypages
HTTP/1.1 201     0.00 secs:     269 bytes ==> POST http://customerview:8080/mypages
:

```

- 새버전으로 재배포 (Azure DevOps Pipelines)

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인

![image](https://user-images.githubusercontent.com/20619166/98184054-ca7b8400-1f4c-11eb-95ad-2949072ff912.png)


배포기간중 Availability 가 평소 100%에서 90% 로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

```
# deployment.yaml 의 readiness probe 의 설정 
  initialDelaySeconds: 10
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 10

kubectl apply -f kubernetes/deployment.yaml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:

![image](https://user-images.githubusercontent.com/20619166/98185439-fb10ed00-1f4f-11eb-8278-ae03158414fd.png)

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.



## Liveness Probe 점검

### 시나리오 1. 파일 상태 점검

5초 간격으로 특정 위치의 파일 생성 여부를 확인하고, 없으면 실패로 인식해서 프로세스를 Kill하고 다시 시작,
일정 시간 (30초)가 지나면 다시 파일을 삭제하고 Liveness 를 위한 서비스 수행한다.

### 설정 확인
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: orderLiveness
  name: order
  namespace: books
spec:
  containers:
  - name: order
    image: admin03.azurecr.io/order:V1
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```
#### 기존 서비스 삭제
```
kubectl delete service order -n books
service "order" deleted
```

#### 기존 deploy 삭제
```
kubectl delete deploy order -n books
deployment.apps "order" deleted
```

#### liveness 적용된 pod 생성
```
kubectl apply -f pod-exec-liveness.yaml
```

#### liveness 적용된 order pod 의 상태 체크( 테스트 결과 )
```
kubectl describe po order -n books
```

#### 테스트 결과 이미지
![image](https://user-images.githubusercontent.com/70673830/98134412-148b4800-1f02-11eb-9189-f38c401c0eb8.png)

### 시나리오 2. TCP 포트 점검

Order서비스의 deployment.yml의 liveness 설정을 tcp socket 방식의 8081 포트를 바라보도록 변경하여 restart여부를 확인한다.

![image](https://user-images.githubusercontent.com/20619166/98126463-f8cf7400-1ef8-11eb-9246-89f425031a86.png)
![image](https://user-images.githubusercontent.com/20619166/98126483-fd942800-1ef8-11eb-99d9-89481b2c62e4.png)
![image](https://user-images.githubusercontent.com/20619166/98126511-0553cc80-1ef9-11eb-9a56-b564c70466d4.png)

## Config Map
```
Order 서비스에 configmap.yml 파일을 생성한다.

apiVersion: v1
kind: ConfigMap
metadata:
  name: apipayurl
data:
  url:  http://payment:8080
```
```
Order 서버스의 deployment.yml에 configmap 파일을 참조할 수 있는 값을 추가한다.

          env:
            - name: payurl
              valueFrom:
                configMapKeyRef:
                  name: apipayurl
                  key: url
```
```
Order 서버스의 apllication.yml에 deployment에 추가된 값을 참조하도록 추가한다.

api:
  payment:
    url: ${payurl}
```
```
Order 서버스의 PaymentService.java에 외부 값을 보도록 변경한다.

@FeignClient(name="Payment", url="${api.payment.url}")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void payReq(@RequestBody Payment payment);

}
```
```
configmap.yml 파일의 url을 임의의 값으로 변경 후 order 서비스의 호출을 확인한다.

data:
  url:  http://payment:8088

root@labs--2023481703:~/src/bookmarket# http http://order:8080/orders bookId=101 qty=1 customerId=10002
HTTP/1.1 500 Internal Server Error
Content-Type: application/json;charset=UTF-8
Date: Wed, 04 Nov 2020 16:17:34 GMT
transfer-encoding: chunked

{
    "error": "Internal Server Error", 
    "message": "Could not commit JPA transaction; nested exception is javax.persistence.RollbackException: Error while committing the transaction", 
    "path": "/orders", 
    "status": 500, 
    "timestamp": "2020-11-04T16:17:34.521+0000"
}

```
