# Log Monitoring System

## 📝프로젝트 소개

---

**Log Monitoring System**은 시스템 로그를 수집, 분석하여 **App별 성능, 작업 처리 시간, 작업 큐 등**을 **모니터링**하고 시스템에서 발생할 수 있는 여러 **문제를 사전에 확인하고 빠르게 대응**하기 위한 **모니터링 솔루션**입니다.

## 🗓️진행 기간

- 2019.02 ~ 2019.12

## 🌟주요 프로세스

---

### Client

- WAS : Apache Tomcat
- 개발 언어 : Java8 + SpringBoot
- 템플릿 엔진 : Handlebars

### Queue Monitoring 프로세스

- 개발 언어 : Java8 + SpringBoot
- 목적 : 여러 MES 서버에서 기동중인 수백, 수천의 Application들의 작업 대기 큐 수를 모니터링하는 것이 목적인 프로세스
- 핵심 기능 : 각 Application에서 처리하는 작업 수보다 대기열에 쌓이는 작업 수가 더 많다면 문제가 될 수 있기 때문에 특정 Application에서 작업 대기 큐가 지속적으로 쌓이는지 모니터링하고 사용자에게 빠르게 알려주는 것이 중요

### Transaction Monitoring 프로세스

- 개발 언어 : Java8 + SpringBoot
- 목적 : 여러 MES 서버에서 기동중인 수백, 수천의 Application들의 각 트랜잭션을 모니터링하는 것이 목적인 프로세스
- 핵심 기능 : 각 Application은 하나의 작업을 수행할 때 거치는 단계가 있는데 그것은 Application 별로 상이하다. 때문에 실시간으로 진행되는 수많은 작업 단계들을 모니터링하고 원하는 값들만 추려서 빠르게 보여주는 것이 중요

### Performance Monitoring 프로세스

- 개발 언어 : Java8 + SpringBoot
- 목적 : 여러 MES 서버에서 기동중인 수백, 수천의 Application들의 각 작업 성능을 모니터링하는 것이 목적인 프로세스
- 핵심 기능 : 수많은 Application들이 처리하는 각 작업의 처리 속도는 모두 제각각이다.  따라서 각 Application들의 작업 처리 속도를 모니터링하고 특정 시점, 특정 작업의 처리 속도 등의 사용자가 원하는 값들만 빠르게 보여주는 것이 중요

## 🤚나의 역할

---

1. Transaction Monitoring 프로세스 개발
2. Performance Monitoring 프로세스 개발
3. Log Monitoring System 운영

## 🗂️프로젝트 구조

1. 각 프로세스-Collector 서버간 통신 프로토콜 : 소켓 방식
2. 사용자 Client-Monitoring 서버간 통신 프로토콜 : HTTP 방식
3. Monitoring 서버 구성 : 클러스터링 구성하여 Fail-Over 기능(Active 프로세스가 Down되면 In-Active 프로세스가 Active로 전환되어 운영) 으로 고가용성 지원
    - Service IP를 두어서 항상 사용자는 Service IP로 접근을 하게 되고 Active 상태의 Monitoring 서버로 요청을 전달한다.

![Log%20Monitoring%20System%2010beef9da9324ba99095d64e3b6f1059/LogMonitoring_Architecture.png](Log%20Monitoring%20System%2010beef9da9324ba99095d64e3b6f1059/LogMonitoring_Architecture.png)

## 결과 / 성과

결과적으로 프로젝트는 고객의 요구사항에 맞춰 개발되어 서비스 되었습니다.

성과로는 데이터 트랜잭션 처리, 병렬 프로그래밍 기술 스킬과 SpringBoot와 JPA의 기술 스택을 얻었습니다.

[✏️프로젝트 기술 스택](Log%20Monitoring%20System%2010beef9da9324ba99095d64e3b6f1059/%E2%9C%8F%EF%B8%8F%E1%84%91%E1%85%B3%E1%84%85%E1%85%A9%E1%84%8C%E1%85%A6%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%80%E1%85%B5%E1%84%89%E1%85%AE%E1%86%AF%20%E1%84%89%E1%85%B3%E1%84%90%E1%85%A2%E1%86%A8%209b09c28305494fe5a71d83b5a200d2a9.csv)
