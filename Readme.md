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
    
![image](https://user-images.githubusercontent.com/70673830/98327204-5cab8700-2036-11eb-8c31-72d97c3c5469.png)

    - 이벤트 흐름에서 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출 관계에서 Pub/Sub 과 Req/Resp 를 구분함
    - 바운디드 컨텍스트에 서브 도메인을 1 대 1 모델링하고 팀원별 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 Bounded Context 별로 대변되는 마이크로 서비스들을 Spring Boot 로 구현하였다. 
구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다. (포트 넘버는 8081 ~ 8084, 8086, 8088 이다)

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

cd Kakaom
mvn spring-boot:run 
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 Kakao 마이크로 서비스)

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

시나리오는 과도한 메시지 요청 시 "circuitBreaker.requestVolumeThreshold"의 옵션을 통한 장애격리 구현.

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
# Kakao.java (Entity)

![image](https://user-images.githubusercontent.com/70673849/98315182-fe24df80-201a-11eb-87de-c11980769635.png)
```

## 부하 발생을 통한 Circuit Breaker 점검
```
root@siege:/# siege -c100 -t30S -v --content-type "application/json" 'http://kakao:8080/kakaos POST {"orderId": "111", "customerId": "33", "status": "Shipped", "message": "sss"}'
** SIEGE 4.0.4
** Preparing 100 concurrent users for battle.
The server is now under siege...
HTTP/1.1 201     1.18 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     1.19 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     1.17 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     1.16 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     1.19 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     1.18 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.19 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.20 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.30 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.28 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.43 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.45 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.45 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.45 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.61 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.62 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.80 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.78 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.84 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.88 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.94 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.96 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     3.99 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     4.02 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     4.15 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     4.15 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     4.27 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     4.31 secs:     241 bytes ==> POST http://kakao:8080/kakaos

* 과도한 요청으로 CB 작동 -> 요청 차단

HTTP/1.1 500     4.75 secs:     820 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 500     4.75 secs:     820 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 500     4.74 secs:     820 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 500     4.75 secs:     820 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 500     4.75 secs:     820 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 500     4.74 secs:     820 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 500     4.74 secs:     820 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 500     4.75 secs:     820 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 500     4.74 secs:     820 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 500     4.82 secs:     820 bytes ==> POST http://kakao:8080/kakaos

* 정상 

HTTP/1.1 201     0.83 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.89 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.48 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.48 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.55 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.56 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.56 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.63 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.64 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.64 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.85 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.79 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.56 secs:     241 bytes ==> POST http://kakao:8080/kakaos
^C
Lifting the server siege...
Transactions:                     78 hits
Availability:                  88.64 %
Elapsed time:                   5.77 secs
Data transferred:               0.03 MB
Response time:                  2.87 secs
Transaction rate:              13.52 trans/sec
Throughput:                     0.00 MB/sec
Concurrency:                   38.76
Successful transactions:          78
Failed transactions:              10
Longest transaction:            4.82
Shortest transaction:           0.47

```
- 시스템은 과도한 Data 생성 요청에 대한 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 
하지만 88.64성공하고 12.3%가 실패했다는 것은 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가, HPA) 을 통하여 시스템을 확장 해주는 후속처리 필요.

### 오토스케일 아웃
Circuite Breaker 는 시스템을 안정되게 운영할 수 있게 해줬지만, 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 

- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 20프로를 넘어서면 replica 를 20개까지 늘려준다:
```
kubectl autoscale deploy kakao --cpu-percent=20 --min=1 --max=20
```
- Circuite Breaker 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
root@siege:/# siege -c100 -t30S -v --content-type "application/json" 'http://kakao:8080/kakaos POST {"orderId": "111", "customerId": "33", "status": "Shipped", "message": "sss"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy payment -w
```
- 어느 정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:

NAME                               READY   STATUS    RESTARTS   AGE
pod/customerview-bfc749846-vkknm   1/1     Running   0          11h
pod/delivery-d77875bb4-mdwn6       1/1     Running   0          5h59m
pod/gateway-7fbdf4c96-2sc9z        1/1     Running   0          6h41m
pod/kakao-5b56b8b686-xwd6n         1/1     Running   2          10m
pod/kakaomsg-558c69446d-nxlwk      1/1     Running   0          5h48m
pod/order-76c9c8494-4449p          1/1     Running   1          9h
pod/payment-7dd7984c8f-k7kzt       1/1     Running   0          11h
pod/siege                          1/1     Running   0          75m

NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
service/customerview   ClusterIP      10.0.161.71    <none>          8080/TCP         18h
service/delivery       ClusterIP      10.0.205.39    <none>          8080/TCP         18h
service/gateway        LoadBalancer   10.0.103.11    40.82.157.161   8080:31007/TCP   8h
service/kakao          ClusterIP      10.0.203.207   <none>          8080/TCP         9h
service/kakaomsg       ClusterIP      10.0.230.187   <none>          8080/TCP         6h5m
service/kubernetes     ClusterIP      10.0.0.1       <none>          443/TCP          21h
service/order          ClusterIP      10.0.106.158   <none>          8080/TCP         20h
service/payment        ClusterIP      10.0.24.255    <none>          8080/TCP         19h

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/customerview   1/1     1            1           18h
deployment.apps/delivery       1/1     1            1           18h
deployment.apps/gateway        1/1     1            1           8h
deployment.apps/kakao          1/1     1            1           11h
deployment.apps/kakaomsg       1/1     1            1           6h5m
deployment.apps/order          1/1     1            1           20h
deployment.apps/payment        1/1     1            1           19h

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/customerview-5977db95fd   0         0         0       18h
replicaset.apps/customerview-5d58796fbf   0         0         0       18h
replicaset.apps/customerview-bfc749846    1         1         1       11h
replicaset.apps/delivery-5d864f56d4       0         0         0       11h
replicaset.apps/delivery-6c75956d69       0         0         0       5h59m
replicaset.apps/delivery-8447b6fcdb       0         0         0       18h
replicaset.apps/delivery-84ff995cdd       0         0         0       15h
replicaset.apps/delivery-857d98dfcb       0         0         0       18h
replicaset.apps/delivery-d77875bb4        1         1         1       5h59m
replicaset.apps/gateway-5674b9d487        0         0         0       7h49m
replicaset.apps/gateway-7768c6bf4d        0         0         0       7h43m
replicaset.apps/gateway-7795789679        0         0         0       8h
replicaset.apps/gateway-7fbdf4c96         1         1         1       6h41m
replicaset.apps/gateway-7ff8c947f8        0         0         0       7h14m
replicaset.apps/gateway-855459c4f8        0         0         0       8h
replicaset.apps/gateway-d8b79ff46         0         0         0       7h56m
replicaset.apps/gateway-f8cd5ff65         0         0         0       8h
replicaset.apps/kakao-544974946d          0         0         0       22m
replicaset.apps/kakao-59d7495d7b          0         0         0       11h
replicaset.apps/kakao-5b56b8b686          1         1         1       10m
replicaset.apps/kakao-5bb47c898           0         0         0       3h3m
replicaset.apps/kakao-66f479bb7d          0         0         0       9h
replicaset.apps/kakao-6c68dbbbdf          0         0         0       36m
replicaset.apps/kakao-76f569ffb4          0         0         0       124m
replicaset.apps/kakao-76ffd8555c          0         0         0       9h
replicaset.apps/kakao-7bc9859ddc          0         0         0       135m
replicaset.apps/kakao-b55bd5c4            0         0         0       32m
replicaset.apps/kakao-fcbb5878c           0         0         0       27m
replicaset.apps/kakaomsg-558c69446d       1         1         1       5h48m
replicaset.apps/kakaomsg-57d7cf5969       0         0         0       5h54m
replicaset.apps/kakaomsg-5c4985b88b       0         0         0       6h5m
replicaset.apps/kakaomsg-795894f65f       0         0         0       6h5m
replicaset.apps/order-5bb6cb44c8          0         0         0       19h
replicaset.apps/order-656675c45d          0         0         0       9h
replicaset.apps/order-66fdf5896f          0         0         0       13h
replicaset.apps/order-67bd5d447d          0         0         0       20h
replicaset.apps/order-6db44b9569          0         0         0       9h
replicaset.apps/order-6f584d4b9           0         0         0       15h
replicaset.apps/order-764bd469fc          0         0         0       9h
replicaset.apps/order-76c9c8494           1         1         1       9h
replicaset.apps/order-bd9f5f459           0         0         0       9h
replicaset.apps/order-f8c7c5db8           0         0         0       13h
replicaset.apps/order-f959cf8b            0         0         0       11h
replicaset.apps/payment-54bdf678b7        0         0         0       19h
replicaset.apps/payment-69575977c4        0         0         0       15h
replicaset.apps/payment-7dd7984c8f        1         1         1       11h
replicaset.apps/payment-d87b4d88f         0         0         0       19h

NAME                                        REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/kakao   Deployment/kakao   <unknown>/20%   1         20        1          7m30s



## Liveness Probe 점검

### 시나리오 1. 파일 상태 점검

5초 간격으로 특정 위치의 파일 생성 여부를 확인하고, 없으면 실패로 인식해서 프로세스를 Kill하고 다시 시작,
일정 시간 (30초)가 지나면 다시 파일을 삭제하고 Liveness 를 위한 서비스 수행한다.

### 설정 확인

![image](https://user-images.githubusercontent.com/70673849/98324399-7bf2e600-202f-11eb-9eca-79b36ca342a5.png)

#### liveness 적용된 order pod 의 상태 체크( 테스트 결과 )

![image](https://user-images.githubusercontent.com/70673849/98324495-c2e0db80-202f-11eb-82a0-baa25213ad8c.png)


## 무정지 재배포 (Readness Probe)

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler, CB 설정을 제거함
![image](https://user-images.githubusercontent.com/70673849/98325560-6e8b2b00-2032-11eb-930e-8de0939850c3.png)

![image](https://user-images.githubusercontent.com/70673849/98327879-e7d94c80-2037-11eb-95cd-4071deedfe26.png)

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인

root@siege:/# siege -c100 -t10S -v --content-type "application/json" 'http://kakao:8080/kakaos POST {"orderId": "111", "customerId": "33", "status": "Shipped", "message": "sss"}'
** SIEGE 4.0.4
** Preparing 100 concurrent users for battle.
The server is now under siege...
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused

Lifting the server siege...
Transactions:                      0 hits
Availability:                   0.00 %
Elapsed time:                   9.51 secs
Data transferred:               0.00 MB
Response time:                  0.00 secs
Transaction rate:               0.00 trans/sec
Throughput:                     0.00 MB/sec
Concurrency:                    0.00
Successful transactions:           0
Failed transactions:               9
Longest transaction:            0.00
Shortest transaction:           0.00
 
root@siege:/# siege -c100 -t10S -v --content-type "application/json" 'http://kakao:8080/kakaos POST {"orderId": "111", "customerId": "33", "status": "Shipped", "message": "sss"}'
** SIEGE 4.0.4
** Preparing 100 concurrent users for battle.
The server is now under siege...
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused

Lifting the server siege...
Transactions:                      0 hits
Availability:                   0.00 %
Elapsed time:                   9.41 secs
Data transferred:               0.00 MB
Response time:                  0.00 secs
Transaction rate:               0.00 trans/sec
Throughput:                     0.00 MB/sec
Concurrency:                    0.00
Successful transactions:           0
Failed transactions:               9
Longest transaction:            0.00
Shortest transaction:           0.00
 
root@siege:/# siege -c100 -t10S -v --content-type "application/json" 'http://kakao:8080/kakaos POST {"orderId": "111", "customerId": "33", "status": "Shipped", "message": "sss"}'
** SIEGE 4.0.4
** Preparing 100 concurrent users for battle.
The server is now under siege...
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
[error] socket: unable to connect sock.c:249: Connection refused
HTTP/1.1 201     1.31 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.03 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.02 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.06 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.02 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.06 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.03 secs:     239 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.02 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.01 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.01 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.02 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.00 secs:     241 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.02 secs:     241 bytes ==> POST http://kakao:8080/kakaos
...
HTTP/1.1 201     0.44 secs:     243 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.50 secs:     243 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.43 secs:     243 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.36 secs:     243 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.29 secs:     243 bytes ==> POST http://kakao:8080/kakaos
HTTP/1.1 201     0.23 secs:     243 bytes ==> POST http://kakao:8080/kakaos

Lifting the server siege...
Transactions:                    756 hits
Availability:                  90.60 %
Elapsed time:                   9.42 secs
Data transferred:               0.17 MB
Response time:                  1.21 secs
Transaction rate:              80.25 trans/sec
Throughput:                     0.02 MB/sec
Concurrency:                   96.79
Successful transactions:         756
Failed transactions:               3
Longest transaction:            8.67
Shortest transaction:           0.00

배포기간중 Availability 가 평소 100%에서 90% 로 떨어지는 것을 확인. 이를 막기위해 Readiness Probe 설정 필요

```
# deployment.yaml 의 readiness probe 의 설정 
  initialDelaySeconds: 10
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 10

