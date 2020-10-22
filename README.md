# bookrental

# Table of contents

- [도서대여](#---)
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
    - [Configmap](#configmap)
    
# 서비스 시나리오

기능적 요구사항
1. 관리자는 도서를 등록할 수 있다.
2. 고객이 도서를 선택해 예약하면 결제가 진행된다.
3. 예약이 결제되면 도서의 예약 가능 여부가 변경된다.
4. 도서가 예약 불가 상태로 변경되면 예약이 확정(대여가능)된다.
5. 고객은 예약을 취소할 수 있다.
6. 예약이 취소되면 결제가 취소되고 도서의 예약 가능 여부가 변경된다.
7. 고객은 도서의 예약가능여부를 확인할 수 있다.

비기능적 요구사항
1. 트랜잭션
    1. 도서 예약은 결제가 취소된 경우 확정이 불가능 해야한다.(Sync 호출)
    1. 결제가 완료 되지 않은 예약 건은 예약이 성립되지 않는다.(Sync 호출)
    1. 예약과 결제는 동시에 진행된다.(Sync 호출)
    1. 예약 취소와 결제 취소는 동시에 진행된다.(Sync 호출)
1. 장애격리
    1. 대여 시스템이 수행되지 않더라도 예약 / 결제는365일 24시간 받을 수 있어야 한다.(Async 호출-event-driven)
    1. 대여 시스템이 과중 되면 예약 / 결제를 받지 않고 결제 취소를 잠시 후에 하도록 유도한다.(Circuit breaker, fallback)
    1. 결제가 취소되면 도서의 예약 취소가 확정되고, 도서의 예약 가능 여부가 변경된다.(Circuit breaker, fallback)
1. 성능
    1. 고객은 도서 예약 가능 여부를 확인할 수 있다.(CQRS)
    1. 예약/결제 취소 정보가 변경 될 때마다 도서 예약 가능 여부가 변경 될 수 있어야 한다.(Event Driven)

# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/40aEL4esvONUyUTOd8WTe3G2N0E3/mine/25597718266cbd8f4a1a3dba7773c803/-MK7g74T3CwyIVYwZSEP

### 이벤트 도출
![image](https://user-images.githubusercontent.com/70302903/96678702-31c20180-13ad-11eb-8bc8-310898d2a648.PNG)

### 어그리게잇으로 묶기 / 액터, 커맨드 부착하여 읽기 좋게
![image](https://user-images.githubusercontent.com/70302903/96678703-32f32e80-13ad-11eb-8025-0e62eeae234d.PNG)

    - 도서 예약, 결제, 도서 관리 등은 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 묶어준다.

### 바운디드 컨텍스트로 묶기

![image](https://user-images.githubusercontent.com/70302903/96678704-338bc500-13ad-11eb-8754-20c0b3d45e2c.PNG)

    - 도메인 서열 분리 
        - 예약 : 고객 예약 오류를 최소화 한다. (Core)
        - 결제 : 결제 오류를 최소화 한다. (Supporting)
        - 도서 : 도서 예약 상태 오류를 최소화 한다. (Supporting)

### 폴리시 부착 (괄호는 수행주체)
![image](https://user-images.githubusercontent.com/70302903/96678706-338bc500-13ad-11eb-8196-9315714cd5a3.PNG)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)
![image](https://user-images.githubusercontent.com/70302903/96678707-34245b80-13ad-11eb-8b02-d9d0b59baef9.PNG)

### 완성된 1차 모형
![image](https://user-images.githubusercontent.com/70302903/96678708-34bcf200-13ad-11eb-9d37-a46bca4061f9.PNG)

### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증
![image](https://user-images.githubusercontent.com/70302903/96678710-35558880-13ad-11eb-8d21-99753fc7e889.png)

    Red
    - 고객이 도서를 선택해 예약을 진행한다. (OK)
    - 예약 시 자동으로 결제가 진행된다. (OK)
    - 결제가 성공하면 도서가 예약불가 상태가 된다. (OK)
    - 도서 상태 변경 시 예약이 확정(대여가능)상태가 된다. (OK) 
    Blue
    - 고객이 예약/결제를 취소한다. (OK)
    - 예약/결제 취소 시 자동 예약/결제 취소된다. (OK)
    - 결제가 취소되면 도서가 예약가능 상태가 된다. (OK)

### 모델 수정
![image](https://user-images.githubusercontent.com/70302903/96678713-35ee1f00-13ad-11eb-9f00-cb8ac9f4b8a4.PNG)
    
    - View Model 추가(CQRS 적용)
    - 수정된 모델은 모든 요구사항을 커버함.

### 비기능 요구사항에 대한 검증
![image](https://user-images.githubusercontent.com/70302903/96678715-3686b580-13ad-11eb-9dd7-ad75ad58c508.png)

    - 도서 등록 서비스를 예약/결제 서비스와 격리하여 도서 등록 서비스 장애 시에도 예약이 가능
    - 도서가 예약 불가 상태일 경우 예약 확정이 불가함
    - 먼저 결제가 이루어진 도서에 대해서는 예약을 불가 하도록 함.

## 헥사고날 아키텍처 다이어그램 도출    
![image](https://user-images.githubusercontent.com/70302903/96678717-371f4c00-13ad-11eb-8198-de0a4f0eed1d.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐

# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd ask
mvn spring-boot:run

cd pay
mvn spring-boot:run 

cd book
mvn spring-boot:run  

cd mypage
mvn spring-boot:run  
```
게이트웨이 내부에서 spring, docker 환경에 따른 각 서비스 uri를 설정해주고 있다.

```
spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: ask
          uri: http://localhost:8081
          predicates:
            - Path=/asks/** 
        - id: pay
          uri: http://localhost:8082
          predicates:
            - Path=/pays/** 
        - id: book
          uri: http://localhost:8083
          predicates:
            - Path=/books/** 
        - id: mypage
          uri: http://localhost:8084
          predicates:
            - Path= /mypages/**
.....생략

---

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: ask
          uri: http://ask:8080
          predicates:
            - Path=/asks/** 
        - id: pay
          uri: http://pay:8080
          predicates:
            - Path=/pays/** 
        - id: book
          uri: http://book:8080
          predicates:
            - Path=/books/** 
        - id: mypage
          uri: http://mypage:8080
          predicates:
            - Path= /mypages/**
...
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다. 

```
package bookrental;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Ask_table")
public class Ask {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String status;
    private Long bookId;
    
    .../... 중략  .../...

    public Long getId() {
        return id;
    }
    public void setId(Long id) {
        this.id = id;
    }
    public String getStatus() {
        return status;
    }
    public void setStatus(String status) {
        this.status = status;
    }
    public Long getBookId() {
        return bookId;
    }
    public void setBookId(Long bookId) {
        this.bookId = bookId;
    }
    public String getAskDate() {
        return askDate;
    }
    .../... 중략  .../...
}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package bookrental;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface AskRepository extends PagingAndSortingRepository<Ask, Long>{
}
```
- 적용 후 REST API 의 테스트
```
# book 서비스의 등록처리
http ask:8080/asks id=20 status=ASKED bookId=11
```
![image](https://user-images.githubusercontent.com/70302903/96830053-b7a97f80-1475-11eb-82a8-2f67b4eeb149.PNG)

```
# ask 서비스의 대여신청처리
http post  ask:8080/asks id=1 status="ASKED"
```
![image](https://user-images.githubusercontent.com/70302903/96830054-b7a97f80-1475-11eb-9619-479e26f12d3b.PNG)

```
# 도서 상태 조회
http get book:8080/books
```
![image](https://user-images.githubusercontent.com/70302903/96830046-b5dfbc00-1475-11eb-955d-e938311388a0.PNG)
![image](https://user-images.githubusercontent.com/70302903/96830048-b6785280-1475-11eb-8c4c-062c628e27d9.PNG)

```
# 도서 대여상태 조회(CQRS)
http get mypage:8080/mypages
```
![image](https://user-images.githubusercontent.com/70302903/96830050-b710e900-1475-11eb-9daf-cba29c667631.PNG)
![image](https://user-images.githubusercontent.com/70302903/96830052-b710e900-1475-11eb-8760-2a616c15b06a.PNG)

## 폴리글랏 퍼시스턴스

각 마이크로서비스는 별도의 H2 DB를 가지고 있으며 CQRS를 위한 Mypage에서는 H2가 아닌 HSQLDB를 적용하였다.

```
# Mypage의 pom.xml에 dependency 추가
<!-- 
		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
 -->
		<dependency>
		    <groupId>org.hsqldb</groupId>
		    <artifactId>hsqldb</artifactId>
		    <version>2.4.0</version>
		    <scope>runtime</scope>
		</dependency>


```

## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 예약(book)->결제(payment) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 결제서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# (ask) PayService.java

package bookrental.external;

@FeignClient(name="pay", url="http://pay:8080")
public interface PayService {

    @RequestMapping(method= RequestMethod.POST, path="/pays")
    public void pay(@RequestBody Pay pay);

    @RequestMapping(method= RequestMethod.POST, path="/pays/{askId}")
    public void payCancel(@RequestBody Pay pay, @PathVariable("bookId") Long bookId);

}
```

- 주문을 받은 직후(@PostPersist) 결제를 요청하도록 처리
```
# Ask.java (Entity)

    @PostPersist 
    public void onPostPersist(){

        if (this.getStatus().equals("ASKED")) {
            Asked asked = new Asked();
            BeanUtils.copyProperties(this, asked);
            asked.publishAfterCommit();

            bookrental.external.Pay pay = new bookrental.external.Pay();
            // mappings goes here
            pay.setaskId(this.getId());
            pay.getBookId(this.getBookId());
            pay.setStatus(this.getStatus());
            AskApplication.applicationContext.getBean(bookrental.external.PayService.class)
                    .pay(pay);
        }
    }    
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:

```
# 결제 (pay) 서비스를 잠시 내려놓음 (ctrl+c)

#신청처리
http ask:8080/asks #Fail

#결제서비스 재기동
cd pay
mvn spring-boot:run

#신청처리
http ask:8080/asks #Fail   #Success
```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

결제가 이루어진 후에 도서에 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 도서 시스템의 처리를 위하여 결제주문이 블로킹 되지 않아도록 처리한다.
 
- 이를 위하여 결제이력에 기록을 남긴 후에 곧바로 승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
    @PostPersist
    public void onPostPersist(){

        ApprovalObtained approvalObtained = new ApprovalObtained();
        BeanUtils.copyProperties(this, approvalObtained);
        approvalObtained.publishAfterCommit();

    }
```
- 도서 서비스에서는 결제승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaid_Rent(@Payload Paid paid){

        if(paid.isMe()){
            System.out.println("##### listener Rent : " + paid.toJson());

            Optional<Book> optional = bookRepository.findById(paid.getBookId());
            Book book = optional.get();
            book.setAskId(paid.getAskId());
            book.setStatus("RENTED");

            bookRepository.save(book);
        }
    }

```

도서 시스템은 신청/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 신청/결제 시스템이 유지보수로 인해 잠시 내려간 상태라도 숙소를 등록하는데 문제가 없다:
```
# 결제 (pay) 서비스를 잠시 내려놓음 (ctrl+c)

#신청처리
http ask:8080/asks #Fail

#결제서비스 재기동
cd pay
mvn spring-boot:run

#신청처리
http ask:8080/asks #Fail   #Success
```


# 운영

## CI/CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS CodeBuild를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다.

## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 : istio destination 룰 적용하여 구현한다.

시나리오는 대여신청(ask)-->결제(pay) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- 피호출 서비스(결제:pay) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# (pay) Pay.java (Entity)

    @PrePersist
    public void onPrePersist(){  //결제이력을 저장한 후 적당한 시간 끌기

        ...
        
        try {
            Thread.currentThread().sleep((long) (400 + Math.random() * 220));

        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```
서킷브레이킹 미적용 시 100%임을 확인:
![image]()

데스티네이션 룰 적용c
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-ask
  namespace: bookrental
spec:
  host: ask
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1024           # 목적지로 가는 HTTP, TCP connection 최대 값. (Default 1024)
      http:
        http1MaxPendingRequests: 1  # 연결을 기다리는 request 수를 1개로 제한 (Default 
        maxRequestsPerConnection: 1 # keep alive 기능 disable
        maxRetries: 3               # 기다리는 동안 최대 재시도 수(Default 1024)
    outlierDetection:
      consecutiveErrors: 8          # 5xx 에러가 5번 발생하면
      interval: 10s                  # 10초마다 스캔 하여
      baseEjectionTime: 30s         # 30 초 동안 circuit breaking 처리   
      maxEjectionPercent: 10       # 100% 로 차단
EOF
```
부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
동시사용자 20명
20초 동안 실시

![image]()

운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌.

키알리 확인
![image]()
예거 확인
![image]()


### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다.

- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy pay --min=1 --max=10 --cpu-percent=15
```
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy pay -w
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
![image]()

- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
![image]()

## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 Readiness Probe 미설정 시 무정지 재배포 가능여부 확인을 위해 buildspec.yml의 Readiness Probe 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c20 -t120S -v  --content-type "application/json" 'http://ask:8080/asks POST {"id":"1","status":"ASKED" }'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
:
```
![image]()

- 새버전으로의 배포 시작
```
kubectl set image ...
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
![image]()
 
배포기간중 Availability 가 평소 100%에서 70% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

```
# deployment.yaml 의 readiness probe 의 설정:

kubectl apply -f kubernetes/deployment.yaml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
![image]()

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.
