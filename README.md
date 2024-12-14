# Private Multi-Account Grafana AMP Monitoring

프라이빗 네트워크 및 멀티 어카운트 AWS 환경에서 **Grafana**, **Amazon Managed Prometheus (AMP)**, **CloudWatch** 및 **Grafana Agent**를 활용한 모니터링 시스템 구축입니다.

- 네트워크
  - Private Subnet에서 VPC Endpoints 이용한 통신
  - Grafana -> Target(Cloudwatch) : sts, monitoring, logs
  - Grafana -> AMP : sts, aps, aps-workspaces, logs, monitoring
  - Target -> AMP : sts, aps, aps-workspaces
![Private Multi-Account Grafana AMP Monitoring - Flow Image](images/flow.png)
- Assume Role
![Private Multi-Account Grafana AMP Monitoring - Role Image](images/role.png)

## 목차
- [AWS Account IAM Configuration Guide](#aws-account-iam-configuration-guide)
- [AWS Account Infrastructure Configuration Guide](#aws-account-infrastructure-configuration-guide)
- [OSS Installation and Configuration Guide](#oss-installation-and-configuration-guide)
- [Grafana Data Sources Configuration Guide](#grafana-data-sources-configuration-guide)
- [Single EC2 Test](#Single-EC2-Test)
- [라이선스](#라이선스)

---

## AWS Account IAM Configuration Guide

이 섹션에서는 각 AWS 계정에서 필요한 IAM(Identity and Access Management) 설정 방법을 다룹니다.

- **Role 설정**
  - 각 계정에서 다른 계정의 리소스에 접근할 수 있도록 역할(Role) 생성
  - 신뢰 정책(Trust Policy) 구성 (- 계정 간 신뢰 관계 설정)
- **Permissions Management**
  - 필요한 권한 정책(Policies) 정의 및 첨부
  - 최소 권한 원칙(Least Privilege Principle) 적용

자세한 내용은 [AWS Account IAM Configuration Docs](aws-iam-config/aws-iam-config.md)를 참조하세요.

## AWS Account Infrastructure Configuration Guide

이 섹션에서는 각 AWS 계정에서 모니터링 인프라를 설정하는 방법을 다룹니다. 각 리소스는 NAT가 있어도 이를 이용하여 통신하지 않도록 구성 (내부 통신)

- **VPC and Subnet Setup**
  - 프라이빗 서브넷 및 VPC 엔드포인트 구성
  - 네트워크 보안 그룹(Security Groups) 설정
- **Grafana OSS and AMP Integration**
  - Grafana OSS 와 Amazon Managed Prometheus(AMP) 연동 설정
- **Resource Deployment**
  - Grafana Agent 및 Node Exporter 배포

자세한 내용은 [AWS Account Infrastructure Configuration Docs](aws-infra-config/aws-infra-config.md)를 참조하세요.


## OSS Installation and Configuration Guide

이 섹션에서는 각 서버에 필요한 구성 요소를 설치하고 설정하는 방법을 다룹니다.

- **Grafana Server Setup**
  - Grafana OSS 설치
  - ~~MySQL DB 구성 및 연동~~
  - ~~보안 설정 및 접근 제어~~
- **Monitoring Target Setup**
  - Node Exporter 설치 및 구성
  - Grafana Agent 설치
  - CloudWatch Agent 설치

자세한 내용은 [OSS Installation and Configuration Docs](oss-installation-and-configuration-guide/oss-installation-and-configuration-guide.md)를 참조하세요.


## Grafana Data Sources Configuration Guide

이 섹션에서는 Grafana에서 CloudWatch와 AMP 데이터 소스를 구성하는 방법을 설명합니다.

- **Data Source Registration**
  - CloudWatch Data Source 등록 및 설정
  - AMP Data Source 등록 및 설정

자세한 내용은 [Grafana Data Sources Configuration Docs](grafana-datasource-config/grafana-datasource-config.md)를 참조하세요.

## ~~Metric~~
- 작성 필요


## Single EC2 Test

Public Single EC2 테스트를 통해 아래 구성 요소의 작동을 검증했습니다:

- **Prometheus**  
  Prometheus는 메트릭 수집 및 저장을 담당하며, EC2 인스턴스의 다양한 메트릭 데이터를 받아 저장합니다.

- **Grafana**  
  Grafana는 Prometheus에서 수집한 메트릭 데이터를 시각화하고, 사용자 정의 대시보드를 통해 시스템 모니터링 환경을 제공합니다.

- **Grafana Agent**  
  EC2 메타데이터 및 Node Exporter의 데이터를 수집하여 Prometheus로 전달하는 경량화된 에이전트입니다.

- **Node Exporter**  
  리눅스 기반 EC2 인스턴스에서 CPU, 메모리, 디스크 사용량 등의 하드웨어 및 시스템 수준 메트릭을 수집합니다.

자세한 구성 및 설정은 [Single EC2 Test Docs](single-ec2-test/single-ec2-test.md)를 참조하세요.

---

## 라이선스
이 프로젝트는 MIT 라이선스 하에 배포됩니다. 자세한 내용은 [LICENSE](LICENSE)를 참조하세요.
