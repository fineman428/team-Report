![OA](https://user-images.githubusercontent.com/70308042/96427791-f226d880-1239-11eb-963b-b7418a6f6005.jpg)

# Table of contents

- [OA사무용품대여](#---)
  - [서비스 시나리오](#서비스-시나리오)
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
1. 상품팀이 상품정보를 등록한다.
1. 상품 등록 정보가 렌탈 상품으로 저장된다.
1. 고객이 상품 렌탈 주문을 한다.
1. 주문이 되면 주문 내역이 배송팀에게 전달된다.
1. 배송팀은 주문정보와 재고수량을 체크해서 배송가능여부를 확인한다.
1. 재고부족으로 배송이 불가능할 경우 주문을 취소한다.
1. 고객이 렌탈 주문을 취소할 수 있다.
1. 주문이 취소되면 배송이 취소된다.
1. 고객이 주문상태를 중간에 조회 가능하다.

비기능적 요구사항
1. 트랜잭션
    1. 배송 취소가 되지 않은 주문건은 주문 취소가 성립되지 않아야 한다.  Sync 호출
1. 장애격리
    1. 상점등록 기능이 수행되지 않더라도 주문은 365일 24시간 받을 수 있어야 한다.  Async (event-driven), Eventual Consistency
    1. 배송취소 요청이 과중되면 사용자에게 주문취소를 잠시후에 하도록 유도한다.  Circuit breaker, fallback
1. 성능
    1. 고객이 렌탈주문 상태를 mypage에서 확인할 수 있어야 한다.  CQRS
    1. 상품등록, 주문, 배송 상태가 바뀔때마다 상품재고가 변경될 수 있어야 한다.  Event driven

# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과: http://www.msaez.io/#/storming/JYXQsarUP6V2T0Rc1FxxiYgyYra2/share/339d8f19217372662969dcd84f74bb39/-MK-Q4zIxxbiRaacIIw3


### 이벤트 도출

![event](https://user-images.githubusercontent.com/70308042/96430939-cc033780-123d-11eb-84a5-77b9de02a9cc.JPG)

### 어그리게잇으로 묶기
![agg](https://user-images.githubusercontent.com/70308042/96430946-ce659180-123d-11eb-9813-0f3b42444fa5.JPG)

    - Product의 상품등록, Rental의 주문처리, Delivery의 배송과 연결된 command / event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 묶어줌

### 바운디드 컨텍스트로 묶기

![bound](https://user-images.githubusercontent.com/70308042/96430954-d02f5500-123d-11eb-9ca1-6df197ed812b.JPG)

    - 도메인 서열 분리 
        - Core Domain: Rental은 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 app 의 경우 1주일 1회 미만, store 의 경우 1개월 1회 미만
        - Supporting Domain: Product는 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain: Delivery는 서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (핑크색으로 이후 전환할 예정)

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![poli](https://user-images.githubusercontent.com/70308042/96430959-d1f91880-123d-11eb-9051-e51d7bf1e66b.JPG)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![map](https://user-images.githubusercontent.com/70308042/96542669-8e55eb80-12dd-11eb-83dc-aa2dff4276dd.JPG)

### 완성된 1차 모형

![fin](https://user-images.githubusercontent.com/70308042/96542671-8e55eb80-12dd-11eb-9c5f-72f311bcbd20.JPG)

### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![t1](https://user-images.githubusercontent.com/70308042/96542672-8eee8200-12dd-11eb-97ff-f5fa582d0385.JPG)

    - 상품팀이 상품정보를 등록한다. (ok)
    - 상품 등록 정보가 렌탈 상품으로 저장된다. (ok)
    - 저장된 렌탈 상품 정보가 Delivery 상품 aggregate에 저장된다. (ok)

![t2](https://user-images.githubusercontent.com/70308042/96542673-8f871880-12dd-11eb-9648-6e5b73d5df4f.JPG)

    - 고객이 상품 렌탈 주문을 한다. (ok)
    - 주문이 되면 주문 내역이 배송팀에게 전달된다. (ok)
    - 배송팀은 주문정보와 재고수량을 체크해서 배송가능여부를 확인한다. (ok) 
    - 재고부족으로 배송이 불가능할 경우 주문을 취소한다. (ok)

![t3](https://user-images.githubusercontent.com/70308042/96542675-901faf00-12dd-11eb-9268-47aeafad8755.JPG)

    - 고객이 렌탈 주문을 취소할 수 있다. (ok)
    - 주문이 취소되면 배송이 취소된다. (ok)
    - 고객이 주문상태를 중간에 조회 가능하다. (?)

### 모델 수정

![fin2](https://user-images.githubusercontent.com/70308042/96542666-8c8c2800-12dd-11eb-8638-485d242a3394.JPG)
    
    - view 추가
    - 고객이 주문상태를 중간에 조회 가능하다. (ok)


## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://github.com/leety007/example-food-delivery/blob/master/%ED%97%A5%EC%82%AC.png)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 자바으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd gateway
mvn spring-boot:run

cd Product
mvn spring-boot:run

cd Rental
mvn spring-boot:run  

cd Delivery
mvn spring-boot:run  

cd Information
mvn spring-boot:run  
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다.

```
package rentalService;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;

@Entity
@Table(name="Rental_table")
public class Rental {

    @Id
    @GeneratedValue(strategy=GenerationType.IDENTITY)
    private Long id;
    private Long productId;
    private int qty;
    private String status;
    private String productName;
    private Long deliveryId;

  public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getProductId() {
        return productId;
    }

    public void setProductId(Long productId) {
        this.productId = productId;
    }

    public int getQty() {
        return qty;
    }

    public void setQty(int qty) {
        this.qty = qty;
    }

    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }

    public String getProductName() {
        return productName;
    }

    public void setProductName(String productName) {
        this.productName = productName;
    }

    public Long getDeliveryId() {
        return deliveryId;
    }

    public void setDeliveryId(Long deliveryId) {
        this.deliveryId = deliveryId;
    }



```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package rentalService;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface RentalRepository extends PagingAndSortingRepository<Rental, Long>{

}
```
- 적용 후 REST API 의 테스트

#### 적용 후 REST API 의 테스트   

##### Step 1-1 상품정보 등록   
컴퓨터 등록 : http POST http://localhost:8081/products name=computer qty=9   
![1-1_상품등록_컴퓨커](https://user-images.githubusercontent.com/70302880/96540543-ff46d480-12d8-11eb-9bcc-33128a10f6f6.PNG)
   
모니터등록 : http POST http://localhost:8081/products name=monitor qty=9   
![1-1_상품등록_모니터](https://user-images.githubusercontent.com/70302880/96540538-fe15a780-12d8-11eb-967f-f25d11259abe.PNG)  
   
##### Step 1-2 상품등록 후 결과 조회   
product : http GET http://localhost:8081/products/1   
![1-2_결과조회_product](https://user-images.githubusercontent.com/70302880/96540548-00780180-12d9-11eb-8a79-35bac4fca8c4.PNG)   
   
rental : http GET http://localhost:8082/products/1   
![1-2_결과조회_rental](https://user-images.githubusercontent.com/70302880/96540551-00780180-12d9-11eb-8924-583e2471c338.PNG)   
   
delivery : http GET http://localhost:8083/products/1   
![1-2_결과조회_delivery](https://user-images.githubusercontent.com/70302880/96540545-ffdf6b00-12d8-11eb-91d8-ddff7abffe51.PNG)  
   
##### Step 2-1 렌탈 요청   
재고 수량 이상 렌탈 요청 : http POST http://localhost:8082/rentals productId=1 qty=12 status=ORDERED productName=computer   
![2-1_렌탈요청_재고수량이상](https://user-images.githubusercontent.com/70302880/96541735-96ad2700-12db-11eb-8256-8b7679516a85.PNG)   
   
미등록 상품 렌탈요청 : http POST http://localhost:8082/rentals productId=22 qty=12 status=ORDERED productName=computer   
![2-1_렌탈요청_미등록상품](https://user-images.githubusercontent.com/70302880/96540552-01109800-12d9-11eb-817f-8b8b3717389c.PNG)   

정상 렌탈 요청 : http POST http://localhost:8082/rentals productId=1 qty=3 status=ORDERED productName=computer
![2-1_렌탈요청_정상](https://user-images.githubusercontent.com/70302880/96540557-0241c500-12d9-11eb-8902-13b64d3f64e4.PNG)


##### Step 2-2 렌탈 요청 후 결과 조회
rental 렌탈 주문 정보(재고 수량 이상) : http GET http://localhost:8082/rentals/1
![2-2_결과조회_렌탈주문정보(재고수량이상)](https://user-images.githubusercontent.com/70302880/96542803-da089500-12dd-11eb-9aec-d3123e2de01f.PNG)

 rental 렌탈 주문 정보(미등록 상품) : http GET http://localhost:8082/rentals/2
 ![2-2_결과조회_렌탈주문정보(미등록상품)](https://user-images.githubusercontent.com/70302880/96540559-02da5b80-12d9-11eb-8251-73f33fa307cc.PNG)
 
 rental 렌탈 주문 정보(정상) : http GET http://localhost:8082/rentals/3
 ![2-2_결과조회_렌탈주문정보(정상)](https://user-images.githubusercontent.com/70302880/96540564-040b8880-12d9-11eb-8ebd-0a646113b4b1.PNG)
 
 delivery 배송정보 : http GET http://localhost:8083/deliveries/3
 ![2-2_결과조회_배송정보](https://user-images.githubusercontent.com/70302880/96540566-04a41f00-12d9-11eb-877e-9f1478e15b3f.PNG)
 
 delivery 재고차감 확인 : http GET http://localhost:8083/products/1  
  .최초 제품 등록시 재고 9개, 3개 배송 되어 최종 수량 6개 변경
 ![2-2_결과조회_배송_재고차감확인](https://user-images.githubusercontent.com/70302880/96540565-040b8880-12d9-11eb-8a37-17ccc7c4e5c8.PNG)
 
 information 내 주문 View : http GET http://localhost:8084/myOrders/3
 ![2-2_결과조회_내주문VIEW](https://user-images.githubusercontent.com/70302880/96540558-02da5b80-12d9-11eb-8850-b2ec04648aba.PNG)
 
 ##### 3-1 렌탈 취소 요청
렌탈취소 :  http DELETE http://localhost:8082/rentals/3
![3-1_렌탈취소요청](https://user-images.githubusercontent.com/70302880/96540570-04a41f00-12d9-11eb-957f-cb58933be879.PNG)

 ##### 3-2 렌탈 취소 후 결과조회
rental :  http GET http://localhost:8082/rentals/3
![3-2_결과조회_rental](https://user-images.githubusercontent.com/70302880/96540574-05d54c00-12d9-11eb-96a6-7cda28638b83.PNG)

delivery :  http GET http://localhost:8083/deliveries/3
![3-2_결과조회_delivery](https://user-images.githubusercontent.com/70302880/96540572-053cb580-12d9-11eb-962b-9cf71791389e.PNG)

information :  http GET http://localhost:8084/myOrders/3
![3-2_결과조회_information](https://user-images.githubusercontent.com/70302880/96540573-05d54c00-12d9-11eb-9b2a-ce15b90a01e3.PNG)

rental :  http GET http://localhost:8082/rentals/3
![3-2_결과조회_rental](https://user-images.githubusercontent.com/70302880/96540574-05d54c00-12d9-11eb-96a6-7cda28638b83.PNG)



## 폴리글랏 퍼시스턴스

팀프로젝트 진행 시 간편한 DB구성을 위해 RDBMS 기반의 H2 DB를 적용하였다. H2는 Docker에서 설정이 가능하기 때문에 application.yml 파일에는 설정하지 않았으며 dependencies에만 추가하여 진행하였다. 
@Table를 사용하여 따로 테이블명을 지정하였으며 Entity Pattern과 Repository Pattern을 적용하였다.

```
#Product.java

package rentalService;

@Data
@Entity
@Table(name="Product_table")
public class Product {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String name;

    @ColumnDefault("10") //default 10
    private int qty ;
}

#ProductRepository.java

package rentalService;

import org.springframework.data.repository.PagingAndSortingRepository;
public interface ProductRepository extends PagingAndSortingRepository<Product, Long>{
}

pom.xml
<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>

```

## 폴리글랏 프로그래밍

물품 대여 시스템의 시나리오인 대여, 배송 등의 시스템 구현 방식은 JPA를 기반으로 구현하였으며 주요 이벤트 처리방식은 Kafka, FeignClient를 적용하였다.
```
#Kafka 적용

kafkaProcessor.java
package rentalService.config.kafka;

import org.springframework.cloud.stream.annotation.Input;
import org.springframework.cloud.stream.annotation.Output;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.SubscribableChannel;

public interface KafkaProcessor {

    String INPUT = "event-in";
    String OUTPUT = "event-out";

    @Input(INPUT)
    SubscribableChannel inboundTopic();

    @Output(OUTPUT)
    MessageChannel outboundTopic();

}

Rental.java
@PostPersist
    public void onPostPersist(){

        Rentaled rentaled = new Rentaled();
        BeanUtils.copyProperties(this, rentaled);
        rentaled.publishAfterCommit();

    }

#FeingClient 적용

DeliveryService.java
package rentalService.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="Delivery", url="${api.delivery.url}")
//@FeignClient(name="Delivery", url="http://localhost:8083")
public interface DeliveryService {

    @RequestMapping(method= RequestMethod.POST, path="/deliveries")
    public void deliveryCancel(@RequestBody Delivery delivery);

}

```


## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문취소->배송취소 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- FeignClient 방식을 통해서 Request-Response 처리.

```
# RentalApplication.java

package rentalService;
import rentalService.config.kafka.KafkaProcessor;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.openfeign.EnableFeignClients;


@SpringBootApplication
@EnableBinding(KafkaProcessor.class)
@EnableFeignClients
public class RentalApplication {
    protected static ApplicationContext applicationContext;
    public static void main(String[] args) {
        applicationContext = SpringApplication.run(RentalApplication.class, args);
    }
}
```

- 주문취소로 주문삭제 하기전에 (@PreRemove) 배송취소를 요청하도록 처리

```
# Rental.java (Entity)

    @PreRemove
    public void onPreRemove(){
        // RentalCanceled
        RentalCanceled rentalCanceled = new RentalCanceled();
        BeanUtils.copyProperties(this, rentalCanceled);
        rentalCanceled.setStatus("CANCELED");
        rentalCanceled.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        // 동기호출
        rentalService.external.Delivery delivery = new rentalService.external.Delivery();
        // mappings goes here
        BeanUtils.copyProperties(this, delivery);
        delivery.setId(deliveryId);
        delivery.setRentalId(this.id);
        delivery.setStatus("CANCELED");
        RentalApplication.applicationContext.getBean(rentalService.external.DeliveryService.class)
                .deliveryCancel(delivery);

    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, Delivery 시스템이 장애가 나면 주문취소를 못받는다는 것을 확인:


```
# Delivery 서비스를 잠시 내려놓음 : 주문취소 처리 시 500 Error 발생
# Delivery 서비스를 다시 올림 : 주문취소 정상처리
```
![del1](https://user-images.githubusercontent.com/70308042/96539442-03bdbe00-12d6-11eb-8d24-7aa9b49bc985.JPG)


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

Rental이 이루어진 후에(Rentaled)  Delivery 시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리.   
Delivery 시스템의 처리를 위하여 Rental이 블로킹 되지 않도록 처리.   
이를 위하여 Rental 이력에 기록을 남긴 후에 곧바로 Rental이 처리 되었다는 도메인 이벤트를 카프카로 송출한다(Publish).   
```
package rentalService;
...
@Data
@Entity
@Table(name="Rental_table")
public class Rental {

...
   
   @PostPersist
   public void onPostPersist(){
       Rentaled rentaled = new Rentaled();
       BeanUtils.copyProperties(this, rentaled);
       rentaled.publishAfterCommit();
   }
...
}
```
- Delivery  서비스에서는 Rental 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현하였고, 실제 구현 된 시스템에서는 Rental 이벤트를 수신한 이후에  Rental 요청에 대한 적합성 여부(등록상품여부, 재고가용여부) 를 판단한 이후에 Delivery  처리하도록 구현   
```
package rentalService;
...
@Service
public class PolicyHandler{
   ...
   @StreamListener(KafkaProcessor.INPUT)
   public void wheneverRentaled_Delivery(@Payload Rentaled rentaled){
       if(rentaled.isMe()){
           // 배송 등록
           Delivery delivery = new Delivery();
           delivery.setProductId(rentaled.getProductId());
           delivery.setRentalId(rentaled.getId());
           delivery.setStatus("DELIVERED");
           delivery.setQty(rentaled.getQty());
           // 재고 확인
           Optional<Product> productOptional = ProductRepository.findByProductId(rentaled.getProductId());
           Product product = null;
           try {
               product = productOptional.get();
           } catch (Exception e) {
               // 상품정보 확인 불가
               System.out.println("rentaled 수신 : 상품정보 확인불가");
               delivery.setStatus("CANCELED_UnregisteredProduct");
               DeliveryRepository.save(delivery);
               return;
           }
           if ( product.getQty() < rentaled.getQty() ){
               // 재고 부족 -> 보상 트랜젝션 (saga pattern)
               System.out.println("재고 수량 비교 : qty="+product.getQty()+" / rentaled.getQty()="+rentaled.getQty());
               System.out.println("rentaled 수신 : 재고 부족 -> 보상 트랜젝션 (saga pattern))");
               delivery.setStatus("CANCELED_OutOfStock");
           } else {
               // 정상 - 재고 차감
               System.out.println("rentaled 수신 : 정상 - 재고 차감");
               product.setQty(  product.getQty()  -  rentaled.getQty() );
               ProductRepository.save(product);
           }
           // 배송정보 저장
           DeliveryRepository.save(delivery);
          
       }
   } 
   ...
}
```

- Delivery 시스템은 Rental 시스템과 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, Delivery 시스템이 유지보수로 인해 잠시 내려간 상태 라도 Rental 요청을 받는데 문제가 없다.
```
#Rental 처리
http POST http://localhost:8082/rentals productId=1 qty=3 status=ORDERED productName=computer

#Rental  확인
http GET http://localhost:8082/rentals/1  #제대로 Data 들어옴   

#Delivery 서비스중지  확인
http GET http://localhost:8083/deliveries/1
```
![8-1 Delivery서비스 중지후 Rental 처리](https://user-images.githubusercontent.com/70302880/96542122-631ecc80-12dc-11eb-9425-5154e8509b18.PNG)

```
Delivery 서비스 기동
cd Delivery
mvn spring-boot:run

#Delivery 상태 확인
http GET http://localhost:8083/deliveries/1    # 제대로 kafka로 부터 data 수신 함을 확인

#이전에 요청한 Rental 상태 확인
http GET http://localhost:8082/rentals/1 # 제대로 kafka로 부터 data 수신 함을 확인
```
(status가 “CANCELED_UnregisteredProduct”으로 처리된 사유는 Rental 적합성 체크시 Delivery 의 Product Reposity에 등록된 Product를 확인하는데, Delivery 서비스를 재기동하면서 Delivery의  Product Repository(h2database)가 초기화되면서 발생)
![8-2 Delivery서비스 기동후 Delivery및 Rental 처리확인](https://user-images.githubusercontent.com/70302880/96542118-61ed9f80-12dc-11eb-9cee-f4442ebcb617.PNG)



# 운영

## CI/CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 GCP를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 cloudbuild.yml 에 포함되었다.


## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 단말앱(app)-->결제(pay) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
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

- 피호출 서비스(결제:pay) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# (pay) 결제이력.java (Entity)

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

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
$ siege -c100 -t60S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.73 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.75 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.77 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.97 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.81 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.87 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.12 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.16 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.17 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.26 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.25 secs:     207 bytes ==> POST http://localhost:8081/orders

* 요청이 과도하여 CB를 동작함 요청을 차단

HTTP/1.1 500     1.29 secs:     248 bytes ==> POST http://localhost:8081/orders   
HTTP/1.1 500     1.24 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.23 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.42 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     2.08 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.29 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.24 secs:     248 bytes ==> POST http://localhost:8081/orders

* 요청을 어느정도 돌려보내고나니, 기존에 밀린 일들이 처리되었고, 회로를 닫아 요청을 다시 받기 시작

HTTP/1.1 201     1.46 secs:     207 bytes ==> POST http://localhost:8081/orders  
HTTP/1.1 201     1.33 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.36 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.63 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.65 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.69 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.71 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.71 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.74 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.76 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     1.79 secs:     207 bytes ==> POST http://localhost:8081/orders

* 다시 요청이 쌓이기 시작하여 건당 처리시간이 610 밀리를 살짝 넘기기 시작 => 회로 열기 => 요청 실패처리

HTTP/1.1 500     1.93 secs:     248 bytes ==> POST http://localhost:8081/orders    
HTTP/1.1 500     1.92 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     1.93 secs:     248 bytes ==> POST http://localhost:8081/orders

* 생각보다 빨리 상태 호전됨 - (건당 (쓰레드당) 처리시간이 610 밀리 미만으로 회복) => 요청 수락

HTTP/1.1 201     2.24 secs:     207 bytes ==> POST http://localhost:8081/orders  
HTTP/1.1 201     2.32 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.16 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.19 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.21 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.29 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.30 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.38 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.59 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.61 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.62 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     2.64 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.01 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.27 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.33 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.45 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.52 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.57 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/orders

* 이후 이러한 패턴이 계속 반복되면서 시스템은 도미노 현상이나 자원 소모의 폭주 없이 잘 운영됨


HTTP/1.1 500     4.76 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.23 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.76 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.74 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.82 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.82 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.84 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.66 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     5.03 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.22 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.19 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.18 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.69 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.65 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     5.13 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.84 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.25 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.25 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.80 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.87 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.33 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.86 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.96 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.34 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 500     4.04 secs:     248 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.50 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.95 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.54 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     4.65 secs:     207 bytes ==> POST http://localhost:8081/orders


:
:

Transactions:		        1025 hits
Availability:		       63.55 %
Elapsed time:		       59.78 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02
Successful transactions:        1025
Failed transactions:	         588
Longest transaction:	        9.20
Shortest transaction:	        0.00

```
- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 63.55% 가 성공하였고, 46%가 실패했다는 것은 고객 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.

- Retry 의 설정 (istio)
- Availability 가 높아진 것을 확인 (siege)

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
```
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
pay     1         1         1            1           17s
pay     1         2         1            1           45s
pay     1         4         1            1           1m
:
```
- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
```
Transactions:		        5078 hits
Availability:		       92.45 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02
```


## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'

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


# 신규 개발 조직의 추가

  ![image](https://user-images.githubusercontent.com/487999/79684133-1d6c4300-826a-11ea-94a2-602e61814ebf.png)


## 마케팅팀의 추가
    - KPI: 신규 고객의 유입률 증대와 기존 고객의 충성도 향상
    - 구현계획 마이크로 서비스: 기존 customer 마이크로 서비스를 인수하며, 고객에 음식 및 맛집 추천 서비스 등을 제공할 예정

## 이벤트 스토밍 
    ![image](https://user-images.githubusercontent.com/487999/79685356-2b729180-8273-11ea-9361-a434065f2249.png)


## 헥사고날 아키텍처 변화 

![image](https://user-images.githubusercontent.com/487999/79685243-1d704100-8272-11ea-8ef6-f4869c509996.png)

## 구현  

기존의 마이크로 서비스에 수정을 발생시키지 않도록 Inbund 요청을 REST 가 아닌 Event 를 Subscribe 하는 방식으로 구현. 기존 마이크로 서비스에 대하여 아키텍처나 기존 마이크로 서비스들의 데이터베이스 구조와 관계없이 추가됨. 

## 운영과 Retirement

Request/Response 방식으로 구현하지 않았기 때문에 서비스가 더이상 불필요해져도 Deployment 에서 제거되면 기존 마이크로 서비스에 어떤 영향도 주지 않음.

* [비교] 결제 (pay) 마이크로서비스의 경우 API 변화나 Retire 시에 app(주문) 마이크로 서비스의 변경을 초래함:

예) API 변화시
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){

        fooddelivery.external.결제이력 pay = new fooddelivery.external.결제이력();
        pay.setOrderId(getOrderId());
        
        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제(pay);

                --> 

        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제2(pay);

    }
```

예) Retire 시
```
# Order.java (Entity)

    @PostPersist
    public void onPostPersist(){

        /**
        fooddelivery.external.결제이력 pay = new fooddelivery.external.결제이력();
        pay.setOrderId(getOrderId());
        
        Application.applicationContext.getBean(fooddelivery.external.결제이력Service.class)
                .결제(pay);

        **/
    }
```
