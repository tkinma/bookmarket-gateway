# Book Market

- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [Book Market](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오

기능적 요구사항
1. 고객은 책을 주문한다 ( Core Domain - Order )
1. 고객이 주문을 할때는 반드시 결제가 되어야 한다.(Req/Rep) ( Circuit Breaker(결제 지연))
1. 결제가 완료되면 배송을 시작한다. ( Pub / Sub Event Dirven )
1. 결제완료되면 주문 상태를 변경한다 ( Pub / Sub Event Dirven )
1. 배송이 시작되면 주문 상태를 변경한다 ( Pub / Sub Event Dirven )
1. 고객은 주문을 취소한다.
1. 주문이 취소되면 결제를 취소하여 고객에게 환불한다. ( Pub / Sub Event Dirven )
1. 결제가 취소되면 배송을 취소한다. ( Pub / Sub Event Dirven )
1. 결제, 주문이 취소되면 주문상태를 변경한다. ( Pub / Sub Event Dirven )

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다  Sync 호출 
1. 장애격리
    1. 배송 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 고객이 주문상태를 시스템에서 확인할 수 있어야 한다  CQRS


# 체크포인트

- 분석 설계


  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/8PeWNorN2DZmr9eaKXl5ZKjKrJJ3/share/e97a1124efe99dc1edb15f8453d6a125/-MLBdLSL1aMfQCoHSb7u


### 이벤트 도출
![image](https://user-images.githubusercontent.com/20619166/98074092-0c54ed80-1ead-11eb-8801-cea6c8e76cf7.png)

    - 도메인 서열 분리 
        - Core Domain:  Order : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 Order 의 경우 1주일 1회 미만
        - Supporting Domain:   Delivery : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain:   Payment : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (핑크색으로 이후 전환할 예정)

## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/20619166/98073892-acf6dd80-1eac-11eb-99ec-0a7521d96aca.PNG)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd Order
mvn spring-boot:run

cd Payment
mvn spring-boot:run 

cd Delivery
mvn spring-boot:run  

cd customerview
mvn spring-boot:run 
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 Order 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 영문으로 하여 사용하려고 노력했다.

```
package bookmarket;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long bookId;
    private Long qty;
    private String status;
    private Long customerId;

    @PostPersist
    public void onPostPersist(){
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.setStatus("Ordered");
        ordered.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        bookmarket.external.Payment payment = new bookmarket.external.Payment();
        payment.setOrderId(this.getId());
        payment.setStatus("Ordered");
        payment.setCustomerId(this.getCustomerId());
        // mappings goes here
        OrderApplication.applicationContext.getBean(bookmarket.external.PaymentService.class)
            .payReq(payment);


    }

    @PreRemove
    public void onPreRemove(){
        OrderCanceled orderCanceled = new OrderCanceled();
        BeanUtils.copyProperties(this, orderCanceled);
        orderCanceled.setStatus("OrderCanceled");
        orderCanceled.publishAfterCommit();
    }


    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getBookId() {
        return bookId;
    }

    public void setBookId(Long bookId) {
        this.bookId = bookId;
    }
    public Long getQty() {
        return qty;
    }

    public void setQty(Long qty) {
        this.qty = qty;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
    public Long getCustomerId() {
        return customerId;
    }

    public void setCustomerId(Long customerId) {
        this.customerId = customerId;
    }
}


```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package bookmarket;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface OrderRepository extends PagingAndSortingRepository<Order, Long>{

}
```
- 적용 후 REST API 의 테스트
```
# Order 서비스의 주문처리
http localhost:8081/orders bookId=10 qty=20 customerId=1001
```
![image](https://user-images.githubusercontent.com/70673830/98118621-dafd1180-1eee-11eb-9899-768519ae80cc.png)

```
# Order 서비스의 주문 상태 확인
http localhost:8081/orders/1
```
![image](https://user-images.githubusercontent.com/70673830/98118737-fff18480-1eee-11eb-92a7-3075aece0ec1.png)


## 폴리글랏 퍼시스턴스

Delivery 서비스에는 H2 DB 대신 HSQL DB를 사용하기로 하였다. 이를 위해 메이븐 설정(pom.xml)상 DB 정보를 HSQLDB를 사용하도록 변경하였다.

![image](https://user-images.githubusercontent.com/20619166/98075211-4fb05b80-1eaf-11eb-9219-d848180c21bd.png)

![image](https://user-images.githubusercontent.com/20619166/98075210-4f17c500-1eaf-11eb-92d1-3d3731bc4e0c.png)

![image](https://user-images.githubusercontent.com/70673830/98119038-68406600-1eef-11eb-803e-a234638ac717.png)


## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문(Order)->결제(Payment) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 결제서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# (Order) PaymentService.java


package bookmarket.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="Payment", url="${api.payment.url}")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void payReq(@RequestBody Payment payment);

}
```

- 주문을 받은 직후(@PostPersist) 결제를 요청하도록 처리
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.setStatus("Ordered");
        ordered.publishAfterCommit();

        bookmarket.external.Payment payment = new bookmarket.external.Payment();
        payment.setOrderId(this.getId());
        payment.setStatus("Ordered");
        payment.setCustomerId(this.getCustomerId());
        // mappings goes here
        OrderApplication.applicationContext.getBean(bookmarket.external.PaymentService.class)
            .payReq(payment);
    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:


```
# 결제 (Payment) 서비스를 잠시 내려놓음 (ctrl+c)

#주문처리
http localhost:8081/orders bookId=2 qty=1 customerId=1002   #Fail

```
![image](https://user-images.githubusercontent.com/70673830/98119212-a89fe400-1eef-11eb-8b8e-196a219b0f38.png)

```
#결제서비스 재기동
cd Payment
mvn spring-boot:run

#주문처리
http localhost:8081/orders bookId=1 qty=1 customerId=1001   #Success
http localhost:8081/orders bookId=2 qty=1 customerId=1002   #Success
```

![image](https://user-images.githubusercontent.com/70673830/98119273-bce3e100-1eef-11eb-9095-5ab722c00185.png)

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)




## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


결제가 이루어진 후에 배송서비스로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 배송서비스의 처리를 위하여 결제주문이 블로킹 되지 않아도록 처리한다.
 
- 이를 위하여 결제이력에 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package bookmarket;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Payment_table")
public class Payment {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long orderId;
    private String status;
    private Long customerId;

    @PostPersist
    public void onPostPersist(){
        Paid paid = new Paid();
        BeanUtils.copyProperties(this, paid);
        paid.publishAfterCommit();
    }
```
- 배송 서비스에서는 결제승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
package bookmarket;

import bookmarket.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class PolicyHandler{
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @Autowired
    DeliveryRepository deliveryRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaid_Ship(@Payload Paid paid){

        if(paid.isMe()){
            System.out.println("##### listener Ship : " + paid.toJson());
            Delivery delivery = new Delivery();
            delivery.setOrderId(paid.getOrderId());
            delivery.setCustomerId(paid.getCustomerId());
            delivery.setStatus("Shipped");

            deliveryRepository.save(delivery);
        }
    }

배송서비스는 주문/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 배송 서비스가 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:

# 배송서비스 (Delivery) 를 잠시 내려놓음 (ctrl+c)

#주문처리
http localhost:8081/orders bookId=2 qty=1 customerId=1002   #Success
```
![image](https://user-images.githubusercontent.com/70673830/98119447-f7e61480-1eef-11eb-958b-4faf1dee47b1.png)
```
#주문상태 확인
http localhost:8081/orders     # 주문상태 안바뀜 확인
```
![image](https://user-images.githubusercontent.com/70673830/98119540-121ff280-1ef0-11eb-93fc-5982582757c2.png)

```
#배송 서비스 기동
cd Delivery
mvn spring-boot:run

#주문상태 확인
http localhost:8081/orders     # 주문의 상태가 "shipped"으로 확인
```
![image](https://user-images.githubusercontent.com/70673830/98119616-3380de80-1ef0-11eb-8760-64d746230321.png)

## CQRS
customerview(mypage)를 통해 구현하였다.

![image](https://user-images.githubusercontent.com/70673830/98119678-48f60880-1ef0-11eb-955e-a99ef278f2d3.png)



## gateway
gateway 프로젝트 내 application.yml

![image](https://user-images.githubusercontent.com/70673830/98119760-65924080-1ef0-11eb-8c21-078c5811c4e0.png)

![image](https://user-images.githubusercontent.com/70673830/98119815-7a6ed400-1ef0-11eb-9576-028614349553.png)

# 운영

## CI/CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 GCP를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 cloudbuild.yml 에 포함되었다.


## Circuit Breaker 점검

시나리오는 주문(Order)-->결제(Payment) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청에 orderId가 미존재 시 "circuitBreaker.requestVolumeThreshold"의 옵션을 통한 n개 이상 결제 요청 시 CB 를 통하여 장애격리.

```
Hystrix Command
	5000ms 이상 Timeout 발생 시 CircuitBearker 발동

CircuitBeaker 발생
	http http://localhost:8080/selectPaymentInfo?orderId=0
		- 잘못된 쿼리 수행 시 CircuitBeaker
		- 10000ms(10sec) Sleep 수행
		- 5000ms Timeout으로 CircuitBeaker 발동
		- 10000ms(10sec) 
    - 1건 이상 발생 시 발동
```

![image](https://user-images.githubusercontent.com/70673830/98113539-28758080-1ee7-11eb-8b34-dab272e9f122.png)
#### 소스 코드
```
# PaymentController.java
 @GetMapping("/selectPaymentInfo")
 @HystrixCommand(fallbackMethod = "fallbackPayment", commandProperties = {
         @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "5000"),
         @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "10000"),
         @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "10"),
         @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "1"),
         @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "10000")
 })
 public String selectPaymentInfo(@RequestParam long orderId) throws InterruptedException {

  if (orderId <= 0) {
   Thread.sleep(10000);
  } else {
   Optional<Payment> payment = paymentRepository.findById(orderId);
   return payment.get().getPaymentStatus();
 }
  
```
![image](https://user-images.githubusercontent.com/70673830/98113341-ddf40400-1ee6-11eb-8d24-7517b4494d10.png)

- 피호출 서비스(결제:Payment) 의 timeoutInMilliseconds의 5초 이후는 아래의 CD에 격리 처리
```
# private String fallbackDelivery(long orderId) {
  return "CircuitBreaker!!!";
 }
```
#### 실행 결과
![image](https://user-images.githubusercontent.com/70673830/98113470-0e3ba280-1ee7-11eb-8830-7c82b27ce0ba.png)

* 결제가 이루어 지지 않은 비정상적인 호출에 대한 CD:
- 10초 동안 격리 실시
- 1번 이상 orderId없을 시 격리



- 운영시스템은 비정상적인 접속 및 과도한 Data 조회에 대한 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 

### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 20프로를 넘어서면 replica 를 20개까지 늘려준다:
```
kubectl autoscale deploy payment --cpu-percent=20 --min=1 --max=20 -n books
```
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
siege -c100 -t120S -v --content-type "application/json" 'http://20.196.153.152:8080/orders POST {"bookId": "10", "qty": "1", "customerId":"1002"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy payment -w
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:

![image](https://user-images.githubusercontent.com/70673830/98115066-915df800-1ee9-11eb-9ebf-f2d79112bec9.png)


- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 

![image](https://user-images.githubusercontent.com/70673830/98115651-7f308980-1eea-11eb-833f-d606aaf6d6d9.png)




## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"bookId": "10", "qty": "1", "customerId": "1002"}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
:

```

- 새버전으로의 배포 시작
```
kubectl set image ...
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:		        3078 hits
Availability:		       70.45 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```
배포기간중 Availability 가 평소 100%에서 70% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

```
# deployment.yaml 의 readiness probe 의 설정:


kubectl apply -f kubernetes/deployment.yaml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Transactions:		        3078 hits
Availability:		       100 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.

## Liveness Probe 점검
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

#### 5. 실습 결과
```
(pwd 로 현 위치가 /container-orchestration/yaml/liveness/ 인지 확인)
(Liveness Command Probe 실습)
kubectl create -f exec-liveness.yaml
(컨테이너가 Running 상태로 보이나, Liveness Probe 실패로 계속 재시작)
(kubectl describe로 실패 메시지 확인)
kubectl describe po liveness-exec
(Liveness HTTP Probe 실습)
kubectl create -f http-liveness.yaml
(kubectl describe로 실패 메시지 확인)
kubectl describe po liveness-http
(Liveness 와 readiness probe 동시 적용 실습)
kubectl create -f tcp-liveness-readiness.yaml
(8080포트에 대해 정상적으로 Liveness 와 readiness Probe를 통과해 서비스가 실행됨)
kubectl describe po goproxy
```
![image](https://user-images.githubusercontent.com/70673830/98134412-148b4800-1f02-11eb-9189-f38c401c0eb8.png)


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
