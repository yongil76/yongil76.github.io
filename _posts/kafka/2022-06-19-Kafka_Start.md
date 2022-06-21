---
title:  "Kafka를 docker로 배포하고, Springboot 테스트 프로젝트 만들기"

excerpt: "Kafka"

categories:
- Kafka

tags:
- Kafka

toc: true

toc_sticky: true

toc_label: "kafka"

last_modified_at: 2022-06-19T09:59:00-05:00

---

## Kafka?

> Apache Kafka is a community distributed event streaming technology capable of handling trillions of events a day. Initially conceived as a messaging queue, Kafka is based on an abstraction of a distributed commit log. Since being created and open sourced by LinkedIn in 2011, Kafka has quickly evolved from messaging queue to a full-fledged event streaming platform.

## Message queue?

>What are Apache Kafka Queues? In the Apache Kafka Queueing system, messages are saved in a queue fashion. This allows messages in the queue to be ingested by one or more consumers, but one consumer can only consume each message at a time.

## Zookeeper?

>ZooKeeper is used in distributed systems for service synchronization and as a naming registry. When working with Apache Kafka, ZooKeeper is primarily used to track the status of nodes in the Kafka cluster and maintain a list of Kafka topics and messages.

>[Apache Kafka Needs No Keeper: Removing the Apache ZooKeeper Dependency](https://www.confluent.io/ko-kr/blog/removing-zookeeper-dependency-in-kafka/)

---

## Docker에 카프카 배포해보기(Mac)

### 조건

- Docker 설치
- docker-compose, git 명령어 사용 가능한 상태

### 작업

- 로컬 디렉토리(Mac)애 카프카 프로젝트 clone
~~~shell
git clone https://github.com/wurstmeister/kafka-docker
~~~

- 로컬 테스트를 위해 설정 파일 변경
~~~shell
cd kafka-docker
vi docker-compose-single-broker.yml
~~~

~~~shell
version: '2'
services:
  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
  kafka:
    build: .
    ports:
      - "9094:9094"
    environment:
      KAFKA_ADVERTISED_LISTENERS: INSIDE://:9092,OUTSIDE://localhost:9094
      KAFKA_LISTENERS: INSIDE://:9092,OUTSIDE://0.0.0.0:9094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INSIDE:PLAINTEXT,OUTSIDE:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: INSIDE
      KAFKA_CREATE_TOPICS: "test:1:1"
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock\
    depends_on:
      - zookeeper
  kafka-ui:
    image: provectuslabs/kafka-ui
    container_name: kafka-ui
    ports:
      - "8080:8080"
    restart: always
    environment:
      - KAFKA_CLUSTERS_0_NAME=local
      - KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kafka:9092
      - KAFKA_CLUSTERS_0_ZOOKEEPER=zookeeper:2181

~~~


- > KAFKA_ADVERTISED_LISTENERS
  - 카프카 접속 가능한 URL
  - localhost는 스프링 부트와 연동을 위한 URL
  

- > KAFKA_LISTENERS
  - 카프카 서버의 내부 리스너
  - 0.0.0.0으로 설정해야, 카프카 서버의 내부 인터페이스와 통신이 가능
  - 0.0.0.0은 카프카 서버의 모든 인터페이스에서 수신이 가능


- docker-compose 명령어로 docker에 배포하기
~~~shell
docker-compose -f ./docker-compose-single-broker.yml up -d
~~~

- docker container 확인
~~~shell
docker container ls
~~~

- docker container 명령어

~~~shell
CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS                                                NAMES
f9057c5969c0   kafka-docker_kafka       "start-kafka.sh"         10 minutes ago   Up 10 minutes   0.0.0.0:9094->9094/tcp                               kafka-docker-kafka-1
40db6adb6d67   provectuslabs/kafka-ui   "/bin/sh -c 'java $J…"   10 minutes ago   Up 10 minutes   0.0.0.0:8080->8080/tcp                               kafka-ui
c551173b726b   wurstmeister/zookeeper   "/bin/sh -c '/usr/sb…"   10 minutes ago   Up 10 minutes   22/tcp, 2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp   kafka-docker-zookeeper-1~~~
~~~

- UI for Apache Kafka 연동
  - GUI 모니터링 툴
  - 브로커 상태 확인
    ![](assets/images/kafka/UI_for_apache_kafka_dashboard.png)
  - 토픽 메시지 확인
    ![](assets/images/kafka/UI_for_apache_kafka_topic.png)

---

## Springboot로 테스트해보기

### 조건
- 위의 Docker에 카프카 배포되기가 완료된 상태

### Configuration
~~~java
package me.yongil.spring_kafka.config;

import java.util.HashMap;
import java.util.Map;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.config.KafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.core.ProducerFactory;
import org.springframework.kafka.listener.ConcurrentMessageListenerContainer;


@Configuration
@EnableKafka
public class KafkaConfiguration {
    @Bean
    public ProducerFactory<String, String> producerFactory() {
        return new DefaultKafkaProducerFactory<>(producerConfigs());
    }

    @Bean
    public Map<String, Object> producerConfigs() {
        final Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9094");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        // See https://kafka.apache.org/documentation/#producerconfigs for more properties
        return props;
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }

    @Bean
    KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>> kafkaListenerContainerFactory() {
        final ConcurrentKafkaListenerContainerFactory<String, String> factory =
                new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);
        factory.getContainerProperties().setPollTimeout(3000);
        return factory;
    }

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        return new DefaultKafkaConsumerFactory<>(consumerConfigs());
    }

    @Bean
    public Map<String, Object> consumerConfigs() {
        final Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9094");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        return props;
    }
}

~~~

### Consumer
~~~java
package me.yongil.spring_kafka.service;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

@Component
public class KafkaConsumer {

    @KafkaListener(
            id = "yong",
            topics = "test",
            clientIdPrefix = "clientId",
            properties = {"enable.auto.commit:false", "auto.offset.reset:latest"}
    )
    public void listen(String data) {
        System.out.println("Consumed data : " + data);
    }

}
~~~

### Producer
~~~java
package me.yongil.spring_kafka.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

/**
 * @author yongil.kwon@linecorp.com
 */
@Component
public class KafkaProducer {
    private final KafkaTemplate<String, String> kafkaTemplate;

    public KafkaProducer(@Autowired KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void send(String text) {
        kafkaTemplate.send("test", text);
    }
}
~~~

- 테스트 방법
  - Docker에 카프카가 정상적으로 배포된 상황에서,
  - [spring_kafka](https://github.com/yongil76/spring_kafka)
  - spring_kafka 프로젝트를 로컬(Spring boot)에서 띄운 후에, {localhost}:{port}/send?text={text}

---


