# 분석 설계

## Event Storming 결과

MSAez를 사용한 event stroming결과 아래와 같이 설계

![캡처](https://user-images.githubusercontent.com/27837607/100529978-e9112a00-322f-11eb-9310-0043d188375b.JPG)



## 헥사고날 아키텍쳐
![12](https://user-images.githubusercontent.com/27837607/100531177-42805580-323e-11eb-9b7f-8bf6c224fd40.JPG)


# 구현

## DDD 의 구현

MSAez로 구현한 Aggregate단위로 Entity를 선언 후, 구현 진행.

```
package shop;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import shop.external.Payment;

import java.util.List;

@Entity
@Table(name="Order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long productId;
    private Long qty;
    private String orderStatus;
    private Long orderId;

    @PostPersist
    public void onPostPersist()
    {
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.setOrderStatus("Order");
        ordered.setOrderId(this.getOrderId());
        ordered.publishAfterCommit();

        shop.external.Payment payment = new shop.external.Payment();
        System.out.println("결제 이벤트 발생");
        payment.setId(this.getOrderId());
        payment.setStatus("Paid");
        OrderApplication.applicationContext.getBean(shop.external.PaymentService.class)
                .pay(payment);

    }
    @PreUpdate
    public void onPrePersist(){


        OrderCanceled orderCanceled = new OrderCanceled();
        BeanUtils.copyProperties(this, orderCanceled);
        if(orderCanceled.getOrderStatus().equals("Cancel"))
        {
            orderCanceled.setOrderStatus(this.getOrderStatus());
            orderCanceled.publishAfterCommit();
        }
        else
        {
            PaymentRequested paymentRequested = new PaymentRequested();
            BeanUtils.copyProperties(this, paymentRequested);
            paymentRequested.publishAfterCommit();



            //orderCanceled.publishAfterCommit();

            //Following code causes dependency to external APIs
            // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

            //shop.external.Cancellation cancellation = new shop.external.Cancellation();
            // mappings goes here

            //OrderApplication.applicationContext.getBean(shop.external.CancellationService.class)
            //    .cancel(cancellation);
        }



    }


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
    public Long getQty() {
        return qty;
    }

    public void setQty(Long qty) {
        this.qty = qty;
    }

    public void setOrderStatus(String orderStatus){this.orderStatus = orderStatus;}
    public String getOrderStatus(){return orderStatus;}

    public Long getOrderId(){return orderId;}
    public void setOrderId(Long orderId){this.orderId = orderId;}
}

```
REST API에서의 테스트를 통하여 구현내용이 정상적으로 동작함을 확인.

![3](https://user-images.githubusercontent.com/27837607/100530571-0f869380-3237-11eb-9bd6-44a0778650d1.JPG)

![4](https://user-images.githubusercontent.com/27837607/100530623-8f146280-3237-11eb-8897-01198543797f.JPG)


## 동기식 호출(Request 방식의 아키텍쳐)

Order 내에 아래와 같은 FeignClient 선언

```
package shop.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="pay", url="${api.url.pay}")
//@FeignClient(name="pay", url="http://pay:8080")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.GET, path="/payments")
    public void pay(@RequestBody Payment payment);

}
```

@PoosePersist에서 이벤트 처리 수행

```
@PostPersist
    public void onPostPersist()
    {
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.setOrderStatus("Order");
        ordered.setOrderId(this.getOrderId());
        ordered.publishAfterCommit();

        shop.external.Payment payment = new shop.external.Payment();
        System.out.println("결제 이벤트 발생");
        payment.setId(this.getOrderId());
        payment.setStatus("Paid");
        OrderApplication.applicationContext.getBean(shop.external.PaymentService.class)
                .pay(payment);

    }
```
Pay서비스와 Order 서비스가 둘 다 돌아가고 있을 때에는 Order서비스에 아래와 같이 수행 하여도 이상 없음.

![5](https://user-images.githubusercontent.com/27837607/100530813-2da1c300-323a-11eb-876f-79c191c700fa.JPG)

Pay 서비스를 내린 후, Order 서비스만 돌아가고 있는 상태에서는 Order 서비스에 아래와 같이 수행시 이상 발생.

![6](https://user-images.githubusercontent.com/27837607/100530816-3397a400-323a-11eb-93c0-ea824f9709c7.JPG)


## 비동기식 호출(Pub/Sub 방식의 아키텍쳐)

Payment.java내에서 아래와 같이 서비스 Publish 구현

```
    @PostUpdate
    public void onPostUpdate()
    {
        Paid paid = new Paid();
        BeanUtils.copyProperties(this, paid);
        paid.publishAfterCommit();

    }
```

Delivery 서비스 내 Policy Handler에서 아래와 같이 Sub 구현

```
@StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaid_Ship(@Payload Paid paid){

        if(paid.isMe()){
            System.out.println("##### listener Ship : " + paid.toJson());
            Delivery delivery = new Delivery();
            delivery.setDeliveryStatus("ordered");
            System.out.println("Delivery start");

            deliveryRepository.save(delivery);
        }

    }
```

Pay 서비스와 Delivery 서비스가 둘 다 동시에 돌아가고 있을때 Pay 서비스 실행시 이상 없음.
![8](https://user-images.githubusercontent.com/27837607/100531031-00a2df80-323d-11eb-8efb-e4818c3d5b55.JPG)

Pay 서비스는 실행한 채, Delivery 서비스를 내린 후, Pay 서비스를 실행하여도 이상 없이 동작 가능.
![9](https://user-images.githubusercontent.com/27837607/100531033-01d40c80-323d-11eb-9edf-5ecb01fb9abb.JPG)


## Gateway

![7](https://user-images.githubusercontent.com/27837607/100531000-55922600-323c-11eb-9fae-61d5cdad63b3.JPG)

Gateway 서비스 실행 상태에서 8088과 8084로 각각 서비스 실행하여도 동일하게 Pay 서비스 실행됨 확인.

![8](https://user-images.githubusercontent.com/27837607/100531031-00a2df80-323d-11eb-8efb-e4818c3d5b55.JPG)

![10](https://user-images.githubusercontent.com/27837607/100531052-3f389a00-323d-11eb-9ec1-9458f5248c2b.JPG)

## CQRS
viewer를 별도로 구현하여 아래와 같이 view 실행 결과 확인 가능

![13](https://user-images.githubusercontent.com/27837607/100533694-88dbb180-324a-11eb-9289-caea76913b8f.JPG)


## Poly Glot

다른 서비스와 달리 Order 서비스는 h2db 가 아닌 hsqldb 를 사용

![11](https://user-images.githubusercontent.com/27837607/100531109-a0f90400-323d-11eb-9f21-3ee4e5e4e4d8.JPG)



# 운영

## CI / CD

![14](https://user-images.githubusercontent.com/27837607/100535315-1b844c80-325b-11eb-9644-5e5f195ff4bd.JPG)

![15](https://user-images.githubusercontent.com/27837607/100535316-1d4e1000-325b-11eb-88cd-e90c1ea654cf.JPG)

![16](https://user-images.githubusercontent.com/27837607/100535317-1e7f3d00-325b-11eb-8e5e-65715d9cac34.JPG)

![17](https://user-images.githubusercontent.com/27837607/100535318-20490080-325b-11eb-9a94-2654eeacd754.JPG)

![18](https://user-images.githubusercontent.com/27837607/100535319-217a2d80-325b-11eb-9bf5-76209f772583.JPG)



## SLA 준수
### Liveness Test
deployment.yml파일 내용 가운데
http -> tcpSocket변경, 8080 -> 8081로 변경 후, 서비스 확인 수행

![20](https://user-images.githubusercontent.com/27837607/100538970-492abf00-3276-11eb-98d0-3fa144917779.JPG)

![18](https://user-images.githubusercontent.com/27837607/100535319-217a2d80-325b-11eb-9bf5-76209f772583.JPG)

kubectl describe deploy order 명령어로 확인시 아래와 같은 결과 확인.

![21](https://user-images.githubusercontent.com/27837607/100542889-489f2200-3290-11eb-868b-db337d434de5.JPG)

### Readness Test



## 유연성
