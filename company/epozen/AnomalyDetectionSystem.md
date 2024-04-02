# Anomaly Detection System

## 📝프로젝트 소개

---

**Anomaly Detection System**은 센서를 통해 수집한 데이터를 이용하여 **인공지능(AI) 분석**을 진행하고 특이 사항을 찾아내어 **기계적 결함을 유추**하는 시스템으로 **AI 도입을 통해 최소한의 인력, 시간 투입**으로 최대의 효과를 내는 **이상 탐지 시스템**입니다.

## 🗓️진행 기간

- 2020.05 ~ 2020.09

## 🌟주요 프로세스

---

### 수집 & 저장 프로세스

- 개발 언어 : Java + SpringBoot
- 목적 : Gateway 서버로 부터 전송되는 데이터를 수집하여 전처리하고 용도에 맞게 DB, Elasticsearch에 유실없이 저장하는 것이 주 목적인 프로세스
- 핵심 기능 : Restful Application으로 외부로 부터 발생하는 모든 요청을 처리하고 기능 확장에 용이하며, 여러 서비스 (Elasitcsearch, PostgreSQL, Notify & Listerner, 분석 서비스) 들 간의 중간 관리자로서 원활한 서비스를 제공하는 것이 중요하다.

### Notify & Listener 프로세스

- 개발 언어 : Java + SpringBoot
- 목적 : PostgreSQL의 지정한 테이블들에 대해 발생하는 CRUD 작업을 캐치하는 것이 주 목적인 프로세스
- 핵심 기능 : Elasticsearch Info 테이블에 데이터가 Insert 되면 분석 서비스로 즉각 Notify 해주는 것이 중요하다.

### 분석 프로세스

- 개발 언어 : Python + Flask
- 목적 : 수집된 센서 데이터를 분석하여 이상여부를 탐지하고 결과를 도출해내는 것이 주 목적인 프로세스
- 핵심 기능 : 수집된 소음 데이터를 푸리에 변환을 사용하여 주파수 그래프인 스펙트로그램을 얻어 학습 모델을 생성한다. 이렇게 기 생성된 학습 모델을 사용하여 새로 수집한 데이터의 이상 탐지 여부를 분석하는 것이 중요하다.

## 🤚나의 역할

---

1. 컨테이너 기반 운영 환경 구축 ( Docker + Kubernates 클러스터링 서버 )
2. 수집&저장 프로세스 개발
3. Notify & Listener 프로세스 개발
4. 분석 프로세스 개발
    1. 학습 모델 작성
    2. 분석 로직 작성

## 🗂️프로젝트 구조

1. Gateway 서버-K8S서버간 통신 프로토콜 : LTE 통신
2. K8S 서버 구성 : 3대의 클러스터링 서버로 구성하여 HA 구현
3. Database 구성 : PostgreSQL을 사용하였고, 역시 Master-Standby 형태로 HA 구현
4. 분석 서비스 학습 모델 구조
    1. 알고리즘 : VGG Net 사용
    2. 입력층 : 48*64 의 Spectrogram 전처리 데이터
    3. 특징 추출층 : 합성곱 필터(Convolution) : 2개, 풀링(Max Pooling) : 1개를 3회 수행
    4. 전 결합층 : 3개의 은닉층(유닛 1024개)
    5. 출력층 : 1개(유닛 18개)
    6. loss 함수 : 평균 제곱 오차 함수
    7. 최적화 함수 : AdamOptimizer

![Anomaly%20Detection%20System%207f6c8677a6c2488bb63227f17847656e/Untitled.png](Anomaly%20Detection%20System%207f6c8677a6c2488bb63227f17847656e/Untitled.png)

## 결과 / 성과

결과적으로는 고객의 요구사항은 맞췄지만 서비스되지 못했습니다. 여러가지 이유가 있었지만 비즈니스적인 이유가 주요했습니다.

성과적으로는 SpringBoot와 Mybatis 또한 인공 지능 개발에 참여하여 Python, ML 에 대한 기술 Skill과 Docker, Kuberantes 의 컨테이너 기반 서비스의 이해도를 높였습니다.

[✏️프로젝트 기술 스택](Anomaly%20Detection%20System%207f6c8677a6c2488bb63227f17847656e/%E2%9C%8F%EF%B8%8F%E1%84%91%E1%85%B3%E1%84%85%E1%85%A9%E1%84%8C%E1%85%A6%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%80%E1%85%B5%E1%84%89%E1%85%AE%E1%86%AF%20%E1%84%89%E1%85%B3%E1%84%90%E1%85%A2%E1%86%A8%20a4beda7dad084ac99eeec0d35fcb54c0.csv)
