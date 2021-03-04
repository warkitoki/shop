# coffeeshop


# 서비스 시나리오

1. 손님이 주문을하고 결재를 실행한다.
2. 결재가 완료되면 접수처리를 하고 제작을 한다.
3. 제작이 완료되면 전달을 한다.
4. 주문이 잘 못 되었을 경우 취소한다
5. 손님이 주문 금액및 결재상태를 확인할 수 있다.
6. 손님과 주인은 모든 진행이력을 볼 수 있다.


비기능적 요구사항
1. 트랜잭션
  - 결재가 되지 않으면 접수가 되지 않도록 한다. (Sync 호출) 
2. 장애격리
  - 주문이 밀려 만들수 없더라도 주문 접수는 계속 받을 수 있다. (Async 호출)
  - 결재 시스템에 문제가 있다면 접수를 잠시 보류한다. (서킷브레이킹)
3. 성능
  - 주문현황에 대해 별도록 확인할 수 있어야 한다. (CQRS)


# 체크포인트
https://workflowy.com/s/assessment/qJn45fBdVZn4atl3


# 모형
![모형2](https://user-images.githubusercontent.com/78134049/110017380-7f8e4380-7d69-11eb-9318-fad32e05bb6c.png)

# 기능적/비기능적 요구사항에 대한 검증
1. 손님이 주문을 한다. (1)
2. 결재처리(2)
   - 결재가 완료되면 접수처리하고 제작을 한다. (3 -> 4)
   - 결재가 완료되지 않으면 진행하지 않는다. (X)
3. 제작이 완료되면 전달한다. (5 -> 6)
4. 모든 주문 및 준비현황은 dashboard에서 볼 수 있다. (7)

# 헥사고날 아키텍쳐 다이어그램 도출 (Polyglot)
![헥사고날](https://user-images.githubusercontent.com/78134049/110017462-93d24080-7d69-11eb-9c3c-f374bb28613d.png)

# 구현
각각의 서비스 실행 명령어는 아래와 같다. 포트넘버는 8081 ~ 8084, 8088이다.

```linux
cd order
mvn spring-boot:run

cd deposit
mvn spring-boot:run

cd product
mvn spring-boot:run

cd dashboard
mvn spring-boot:run

cd gateway
mvn spring-boot:run

```

# DDD의 적용
order 서비스의 order.java
```java
package coffeeshop;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String productId;
    private Integer qty;
    private String status;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public String getProductId() {
        return productId;
    }

    public void setProductId(String productId) {
        this.productId = productId;
    }
    public Integer getQty() {
        return qty;
    }

    public void setQty(Integer qty) {
        this.qty = qty;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }

    @PostPersist
    public void onPostPersist(){
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        coffeeshop.external.Deposit deposit = new coffeeshop.external.Deposit();
        deposit.setOrderId(this.getId());
        deposit.setProductId(this.getProductId());
        deposit.setQty(this.getQty());
        deposit.setStatus("PayCompleted");
        
        // mappings goes here
        OrderApplication.applicationContext.getBean(coffeeshop.external.DepositService.class)
            .pay(deposit);


    }

    @PrePersist
    public void onPrePersist(){
        /*OrderCanceled orderCanceled = new OrderCanceled();
        BeanUtils.copyProperties(this, orderCanceled);
        orderCanceled.publishAfterCommit();
        */
        try {
            Thread.currentThread().sleep((long) (1000 + Math.random() * 220));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }

}
```
order 서비스의 PolicyHandler.java
```java
package coffeeshop;

import coffeeshop.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

import java.util.Optional;

@Service
public class PolicyHandler{
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @Autowired
    OrderRepository orderRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverDeliveryStarted_(@Payload DeliveryStarted deliveryStarted){

        if(deliveryStarted.isMe()){
            System.out.println("##### listener  : " + deliveryStarted.toJson());

            Optional<Order> orderOptional = orderRepository.findById(deliveryStarted.getOrderId());
            Order order = orderOptional.get();
            order.setStatus(deliveryStarted.getStatus());
            order.setStatus("DeliveryStarted");
            orderRepository.save(order);
        }
    }

}
```

- 적용 후 REST API의 테스트를 통해 정상적으로 작동함을 알 수 있었다.
  - order 주문 
 ```
 http http://52.231.66.165:8080/orders productId=10 qty=5
 ```
![주문_1](https://user-images.githubusercontent.com/78134049/110017553-afd5e200-7d69-11eb-9f65-22ba3e08878b.png)
 
 - 주문 현황
 ```
 http http://52.231.66.165:8080/orders/1
 ```
![주문2](https://user-images.githubusercontent.com/78134049/110017583-b6645980-7d69-11eb-898a-08ba20fb137e.png)
 
 ```
 http http://52.231.66.165:8080/products/1
 ```
![주문3](https://user-images.githubusercontent.com/78134049/110017630-c419df00-7d69-11eb-8d03-41358adaa207.png)


# Gateway 적용
API Gateway를 통해 마이크로 서비스들의 진입점을 하나로 진행하였다.
```yml
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://localhost:8081
          predicates:
            - Path=/orders/** 
        - id: deposit
          uri: http://localhost:8082
          predicates:
            - Path=/deposits/** 
        - id: product
          uri: http://localhost:8083
          predicates:
            - Path=/products/** 
        - id: dashboard
          uri: http://localhost:8084
          predicates:
            - Path= /dashboards/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true


---

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://order:8080
          predicates:
            - Path=/orders/** 
        - id: deposit
          uri: http://deposit:8080
          predicates:
            - Path=/deposits/** 
        - id: product
          uri: http://product:8080
          predicates:
            - Path=/products/** 
        - id: dashboard
          uri: http://dashboard:8080
          predicates:
            - Path= /dashboards/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true

server:
  port: 8080
```


# Polyglot Persistence
 - dashboard 서비스의 경우, 다른 서비스들이 h2 저장소를 이용한 것과는 다르게 hsql을 이용하였다.
 - 이 작업을 통해 서비스들이 각각 다른 데이터베이스를 사용하더라도 전체적인 기능엔 문제가 없음을, 즉 Polyglot Persistence를 충족하였다.
```xml
		<!-- <dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency> -->
		<dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<version>2.4.1</version>
		</dependency>
```

# 동기식 호출(Req/Res 방식)과 Fallback 처리
 - order 서비스의 external/DepositService.java 내에 결제를 위한 Service 대행 인터페이스(Proxy)를 FeignClient를 이용하여 구현하였다.
```java
@FeignClient(name="deposit", url="api.deposit.url")
public interface DepositService {

    @RequestMapping(method= RequestMethod.GET, path="/deposits")
    public void pay(@RequestBody Deposit deposit);

}
```
 
 - order 서비스의 Order.java 내에서 결재처리 완료. (@PostPersist)
```java
    @PostPersist
    public void onPostPersist(){
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        coffeeshop.external.Deposit deposit = new coffeeshop.external.Deposit();
        deposit.setOrderId(this.getId());
        deposit.setProductId(this.getProductId());
        deposit.setQty(this.getQty());
        deposit.setStatus("PayCompleted");
        
        // mappings goes here
        OrderApplication.applicationContext.getBean(coffeeshop.external.DepositService.class)
            .pay(deposit);


    }
```
 
 - 동기식 호출에서는 호출 시간에 따른 커플링이 발생하여, deposit 시스템에 장애가 나면 주문을 할 수 없다.
![동기_장애_1](https://user-images.githubusercontent.com/78134049/110017928-178c2d00-7d6a-11eb-8a25-09a4201c5a77.png)
![동기_장애_2](https://user-images.githubusercontent.com/78134049/110017955-21ae2b80-7d6a-11eb-9dac-47960e8b236d.png)

# 비동기식 호출 (Pub/Sub 방식)
 - deposit 서비스 내 Deposit.java에서 아래와 같이 서비스 Pub 구현
```java
package coffeeshop;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Deposit_table")
public class Deposit {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long orderId;
    private String productId;
    private String status;
    private Integer qty;

    @PostPersist
    public void onPostPersist(){
        if(this.getStatus().equals("PayCompleted")){
            PayCompleted payCompleted = new PayCompleted();
            BeanUtils.copyProperties(this, payCompleted);
            payCompleted.publishAfterCommit();
        }else if(this.getStatus().equals("PayCanceled")){
            PayCanceled payCanceled = new PayCanceled();
            BeanUtils.copyProperties(this, payCanceled);
            payCanceled.publishAfterCommit();
        }
```
 
 - product 서비스 내 PolicyHandler.java 에서 아래와 같이 Sub 구현
```java
@Service
public class PolicyHandler{
    @Autowired
    ProductRepository productRepository;   

    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPayCompleted_(@Payload PayCompleted payCompleted){

        if(payCompleted.isMe()){
            System.out.println("##### listener  : " + payCompleted.toJson());

            Product product = new Product();
            product.setOrderId(payCompleted.getOrderId());
            product.setProductId(payCompleted.getProductId());
            product.setQty(payCompleted.getQty());
            product.setStatus("DeliveryStarted");

            productRepository.save(product);
        }
    }
```

- 비동기 호출은 다른 서비스 하나가 비정상이어도 해당 메세지를 다른 메시지 큐에서 보관하고 있기에, 서비스가 다시 정상으로 돌아오게 되면 그 메시지를 처리하게 된다.
  -> product 서비스를 내려도 order 주문에는 문제 없이 동작한다.
![비동기_장애_1](https://user-images.githubusercontent.com/78134049/110017974-270b7600-7d6a-11eb-837b-a6fb17c00790.png)
![비동기_장애_2](https://user-images.githubusercontent.com/78134049/110018001-2f63b100-7d6a-11eb-9db9-8ee618c63e41.png)

   
# CQRS
viewer인 dashboard 서비스를 별도로 구현하여 아래와 같이 view를 출력한다.
![주문4_CQRS](https://user-images.githubusercontent.com/78134049/110017695-d6941880-7d69-11eb-9b25-d9259f1e8162.png)

  
# 운영
# CI/CD 설정
  - git에서 소스 가져오기
  - Build 하기
  ```
  각 서비스 폴더에서 아래 명령어로 빌드 수행
  mvn package
  ```
  - Dockerlizing, ACR(Azure Container Registry에 Docker Image Push하기
  ```
  az acr build --registry skuser15 --image skuser15.azurecr.io/gateway:v2 .
  az acr build --registry skuser15 --image skuser15.azurecr.io/order:v1 .
  az acr build --registry skuser15 --image skuser15.azurecr.io/deposit:v2 .
  az acr build --registry skuser15 --image skuser15.azurecr.io/product:v2 .
  az acr build --registry skuser15 --image skuser15.azurecr.io/dashboard:v2 .
  ```
  - ACR에서 이미지 가져와서 Kubernetes에서 Deploy하기
  ```
  kubectl create deploy gateway --image=skuser15.azurecr.io/gateway:v2
  kubectl create deploy order --image=skuser15.azurecr.io/order:v1
  kubectl create deploy deposit --image=skuser15.azurecr.io/deposit:v2
  kubectl create deploy product --image=skuser15.azurecr.io/product:v2
  kubectl create deploy dashboard --image=skuser15.azurecr.io/dashboard:v2
  ```
  - Kubernetes에서 서비스 생성하기 (Docker 생성이기에 Port는 8080이며, Gateway는 LoadBalancer로 생성)
  ```
  kubectl expose deploy gateway --type="LoadBalancer" --port=8080
  kubectl expose deploy order --type="ClusterIP" --port=8080
  kubectl expose deploy deposit --type="ClusterIP" --port=8080
  kubectl expose deploy product --type="ClusterIP" --port=8080
  kubectl expose deploy dashboard --type="ClusterIP" --port=8080
  ```
  - Kubectl Expose 결과 확인
 
# 오토스케일 아웃

# ConfigMap 적용

# 동기식 호출 / 서킷 브레이킹 / 장애격리

# 무정지 재배포

# Self-healing (Liveness Probe)
