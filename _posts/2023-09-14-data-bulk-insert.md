---
title: 대용량 데이터 삽입
layout: post
author: 임동현
subtitle: 'bulk insert'
date: 2023-09-14 00:00:00 +0900
categories:
- MySQL
tags:
- MySQL
---

## 1. 문제 정의

데이터 베이스 성능을 개선하는 것은 품질 좋은 서버를 유지하기 위해서 필수적입니다.

하지만 개발자들이 수작업으로 데이터를 삽입한 후 테스트 하는 것은 한계가 있습니다.

따라서 업브렐라 개발팀은 Data Bulk Insert를 통해 DB에 대용량의 데이터를 삽입하고 성능 개선을 해보도록 하겠습니다.

## 2. Bulk Insert

### 2 - 1 데이터 만들기

우선 간단한 테이블인 협업지점 분류에 대한 데이터를 만들어보겠습니다.

classification 테이블은 id, type, name, latitude, longitude를 필드로 가지고 있는데, 이에 해당하는 데이터를 만들어 보겠습니다.

```java
private static void makeStoreMeta() {
        String filePath = "store_meta.csv";
        Random random = new Random();

        try (FileWriter writer = new FileWriter(filePath)) {
            writer.append("id,classification_id,sub_classification_id,activated,deleted,name,category,latitude,longitude,password");
            writer.append("\n");

            for (int i = 1; i <= 1_000_000; i++) {
                writer.append(Integer.toString(i));
                writer.append(",");
                writer.append(Long.toString(random.nextLong() % 1000)); // classification_id
                writer.append(",");
                writer.append(Long.toString(random.nextLong() % 1000)); // sub_classification_id
                writer.append(",");
                writer.append(Integer.toString(random.nextInt(2))); // activated
                writer.append(",");
                writer.append(Integer.toString(random.nextInt(2))); // deleted
                writer.append(",");
                writer.append("Store_" + i); // name
                writer.append(",");
                writer.append("Category_" + (random.nextInt(10) + 1)); // category
                writer.append(",");
                writer.append(random.nextDouble() * 180 - 90 + ""); // latitude
                writer.append(",");
                writer.append(random.nextDouble() * 360 - 180 + ""); // longitude
                writer.append(",");
                writer.append("Password_" + i); // password
                writer.append("\n");
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

        System.out.println("CSV file created successfully at: " + filePath);
    }
```

위와 같이 FileWriter를 통해 백만건의 랜덤한 Classification을 생성하였습니다.

### 2 - 2 데이터 전송

이와 같은 대용량의 데이터를 서버에서 직접 만들 경우, CPU 부족 현상으로 서버가 다운될 우려가 있어 데이터 생성은 로컬에서 진행하였습니다.

이제 scp 명령어를 통해 데이터를 서버에 전송해주겠습니다.

```java
scp -p classification.csv ubuntu@127.0.0.1:~/classification.csv
```

127.0.0.1에 서버의 ip를 기입하면 전송을 완료할 수 있습니다.

```
scp : 파일 전송 명령어  

p : 작성, 수정한 날짜를 보존함  

clsasification.csv 전송하려는 파일 명  

ubuntu@127.0.0.1 : 서버의 사용자명과 서버의 ip  

:~/classification.csv : 파일이 저장될 위치  
```
### 2 - 3 MySQL에 데이터 삽입

MySQL에서 파일을 Load해서 파일에 맞게 데이터를 INSERT 하기 위해서는 특정 위치에 해당 파일이 위치해야 합니다.

파일을 MySQL이 읽을 수 있는 위치로 이동시켜 줍니다.

```java
sudo mv classification.csv /var/lib/mysql-files/
```

이제 파일을 이동시켰으니 데이터를 삽입해보도록 하겠습니다.

```java
LOAD DATA INFILE '/var/lib/mysql-files/classification.csv' INTO TABLE classification
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

csv 파일을 생성할 때, 맨 윗 줄에 필드에 대한 설명을 적었기 때문에 ignore 1 rows를 해줍니다.

field 의 구분은 , 로 진행하고

line의 구분은 \n 으로 진행해줍니다.

## 3 마치며

이번 게시글을 통해 대용량의 데이터를 삽입하는 방법을 알아보았습니다.

다음 게시글에서는 대용량의 데이터를 통해 성능이 좋지 않은 API를 판별해보고 개선하는 방법에 대해 알아보겠습니다.
