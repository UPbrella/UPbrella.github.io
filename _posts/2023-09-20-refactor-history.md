---
title: JPA Repository는 Entity만을 조회해야할까? 
layout: post
author: 임동현
subtitle: 'jpa repository'
date: 2023-09-14 00:00:00 +0900
categories:
- JPA
tags:
- JPA
---

## 1. 문제 정의
대여 기록을 관리하기 위해서는 여러 테이블과의 조인이 필요합니다. 
문제는 여기서 발생하는데요, 기존의 업브렐라 개발팀은 개발 속도 및 편의를 위해 JPA Repository에서 Entity만 조회하였습니다.  
하지만 이는 성능에 많은 영향을 미치고 있었습니다.

## 2. 대여 기록 분석 
대여 기록은 여러 테이블을 조인하는데 비해 적은 수의 데이터를 반환하고 있습니다.   
코드를 먼저 살펴보겠습니다.

```java
    @Override
    public List<History> findAll(HistoryFilterRequest filter, Pageable pageable) {

        return queryFactory
                .selectFrom(history)
                .join(history.user, user).fetchJoin()
                .leftJoin(history.refundedBy, user).fetchJoin()
                .join(history.umbrella, umbrella).fetchJoin()
                .join(history.rentStoreMeta, storeMeta).fetchJoin()
                .leftJoin(history.returnStoreMeta, storeMeta).fetchJoin()
                .where(filterRefunded(filter))
                .orderBy(history.id.desc())
                .offset(pageable.getOffset())
                .limit(pageable.getPageSize())
                .fetch();
    }
```
history를 한번에 조회하면서 연관된 Entity들을 fetchJoin으로 가져오고 있습니다.  
이는 코드를 간편하게 작성할 수 있게 해주지만, 필요하지 않은 필드들까지 가져온다는 단점을 가지고 있습니다. 

이에 해당하는 쿼리는 다음과 같습니다.  
```sql
select
        history0_.id as id1_4_0_,
        user1_.id as id1_10_1_,
        user2_.id as id1_10_2_,
        umbrella3_.id as id1_9_3_,
        storemeta4_.id as id1_8_4_,
        storemeta5_.id as id1_8_5_,
        history0_.account_number as account_2_4_0_,
        history0_.bank as bank3_4_0_,
        history0_.etc as etc4_4_0_,
        history0_.paid_at as paid_at5_4_0_,
        history0_.paid_by as paid_by9_4_0_,
        history0_.refunded_at as refunded6_4_0_,
        history0_.refunded_by as refunde10_4_0_,
        history0_.rent_store_meta_id as rent_st11_4_0_,
        history0_.rented_at as rented_a7_4_0_,
        history0_.return_store_meta_id as return_12_4_0_,
        history0_.returned_at as returned8_4_0_,
        history0_.umbrella_id as umbrell13_4_0_,
        history0_.user_id as user_id14_4_0_,
        user1_.account_number as account_2_10_1_,
        user1_.admin_status as admin_st3_10_1_,
        user1_.bank as bank4_10_1_,
        user1_.email as email5_10_1_,
        user1_.name as name6_10_1_,
        user1_.phone_number as phone_nu7_10_1_,
        user1_.social_id as social_i8_10_1_,
        user2_.account_number as account_2_10_2_,
        user2_.admin_status as admin_st3_10_2_,
        user2_.bank as bank4_10_2_,
        user2_.email as email5_10_2_,
        user2_.name as name6_10_2_,
        user2_.phone_number as phone_nu7_10_2_,
        user2_.social_id as social_i8_10_2_,
        umbrella3_.created_at as created_2_9_3_,
        umbrella3_.deleted as deleted3_9_3_,
        umbrella3_.etc as etc4_9_3_,
        umbrella3_.missed as missed5_9_3_,
        umbrella3_.rentable as rentable6_9_3_,
        umbrella3_.store_meta_id as store_me8_9_3_,
        umbrella3_.uuid as uuid7_9_3_,
        storemeta4_.activated as activate2_8_4_,
        storemeta4_.category as category3_8_4_,
        storemeta4_.classification_id as classifi9_8_4_,
        storemeta4_.deleted as deleted4_8_4_,
        storemeta4_.latitude as latitude5_8_4_,
        storemeta4_.longitude as longitud6_8_4_,
        storemeta4_.name as name7_8_4_,
        storemeta4_.password as password8_8_4_,
        storemeta4_.sub_classification_id as sub_cla10_8_4_,
        storemeta5_.activated as activate2_8_5_,
        storemeta5_.category as category3_8_5_,
        storemeta5_.classification_id as classifi9_8_5_,
        storemeta5_.deleted as deleted4_8_5_,
        storemeta5_.latitude as latitude5_8_5_,
        storemeta5_.longitude as longitud6_8_5_,
        storemeta5_.name as name7_8_5_,
        storemeta5_.password as password8_8_5_,
        storemeta5_.sub_classification_id as sub_cla10_8_5_ 
    from
        history history0_ 
    inner join
        user user1_ 
            on history0_.user_id=user1_.id 
    left outer join
        user user2_ 
            on history0_.refunded_by=user2_.id 
    inner join
        umbrella umbrella3_ 
            on history0_.umbrella_id=umbrella3_.id 
    inner join
        store_meta storemeta4_ 
            on history0_.rent_store_meta_id=storemeta4_.id 
    left outer join
        store_meta storemeta5_ 
            on history0_.return_store_meta_id=storemeta5_.id 
    order by
        history0_.id desc
```
  
