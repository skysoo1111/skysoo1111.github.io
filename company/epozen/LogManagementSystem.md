# Log Management System

## 📝프로젝트 소개

---

**Log Management System**은 각 시스템 별 Application들의 로그를 모두 수집하여 저장하고, 시스템의 warning 로그, error 로그, 트랜잭션 관리, 전문 검색 등의 검색 기능을 제공하는 통합 로그 관리 시스템 입니다. 로그 발생량이 분당 약 1.5GB(약 8만건), **하루 약 2TB(약1.1억건)** 에 달하는  **대용량 로그를 실시간으로 처리하고 관리하는** **시스템**입니다.

<aside>
💡 로그 크기는 건당 약 20KB

</aside>

Message Broker로는 Tibco Rendezvous를 사용하였고 로그 수집기는 C, 로그 저장기는 Java로 개발 되었으며 Data Processor는 Elasticsearch를 이용했습니다.

## 🗓️ 진행 기간

- 2017.04 ~ 2017.12

## 🌟주요 프로세스

---

### 로그 수집 프로세스

- 개발 언어 : C
- 목적 : 특정 시스템에서 실시간으로 발생하는 대용량 로그에 대한 시스템 부하를 최소화하고 손실 없이 수집하는 것이 목적인 프로세스
- 핵심 기능 : 실시간으로 분당 약 8만건의 데이터가 수집되므로 프로세스의 안정적인 데이터 관리가 핵심이며, 수집 프로세스에 부하가 발생했을 경우 다른 프로세스들과 서버에 미치는 영향을 최소화하는 것이 중요

### 로그 저장 프로세스

- 개발 언어 : Java7
- 목적 : 수집된 대용량 로그를 분산 처리 시스템인 Elasticsearch에 손실 없이 저장을 하는 것이 목적인 프로세스
- 핵심 기능 : Elastcisearch에 데이터 저장시, 데이터별 템플릿 매핑을 최적화하여 인덱싱 처리를 최적화하는 것이 중요

### 로그 검색 프로세스

- Front-End 개발 언어 : C#
- Back-End 개발 언어 : Java7
- 목적 : 수십억건의 저장 데이터 중 검색 조건에 충족하는 필요 데이터만을 빠르게 찾아서 사용자에게 보여주는 것이 목적인 프로세스
- 핵심 기능 : 정확한 Elasticsearch Query DSL을 구성하여 빠르게 검색 결과를 도출하는 것이 중요

## 🤚나의 역할

---

1. Back-End 로그 저장기 개발
2. Back-End 로그 검색기 개발
3. Elasticsearch 클러스터링 서버 구성
4. Log Management System 운영

## 🗂️프로젝트 구조

1. MES 서버-Collector 서버간 통신 프로토콜 : 소켓 방식
2. Collector 서버-Elasticsearch간 통신 프로토콜 : HTTP 방식
3. Collector 서버 구성 : 클러스터링 구성하여 Fail-Over 기능(Active 프로세스가 Down되면 In-Active 프로세스가 Active로 전환되어 운영) 으로 고가용성 지원
    - Log Collector : MES 서버로부터 로그를 수집
    - ByPass : Log Collector가 수집한 메세지 로그를 Log Writer로 ByPass (부하 분산)
    - Log Writer : ByPass로부터 받은 로그를 Elasticsearch에 저장
    - Log Reader : Elasticsearch에 저장된 데이터를 검색
4. Elasticsearch 서버 구성
    - Master Node : 전체 노드의 관리 및 노드간 통신을 담당하는 역할
    - Data Node : 디스크에 데이터를 저장하는 역할
    - Searcher Node : 저장된 데이터를 검색하는 역할

![Log%20Management%20System%20ab5c74fa326645ef8971148311c3e21e/LogCollector_Architecture.png](Log%20Management%20System%20ab5c74fa326645ef8971148311c3e21e/LogCollector_Architecture.png)

## 결과 / 성과

결과적으로 프로젝트는 고객의 요구사항에 맞춰 개발되어 서비스 되었습니다.

성과로는 해당 프로젝트를 통해 대용량 로그의 실시간 처리, Elasticsearch를 이용한 데이터 분산 처리 및 검색, 큐의 이해 등 다양한 기술 스킬 및 운영 노하우를 얻었습니다.

[✏️프로젝트 기술 스택](Log%20Management%20System%20ab5c74fa326645ef8971148311c3e21e/%E2%9C%8F%EF%B8%8F%E1%84%91%E1%85%B3%E1%84%85%E1%85%A9%E1%84%8C%E1%85%A6%E1%86%A8%E1%84%90%E1%85%B3%20%E1%84%80%E1%85%B5%E1%84%89%E1%85%AE%E1%86%AF%20%E1%84%89%E1%85%B3%E1%84%90%E1%85%A2%E1%86%A8%20252e65361c604b3cb1f8f43934a12ebf.csv)
