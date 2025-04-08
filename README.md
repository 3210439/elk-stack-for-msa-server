# ELK Stack for MSA Server

## 개요
이 레포지토리는 전자상거래 주문 관리 시스템의 마이크로서비스 아키텍처(MSA)에서 로그 수집, 분석 및 시각화를 위해 ELK 스택(Elasticsearch, Logstash, Kibana)을 구현한 프로젝트입니다.  
`msa-order-api`와 통합되어 주문 관련 로그와 Kafka 이벤트를 수집하며, 운영 중 발생하는 문제를 디버깅하고 시스템 상태를 모니터링하는 데 초점을 맞췄습니다.  
Docker를 활용한 컨테이너화로 배포가 간편하며, 확장성과 유지보수성을 고려해 설계되었습니다.

이 프로젝트는 전체 MSA 시스템의 일부로, 아래와 같은 관련 레포지토리와 함께 동작합니다:
- **[`msa-order-api`](https://github.com/3210439/msa-order-api)**: 주문 관리 핵심 서비스
- **[`msa-product-api`](https://github.com/3210439/msa-product-api)**: 제품 관리 서비스
- **[`order-api-monitoring`](https://github.com/3210439/order-api-monitoring)**: Prometheus와 Grafana를 사용한 모니터링 스택

---

## 아키텍처
ELK 스택은 로그 데이터를 수집하고 분석하며 시각화하는 역할을 합니다. 아래는 아키텍처 개요입니다:

- **Elasticsearch**: 로그 데이터를 저장하고 검색 가능한 인덱스로 관리.
- **Logstash**: `msa-order-api`의 TCP 로그와 Kafka의 `order-event` 토픽에서 데이터를 수집하여 필터링 후 Elasticsearch로 전송.
- **Kibana**: 로그 데이터를 시각화하고 대시보드를 통해 시스템 상태를 모니터링.
- **Kafka**: `msa-order-api`에서 발행된 이벤트를 Logstash로 전달 (외부 서비스와 연결).

모든 컴포넌트는 Docker 컨테이너로 실행되며, `msa-order-api`와 동일한 네트워크(`msa-order-api_order-network`)를 공유합니다.

---

## 프로젝트 구성 요소


### docker-compose.yml
- **Elasticsearch**:
  - 이미지: `docker.elastic.co/elasticsearch/elasticsearch:8.13.0`
  - 포트: `9200` (REST API), `9300` (노드 간 통신)
  - 환경 변수: 단일 노드 설정, 보안 비활성화, JVM 메모리 설정(`-Xms512m -Xmx512m`)
  - 볼륨: `esdata` (데이터 지속성 보장)
- **Logstash**:
  - 이미지: `docker.elastic.co/logstash/logstash:8.13.0`
  - 포트: `5000` (TCP 입력), `9600` (모니터링)
  - 볼륨: `./logstash/pipeline` (커스텀 파이프라인 설정)
  - 의존성: Elasticsearch
- **Kibana**:
  - 이미지: `docker.elastic.co/kibana/kibana:8.13.0`
  - 포트: `5601` (웹 UI)
  - 환경 변수: Elasticsearch 호스트 설정(`http://elasticsearch:9200`)
  - 의존성: Elasticsearch

### logstash.conf
Logstash 파이프라인은 두 가지 입력 소스를 처리합니다:
1. **TCP 입력**:
   - 포트: `5000`
   - 형식: JSON
   - 출처: `msa-order-api`의 애플리케이션 로그
2. **Kafka 입력**:
   - 브로커: `kafka:29092`
   - 토픽: `order-event`
   - 형식: JSON
   - 출처: `msa-order-api`에서 발행된 주문 이벤트

**필터**:
- 로그 소스가 없는 경우 `source` 필드를 `order-api-log`로 추가.
- `@timestamp`를 ISO8601 형식으로 파싱.

**출력**:
- Elasticsearch로 전송하며, 인덱스는 `%{source}-logs-%{+YYYY.MM.dd}` 형식(예: `order-api-log-logs-2025.04.08`).

---

## 설치 및 실행

### 사전 요구 사항
- Docker 및 Docker Compose 설치
- `msa-order-api`가 실행 중이어야 하며, 동일한 네트워크(`msa-order-api_order-network`)가 생성되어 있어야 함.
- Kafka 브로커(`kafka:29092`)가 접근 가능해야 함.