실제로 필요한 필드는 12개인데 반해, 58개의 필드를 조회하고 있습니다.  
이를 개선하기 위해 Entity 대신 DTO를 조회하도록 하겠습니다.

```java
public List<HistoryInfoDto> findHistoryInfos(HistoryFilterRequest filter, Pageable pageable) {

        return queryFactory
                .select(new QTestDto(
                        history.id,
                        history.user.name,
                        history.user.phoneNumber,
                        history.rentStoreMeta.name,
                        history.rentedAt,
                        history.umbrella.uuid,
                        history.returnStoreMeta.name,
                        history.returnedAt,
                        history.paidAt,
                        history.bank,
                        history.accountNumber,
                        history.etc,
                        history.refundedAt
                ))
                .from(history)
                .join(history.user, user)
                .join(history.umbrella, umbrella)
                .join(history.rentStoreMeta, storeMeta)
                .leftJoin(history.returnStoreMeta, storeMeta)
                .where(filterRefunded(filter))
                .orderBy(history.id.desc())
                .offset(pageable.getOffset())
                .limit(pageable.getPageSize())
                .fetch();
    }
```
코드가 길어지긴 하였지만, SQL을 살펴보면 획기적으로 개선되었음을 알 수 있습니다.

```sql
select
        history0_.id as col_0_0_,
        user1_.name as col_1_0_,
        user1_.phone_number as col_2_0_,
        storemeta4_.name as col_3_0_,
        history0_.rented_at as col_4_0_,
        umbrella2_.uuid as col_5_0_,
        storemeta4_.name as col_6_0_,
        history0_.refunded_at as col_7_0_,
        history0_.paid_at as col_8_0_,
        history0_.bank as col_9_0_,
        history0_.account_number as col_10_0_,
        history0_.etc as col_11_0_ 
    from
        history history0_ 
    inner join
        user user1_ 
            on history0_.user_id=user1_.id 
    inner join
        umbrella umbrella2_ 
            on history0_.umbrella_id=umbrella2_.id 
    inner join
        store_meta storemeta3_ 
            on history0_.rent_store_meta_id=storemeta3_.id 
    left outer join
        store_meta storemeta4_ 
            on history0_.return_store_meta_id=storemeta4_.id 
    order by
        history0_.id desc limit ?
``` 

## 3. 쿼리 실행 계획 분석  
**개선 전**
![image](https://user-images.githubusercontent.com/115435784/269452425-9774239a-9fc2-450d-b870-a636504dfa52.png)

**개선 후**
![image](https://user-images.githubusercontent.com/115435784/269452308-5960b9e5-62f4-486f-8632-17befaf18ced.png)
  
JPA Repository에서 Entity만 조회했을 때와, DTO를 조회했을 때의 성능을 비교해보면,  

쿼리 속도는 0.032s -> 0.012s 로 62.5% 개선되었고,  
쿼리 비용은 6284054.81 -> 5198930.47로 17.3% 개선되었습니다.
  
## 4. 마치며 
이번 포스팅을 통해 JPA Repsitory에서 Entity를 조회하는 것이 성능에 미치는 영향을 알아보았습니다.  
이처럼 Entity를 조회하는 경우 성능에 문제가 생길 수 있기 때문에, 실시간으로 Entity를 변경해야 하는 것이 아니라면, DTO를 조회하는 것이 성능에 좋은 영향을 미치게 됩니다. 
