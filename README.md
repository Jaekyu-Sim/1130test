# 분석 설계

## Event Storming 결과

MSAez를 사용한 event stroming결과

![캡처](https://user-images.githubusercontent.com/27837607/100529978-e9112a00-322f-11eb-9310-0043d188375b.JPG)



## 헥사고날 아키텍쳐


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


## Request 방식의 아키텍쳐

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


## 이벤트 드리븐 아키텍쳐

## Gateway

## Poly Glot


# 운영

## SLA 준수

## CI / CD

## 유연성
