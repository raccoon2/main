![저기어때](https://user-images.githubusercontent.com/73212897/96727589-2d6a0880-13ee-11eb-91ab-acf6e0cc9526.jpg)


# 객실예약서비스
- 4조

# Table of contents

- [객실예약서비스](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
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
1. 숙박관리자는 객실을 등록할 수 있다
2. 고객이 객실을 선택하여 예약한다.
3. 예약이 완료되면 해당 객실은 '예약불가' 상태로 변경된다.
4. 결제가 완료되면 객실 예약이 확정된다.
5. 고객이 예약을 취소할 수 있다.
6. 숙박관리자가 객실을 체크아웃 하면 객실은 예약가능 상태로 변경된다.
7. 고객이 숙소 예약상태를 중간중간 조회한다.
8. 예약상태가 바뀔 때 마다 카톡으로 알림을 보낸다

비기능적 요구사항
1. 트랜잭션

    i. 결제가 되지 않은 예약건은 아예 예약이 완료되지 않아야 한다  Sync 호출 
2. 장애격리

    i. 객실관리시스템이 수행되지 않더라도 객실 예약은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    
    ii. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다  Circuit breaker, fallback
3. 성능

    i. 고객이 객실상태를 예약시스템(프론트엔드)에서 확인할 수 있어야 한다  CQRS    
    
    ii. 객실상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다 Event driven


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
    - 서킷브레이커를 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?

  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등
    - Contract Test : 자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?

# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
![image](https://user-images.githubusercontent.com/69283674/97150539-99af8800-17b1-11eb-8b8e-c6b8bcd1d537.png)

## TO-BE 조직 (Vertically-Aligned)
![image](https://user-images.githubusercontent.com/69283674/97150593-b350cf80-17b1-11eb-8c7f-68c5be80badc.png)

[조직 KPI]
예약팀 : 고객의 예약을 최대한 많이 받아야 함.
결제팀 : 결제 안정성을 최대한 확보해야 함.
객실팀 : 객실관리 편의성이 높아야 함.

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: 
https://www.msaez.io/#/storming/vK3Ti7jb85Q5GVnPwKO5ecQpjRJ2/every/a1a546e3387be89639c5a1c96210c4bb/-MKbLyDZELkpG9pKR7Qa



### 이벤트 도출
![image](https://user-images.githubusercontent.com/69283674/97150926-307c4480-17b2-11eb-8c70-251530e0d9fd.png)
    
### 부적격 이벤트 탈락
![image](https://user-images.githubusercontent.com/69283674/97151026-4ee24000-17b2-11eb-8b90-2e035743d335.png)

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
        - 결재시>결재버튼이 클릭됨  : UI 의 이벤트이지, 업무적인 의미의 이벤트가 아니라서 제외

### 액터 및 커맨드 부착하여 읽기 좋게

![image](https://user-images.githubusercontent.com/69283674/97151489-e8a9ed00-17b2-11eb-879d-75de8f6a129b.png)


### 어그리게잇으로 묶기

![image](https://user-images.githubusercontent.com/69283674/97152033-af25b180-17b3-11eb-91dd-cfe3e514cfd6.png)

### 바운디드 컨텍스트로 묶기

![image](https://user-images.githubusercontent.com/69283674/97152594-79cd9380-17b4-11eb-9bbe-5e0f02421917.png)

    - 도메인 서열 분리 
        - Core Domain:  Reservation : 없어서는 안될 핵심 서비스이며, 연간 Up-time SLA 수준을 99.999% 목표, 배포주기는 app 의 경우 1주일 1회 미만
        - Supporting Domain:   Room : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain:   Payment : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![image](https://user-images.githubusercontent.com/69283674/97152713-a4b7e780-17b4-11eb-9bb5-8f498bf5de5f.png)


### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![image](https://user-images.githubusercontent.com/69283674/97153018-2871d400-17b5-11eb-9773-276a38813eb9.png)


### 1차 완성본

![image](https://user-images.githubusercontent.com/69283674/97153154-63740780-17b5-11eb-904e-74bb90a97738.png)

### 기능적 요구사항을 커버하는지 검증(1)

![image](https://user-images.githubusercontent.com/69283674/97153235-81416c80-17b5-11eb-8d93-ab678184e2c7.png)

기능적 요구사항

    - 숙박관리자는 객실을 등록할 수 있다 (OK)
   
    - 고객이 객실을 선택하여 예약한다 (OK)
    
    - 예약이 완료되면 해당 객실은 '예약불가' 상태로 변경된다 (OK)
    
    - 결제가 완료되면 예약이 확정된다 (OK)

### 기능적 요구사항을 커버하는지 검증(2)

![image](https://user-images.githubusercontent.com/69283674/97153635-28260880-17b6-11eb-9e2c-225e1407d31f.png)

기능적 요구사항

    - 고객은 예약을 취소할 수 있다 (OK)
    
    - 예약이 취소되면 해당 객실은 '예약가능' 상태로 변경된다 (OK)
    
    - 고객이 숙소 예약상태를 중간중간 조회한다 (OK)
    
    - 예약상태가 바뀔때마다 알람을 보낸다 (OK)


### 비기능 요구사항에 대한 검증

![image](https://user-images.githubusercontent.com/69283674/97153970-ada9b880-17b6-11eb-877b-95b4f9813a7d.png)

1. 트랜잭션
 - 결제가 되지 않은 예약건은 아예 예약이 완료되지 않아야 한다 Sync 호출
 -> 예약 시 결제처리:  결제가 완료되지 않은 주문은 절대 받지 않는다는 경영자의 오랜 신념(?) 에 따라, ACID 트랜잭션 적용. 예약완료시 결제처리에 대해서는 Request-Response 방식 처리
 
2. 장애격리
 - 객실관리시스템이 수행되지 않더라도 객실 예약은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
 -> 결제 완료시 객실연결 및 확정처리:  App(front) 에서 Room 마이크로서비스로 예약요청이 전달되는 과정에 있어서 Room 마이크로 서비스가 별도의 배포주기를 가지기 때문에 Eventual Consistency 방식으로 트랜잭션 처리함.
 - 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 Circuit breaker, fallback
 
3. 성능
 - 고객이 객실상태를 수신하여 예약시스템(프론트엔드)에서 확인할 수 있어야 한다 CQRS
 - 예약상태가 바뀔때마다 카톡 등으로 알림을 줄 수 있어야 한다 Event driven
 -> 나머지 모든 inter-microservice 트랜잭션: 예약상태, 결제상태 등 모든 이벤트에 대해 카톡을 처리하는 등, 데이터 일관성의 시점이 크리티컬하지 않은 모든 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택함.



## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/69283674/97154475-40e2ee00-17b7-11eb-9bd6-1d19a6f9e157.png)

    - Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐
    

# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd Reservation
mvn spring-boot:run

cd Payment
mvn spring-boot:run 

cd Room
mvn spring-boot:run  

cd Notice
mvn spring-boot:run  

cd Gateway
mvn spring-boot:run  
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 pay 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 하지만, 일부 구현에 있어서 영문이 아닌 경우는 실행이 불가능한 경우가 있기 때문에 계속 사용할 방법은 아닌것 같다. (Maven pom.xml, Kafka의 topic id, FeignClient 의 서비스 id 등은 한글로 식별자를 사용하는 경우 오류가 발생하는 것을 확인하였다)

```
package accommodation;

import javax.persistence.*;

import accommodation.external.Payment;
import accommodation.external.PaymentManagementService;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import accommodation.config.kafka.KafkaProcessor;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.MessageHeaders;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.util.MimeTypeUtils;

@Entity
@Table(name="Reservation_table")
public class Reservation {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Integer reservationNumber;
    private Integer roomNumber;
    private Integer customerId;
    private String customerName;
    private String reserveStatus;
    private Integer PaymentPrice;

    public Integer getReservationNumber() {
        return reservationNumber;
    }

    public void setReservationNumber(Integer reservationNumber) {
        this.reservationNumber = reservationNumber;
    }
    public String getCustomerName() {
        return customerName;
    }

    public void setCustomerName(String customerName) {
        this.customerName = customerName;
    }
    public Integer getCustomerId() {
        return customerId;
    }

    public void setCustomerId(Integer customerId) {
        this.customerId = customerId;
    }
    public String getReserveStatus() {
        return reserveStatus;
    }

    public void setReserveStatus(String reserveStatus) {
        this.reserveStatus = reserveStatus;
    }
    public Integer getRoomNumber() {
        return roomNumber;
    }

    public void setRoomNumber(Integer roomNumber) {
        this.roomNumber = roomNumber;
    }

    public Integer getPaymentPrice() {
        return PaymentPrice;
    }

    public void setPaymentPrice(Integer paymentPrice) {
        PaymentPrice = paymentPrice;
    }


}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package accommodation;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface ReservationRepository extends PagingAndSortingRepository<Reservation, Integer>{

}
```
- 적용 후 REST API 의 테스트
```
#객실등록
http POST http://room:8080/rooms roomType="DERUX" roomStatus="EMPTY" roomName="DERUX ROOM" roomQty=10 roomPrice=1200000\

#객실 조회
http http://roomInfo:8080/roomInfoes 

#예약요청 (객실이 있을경우)
http post http://reservation:8080/reservations customerName="PARK CHANG SEON" customerId=9805 reserveStatus="reserve" roomNumber=101 paymentPrice=120000

#결제요청
http post http://reservation:8080/reservations reservationNumber=1 reserveStatus="payment" customerName="PARK CHANG SEON" customerId=9805 roomNumber=101 paymentPrice=120000

#결제 내역 조회
http http://payment:8080/payments

#Check Out
http post http://reservation:8080/reservations reservationNumber=1 reserveStatus="checkOut" customerName="PARK CHANG SEON" customerId=9805 roomNumber=101 paymentPrice=120000
```

![image](https://user-images.githubusercontent.com/69283674/97377996-9ed11c00-1904-11eb-9705-b81ec3e7399e.png)

![image](https://user-images.githubusercontent.com/69283674/97378172-199a3700-1905-11eb-9e97-5450b00e37df.png)

![image](https://user-images.githubusercontent.com/69283674/97378209-2ae34380-1905-11eb-9af7-f428c2c0ac81.png)

![image](https://user-images.githubusercontent.com/69283674/97378300-667e0d80-1905-11eb-816a-fdd22398372b.png)

![image](https://user-images.githubusercontent.com/69283674/97378332-7ac20a80-1905-11eb-85f2-be38ba7003f3.png)

![image](https://user-images.githubusercontent.com/69283674/97378372-90cfcb00-1905-11eb-9707-88259c10d5a3.png)





## 동기식 호출 

분석단계에서의 조건 중 하나로 예약(Reservation)->결제(Payment) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 결제서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# (payment) PaymentManagementService.java
package accommodation.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.*;

@org.springframework.stereotype.Service
@FeignClient(name="reservationNumber", url="${api.url.payment}")
public interface PaymentManagementService {

    @RequestMapping(method= RequestMethod.POST, path="/payments", consumes = "application/json")
    public void CompletePayment(@RequestBody Payment payment);

}
```

- 결제 요청 시 동기적으로 실행
```
# Reservation.java (Entity)

    @PreUpdate
    public void onPreUpdate(){
        ...
            Payment payment = new Payment();
            payment.setPaymentPrice(getPaymentPrice());
            payment.setReservationNumber(getReservationNumber());
            payment.setReservationStatus(getReserveStatus()); 
            
            Application.applicationContext.getBean(PaymentManagementService.class).CompletePayment(payment);
          
    }
```

- 동기식으로 결제 요청시 결제 시스템 장애의 경우 예약 불가 확인 :
```
# 결제 서비스 (pament) 를 잠시 내려놓음 
kubectl scale deploy room --replicas=0
```
![image](https://user-images.githubusercontent.com/69283674/97378907-db9e1280-1906-11eb-8ea9-09d6e20272d4.png)

```
#결제요청
http post http://reservation:8080/reservations reservationNumber=1 reserveStatus="payment" customerName="PARK" customerId=9805 roomNumber=1 paymentPrice=50001
```
![image](https://user-images.githubusercontent.com/69283674/97378963-ff615880-1906-11eb-973c-dd307e672a9b.png)


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


결제 이후 이를 예약 정보 변경은 비동기로 호출하여 개별적으로 조회 가능하도록 구현
 
- 이를 위하여 결제이력에 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package accommodation;

@Entity
@Table(name="Payment_table")
public class Payment   {

 ...
    @PrePersist
    public void onPrePersist(){
        if ("payment".equals(ReservationStatus) ) {
            System.out.println("=============결재 승인 처리중=============");
            PaymentCompleted paymentCompleted = new PaymentCompleted();

            setPaymentStatus("Y");
            paymentCompleted.setPaymentId(PaymentId);
            paymentCompleted.setReservationNumber(ReservationNumber);
            paymentCompleted.setPaymentPrice(PaymentPrice);
            paymentCompleted.setReservationStatus(ReservationStatus);
            paymentCompleted.setPaymentStatus(PaymentStatus);
            BeanUtils.copyProperties(this, paymentCompleted);
            paymentCompleted.publishAfterCommit();

        }
    }    
```
- 예약 서비스에서 해당 비동기 호출을 수신할 PolicyHandler를 구현

```
package accommodation;

...

@Service
public class PolicyHandler{

    @Autowired
    RoomRepository roomManagementRepository;
    
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverReserved_객실상태변경(@Payload Reserved reserved){
        if(reserved.isMe()){
            System.out.println("##### listener 객실상태변경 : " + reserved.toJson());
            if(roomManagementRepository.findById(reserved.getRoomNumber()) != null && "reserve".equals(reserved.getReserveStatus())){
                Room room = new Room();
                room.setRoomNo(reserved.getRoomNumber());
                room.setRoomStatus("RoomNotAvaliable");
                roomManagementRepository.save(room);
            }
        }
    }

}

```

객실 시스템은 예약/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 객실시스템이 유지보수로 인해 잠시 내려간 상태라도 예약을 받는데 문제가 없다:
```
# 객실 서비스 (room) 를 잠시 내려놓음 (ctrl+c)
kubectl scale deploy room --replicas=0
```
![image](https://user-images.githubusercontent.com/69283674/97379442-28ceb400-1908-11eb-8171-376af4ecdc6d.png)

```
#예약 및 결제처리
http post http://reservation:8080/reservations customerName="PARK" customerId=9805 reserveStatus="reserve" roomNumber=1 paymentPrice=50000
   #Success
http post http://reservation:8080/reservations reservationNumber=1 reserveStatus="payment" customerName="PARK" customerId=9805 roomNumber=101 paymentPrice=50001
   #Success
```
![image](https://user-images.githubusercontent.com/69283674/97379492-4734af80-1908-11eb-9164-74a2ad44d093.png)
![image](https://user-images.githubusercontent.com/69283674/97379504-50258100-1908-11eb-8d8d-81837dceb29d.png)


```
#객실상태 확인
http http://roomInfo:8080/roomInfoes     # 객실 상태 안바뀜 확인
```
![image](https://user-images.githubusercontent.com/69283674/97379568-6e8b7c80-1908-11eb-9def-e0a6d327cd2c.png)


```
#객실 서비스 기동
kubectl scale deploy room --replicas=1
```
![image](https://user-images.githubusercontent.com/69283674/97379616-8400a680-1908-11eb-8805-0ba9312dca5c.png)


```
#객실상태 확인
http localhost:8084/roomInfos     # 객실의 상태가 "NotavAilable"으로 확인
```
![image](https://user-images.githubusercontent.com/69283674/97379659-a09cde80-1908-11eb-85b0-5bbf408143ad.png)



## CQRS 패턴 
사용자 View를 위한 객실 정보 조회 서비스를 위한 별도의 객실 정보 저장소를 구현
- 이를 하여 RoomInfo 서비스를 별도로 구축하고 저장 이력을 기록한다.
- 모든 정보는 비동기 방식으로 호출한다.


```
Reservation.java(Entity)
package accommodation;

@Entity
@Table(name="Reservation_table")
public class Reservation {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Integer reservationNumber;
    private String customerName;
    private Integer customerId;
    private String reserveStatus;
    private Integer roomNumber;
    private Integer PaymentPrice;

    @PrePersist
    public void onPrePersist(){
        setReserveStatus("reserve");
        Reserved reserved = new Reserved();
        reserved.setReservationNumber(this.getReservationNumber());
        reserved.setReserveStatus(this.getReserveStatus());
        reserved.setCustomerName(this.getCustomerName());
        reserved.setCustomerId(this.getCustomerId());
        reserved.setRoomNumber(this.getRoomNumber());
        reserved.setPaymentPrice(this.getPaymentPrice());

        reserved.publishAfterCommit();
    }

```
```
Room.java(Entity)

@Entity
@Table(name="Room_table")
public class Room {
    ...

    @PostPersist
    public void onPostPersist(){
        RoomConditionChanged roomConditionChanged = new RoomConditionChanged();
        roomConditionChanged.setRoomNo(this.getRoomNo());
        roomConditionChanged.setRoomType(this.getRoomType());
        roomConditionChanged.setRoomStatus(this.getRoomStatus());
        roomConditionChanged.setRoomName(this.getRoomName());
        roomConditionChanged.setRoomQty(this.getRoomQty());
        roomConditionChanged.setRoomPrice(this.getRoomPrice());

        BeanUtils.copyProperties(this, roomConditionChanged);

        roomConditionChanged.publishAfterCommit();
    }
    @PostUpdate
    public void onPostUpdate(){
        System.out.println("예약가능?:"+this.roomStatus);
        RoomConditionChanged roomConditionChanged = new RoomConditionChanged();
        roomConditionChanged.setRoomNo(this.getRoomNo());
        roomConditionChanged.setRoomType(this.getRoomType());
        roomConditionChanged.setRoomStatus(this.getRoomStatus());
        roomConditionChanged.setRoomName(this.getRoomName());
        roomConditionChanged.setRoomQty(this.getRoomQty());
        roomConditionChanged.setRoomPrice(this.getRoomPrice());

        BeanUtils.copyProperties(this, roomConditionChanged);
        roomConditionChanged.publishAfterCommit();
        System.out.println("예약가능으로 변경");
    }
```

- RoomInfo에 저장하는 서비스 정책 (PolicyHandler)구현
```
PolicyHandler.java

@Service
public class PolicyHandler{

    @Autowired
    RoomInfoRepository roomInfoRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverSave_RoomInfo(@Payload RoomConditionChanged roomConditionChanged){
        if(roomConditionChanged.isMe()){
            System.out.println("##### listener 객실정보저장 : " + roomConditionChanged.toJson());
            RoomInfo roomInfo = new RoomInfo();
            roomInfo.setRoomNumber(roomConditionChanged.getRoomNumber());
            roomInfo.setRoomName(roomConditionChanged.getRoomName());
            roomInfo.setRoomStatus(roomConditionChanged.getRoomStatus());
            roomInfo.setRoomPrice(roomConditionChanged.getRoomPrice());
            roomInfo.setRoomQty(roomConditionChanged.getRoomQty());
            roomInfo.setRoomStatus(roomConditionChanged.getRoomStatus());
            roomInfo.setRoomType(roomConditionChanged.getRoomType());

            roomInfoRepository.save(roomInfo);
        }
    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverSave_Reserved(@Payload Reserved reserved) {
        if (reserved.isMe()) {
            // external message send
            System.out.println("##### listener 예약정보 저장 : " + reserved.toJson());
            RoomInfo roomInfo = new RoomInfo();
            roomInfo.setReserveNo(reserved.getReserveNo());
            roomInfo.setCustomerName(reserved.getCustomerName());
            roomInfo.setCustomerId(reserved.getCustomerId());
            roomInfo.setReservationStatus(reserved.getReserveStatus());
            roomInfo.setRoomNo(reserved.getRoomNo());
            roomInfo.setReservePrice(reserved.getReservePrice());
            
            roomInfoRepository.save(roomInfo);
        }
    }    

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverSave_PaymentCompleted(@Payload PaymentCompleted paymentCompleted){
        if(paymentCompleted.isMe()){
            // external message send
            System.out.println("##### listener 결제 정보저장 : " + paymentCompleted.toJson());
            RoomInfo roomInfo = new RoomInfo();
            roomInfo.setReservationNumber(paymentCompleted.getReservationNumber());
            roomInfo.setPaymentId(paymentCompleted.getPaymentId());
            roomInfo.setPaymentPrice(paymentCompleted.getPaymentPrice());
            roomInfo.setReservationStatus(paymentCompleted.getReservationStatus());

            roomInfoRepository.save(roomInfo);
        }
    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverSave_CheckOuted(@Payload CheckedOut checkedOut){
        if(checkedOut.isMe()){
            System.out.println("##### listener 체크아웃 정보저장 : " + checkedOut.toJson());
            RoomInfo roomInfo = new RoomInfo();
            roomInfo.setReserveNo(checkedOut.getReserveNo());
            roomInfo.setCustomerName(checkedOut.getCustomerName());
            roomInfo.setRoomNo(checkedOut.getRoomNo());
            roomInfo.setCustomerId(checkedOut.getCustomerId());
            roomInfo.setCustomerName(checkedOut.getCustomerName());
            roomInfo.setReservationStatus(checkedOut.getReserveStatus());
            
            roomInfoRepository.save(roomInfo);
        }
    }
}

```

# API 게이트웨이
Clous 환경에서는 //서비스명:8080 에서 Gateway API가 작동해야함 application.yml 파일에 profile별 gateway 설정
-  Gateway 설정 파일 

![image](https://user-images.githubusercontent.com/69283674/97270866-279f7780-1873-11eb-958f-90ed7d0eec47.png)


# 운영

## Deploy 

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AZURE를 사용하였다.

-  아래와 같이 pod 가 정상적으로 올라간 것을 확인하였다. 

![image](https://user-images.githubusercontent.com/69283674/97273735-10628900-1877-11eb-9f62-2d5fccea4c13.png)

-  아래와 같이 쿠버네티스에 모두 서비스로 등록된 것을 확인할 수 있다. 

![image](https://user-images.githubusercontent.com/69283674/97273520-d6918280-1876-11eb-8530-1447d2000735.png)


## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 예약(Reservation)-->결제(Payment) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- 피호출 서비스(결제:Payment) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# (Payment) Payment.java (Entity)

    @PrePersist
    public void onPrePersist(){  //결제이력을 저장한 후 적당한 시간 끌기

        ...
        
        try {
                Thread.currentThread().sleep((long) (400 + Math.random() * 220));
                System.out.println("=============결재 승인 완료=============");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
    }
```


### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy payment --min=1 --max=10 --cpu-percent=15
```
![image](https://user-images.githubusercontent.com/69283674/97295328-7b6d8900-1892-11eb-9581-f40d9b09b5de.png)


- 결제서비스의 deployment.yaml의 spec에 아래와 같이 자원속성을 설정한다:

![image](https://user-images.githubusercontent.com/69283674/97291666-8f62bc00-188d-11eb-9594-c14a11328bb0.png)

- CB 에서 했던 방식대로 워크로드를 1분 동안 걸어준다.
```
siege -c100 -t60S -content-type "application/json" 'http://payment:8080/payments'

```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy payment -w
```

- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 

![image](https://user-images.githubusercontent.com/69283674/97295982-5fb6b280-1893-11eb-89ef-741b220b2201.png)


### 무정지 재배포

- 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함
- seige 로 배포작업 직전에 워크로드를 모니터링 함.
![image](https://user-images.githubusercontent.com/69283674/97297532-837af800-1895-11eb-868d-f7c70ab6b3c6.png)


- 새버전으로의 배포 시작

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인

- 동일한 시나리오로 재배포 한 후 Availability 확인:

![image](https://user-images.githubusercontent.com/69283674/97297629-a1e0f380-1895-11eb-89f3-acc70aa3a30c.png)

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.
