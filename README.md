# Private Multi-Account Grafana AMP Monitoring

![Private Multi-Account Grafana AMP Monitoring - Image](images/dashboard-screenshot.png)

프라이빗 네트워크 및 멀티 어카운트 AWS 환경에서 **Grafana**, **Amazon Managed Prometheus (AMP)**, **CloudWatch** 및 **Grafana Agent**를 활용한 모니터링 시스템 구축 프로젝트입니다.

## 목차
- [Single EC2 Test](#Single-EC2-Test)
- [라이선스](#라이선스)

---

## Single EC2 Test

싱글 퍼블릭 EC2 테스트를 통해 아래 구성 요소의 작동을 검증했습니다:

- **Prometheus**  
  Prometheus는 시스템 및 애플리케이션의 다양한 메트릭을 수집하고 저장하는 오픈 소스 모니터링 솔루션입니다. 이를 통해 서버의 상태와 성능을 실시간으로 모니터링할 수 있습니다.

- **Grafana**  
  Grafana는 Prometheus 등에서 수집한 메트릭 데이터를 시각화하고, 대시보드를 통해 모니터링 환경을 제공합니다. 또한, 특정 조건에 따른 알림 설정을 통해 시스템 이상을 빠르게 감지하고 대응할 수 있도록 지원합니다.

- **Grafana Agent**  
  Grafana Agent는 경량화된 데이터 수집기로, EC2 인스턴스의 메타데이터와 Node Exporter에서 수집한 메트릭을 Prometheus 서버로 전송합니다. 이를 통해 중앙 집중식 모니터링이 가능해집니다.

- **Node Exporter**  
  Node Exporter는 리눅스 시스템의 CPU 사용률, 메모리 사용량 등 다양한 하드웨어 및 운영 체제 수준의 메트릭을 수집하는 도구입니다. 이를 통해 시스템의 상태를 상세하게 모니터링할 수 있습니다.

자세한 구성 및 설정은 [Single EC2 Test Docs](docs/single-ec2-test.md)를 참조하세요.

---

## 라이선스
이 프로젝트는 MIT 라이선스 하에 배포됩니다. 자세한 내용은 [LICENSE](LICENSE)를 참조하세요.
