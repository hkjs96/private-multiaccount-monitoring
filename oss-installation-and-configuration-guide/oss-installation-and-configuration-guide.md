# OSS Installation and Configuration Guide

이 가이드에서는 프라이빗 네트워크 환경의 멀티 어카운트 AWS 구성에서 Grafana OSS 설치 및 구성 방법을 설명합니다. 

> 모든 설치 과정은 **Amazon Linux 2023** 기준으로 작성되었습니다.

## 목차
- [1. Grafana OSS 설치](#1-grafana-oss-설치)
  - [1.1 패키지 설치 및 기본 설정](#11-패키지-설치-및-기본-설정)
  - [1.2 AWS 환경 설정](#12-aws-환경-설정)
  - [1.3 Grafana 설정](#13-grafana-설정)
  - [1.4 플러그인 설치](#14-플러그인-설치)
  - [1.5 서비스 관리](#15-서비스-관리)

- [2. Monitoring Agent 설치](#2-monitoring-agent-설치)
  - [2.1 Node Exporter 설치](#21-node-exporter-설치)
  - [2.2 Grafana Agent Flow 설치](#22-grafana-agent-flow-설치)
  - [2.3 Grafana Agent 설정](#23-grafana-agent-설정)
  - [2.4 Grafana Agent 서비스 설정](#24-grafana-agent-서비스-설정)
  - [2.5 메트릭 수집 흐름](#25-메트릭-수집-흐름)
  - [2.6 구성 요소 설명](#26-구성-요소-설명)
  - [2.7 메타데이터 Relabeling의 필요성](#27-메타데이터-relabeling의-필요성)

- [3. CloudWatch Agent 설치](#3-cloudwatch-agent-설치)
  - [3.1 운영체제별 설치 방법](#31-운영체제별-설치-방법)
  - [3.2 에이전트 설정](#32-에이전트-설정)
  - [3.3 수집 메트릭 상세](#33-수집-메트릭-상세)
  - [3.4 서비스 관리](#34-서비스-관리)


## 1. Grafana OSS 설치

### 1.1 패키지 설치 및 기본 설정
```bash
# Grafana Enterprise 11.1.3 설치 (2024년 7월 26일 기준)
sudo yum install -y https://dl.grafana.com/enterprise/release/grafana-enterprise-11.1.3-1.x86_64.rpm

# 서비스 활성화
sudo systemctl enable --now grafana-server
```

### 1.2 AWS 환경 설정
```bash
# 환경 설정 디렉토리 생성
sudo mkdir -p /etc/systemd/system/grafana-server.service.d/

# 서비스 오버라이드 설정 파일 생성
sudo tee /etc/systemd/system/grafana-server.service.d/override.conf << 'EOF'
[Service]
# STS 엔드포인트 설정 - IAM 자격 증명 검증
Environment="AWS_STS_REGIONAL_ENDPOINTS=regional"
Environment="GF_AWS_ASSUME_ROLE_ENABLED=true"

# APS 엔드포인트 설정 - 메트릭 쿼리/수집
Environment="AWS_PROMETHEUS_ENDPOINT=https://aps.<AMP_리전>.amazonaws.com"

# APS Workspaces 엔드포인트 설정 - 작업공간 관리
Environment="AWS_APS_WORKSPACES_ENDPOINT=https://aps-workspaces.<AMP_리전>.amazonaws.com"
EOF
```

### 1.3 Grafana 설정
```bash
# AWS SigV4 인증 활성화
sudo tee -a /etc/grafana/grafana.ini << 'EOF'
[auth.sigv4]
sigv4_auth_enabled = true
EOF
```

### 1.4 플러그인 설치
```bash
# Amazon Prometheus 데이터소스 플러그인 설치 (버전 1.0.1)
sudo rm -rf /var/lib/grafana/plugins/grafana-amazonprometheus-datasource
sudo grafana-cli plugins install grafana-amazonprometheus-datasource 1.0.1
```

### 1.5 서비스 관리
```bash
# 데몬 리로드
sudo systemctl daemon-reload

# 서비스 재시작 (설정 변경 시)
sudo systemctl restart grafana-server

# 서비스 상태 확인
sudo systemctl status grafana-server
```

### 버전 호환성 참고
2024년 11월 20일 이후 버전을 사용하는 경우 -> 참고: Amazon Prometheus 데이터소스 플러그인 1.0.3 버전과 호환
```bash
# Grafana 11.3.1 설치
sudo yum install -y https://dl.grafana.com/enterprise/release/grafana-enterprise-11.3.1-1.x86_64.rpm
```

## 2. Monitoring Agent 설치

### 2.1 Node Exporter 설치
Node Exporter는 시스템 메트릭을 수집하며, Grafana Agent는 이를 통해 메트릭을 수집하고 EC2 메타데이터와 함께 제공합니다.

```bash
# Node Exporter 1.8.2 다운로드 및 설치
sudo wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
sudo tar xvfz node_exporter-*.tar.gz
sudo mv node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/
sudo useradd -rs /bin/false node_exporter

# Node Exporter 서비스 설정
sudo tee /etc/systemd/system/node_exporter.service << EOF
[Unit]
Description=Node Exporter
[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter
[Install]
WantedBy=default.target
EOF

# Node Exporter 서비스 시작
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter
```

### 2.2 Grafana Agent Flow 설치
Grafana Agent Flow는 EC2 메타데이터를 수집하고 Node Exporter의 메트릭과 함께 AMP로 전송합니다.

```bash
# Grafana Agent Flow 패키지 설치
sudo curl -LO https://github.com/grafana/agent/releases/download/v0.43.3/grafana-agent-flow-0.43.3-1.amd64.rpm
sudo rpm -ivh grafana-agent-flow-0.43.3-1.amd64.rpm

# 사용자 및 그룹 설정
sudo groupadd grafana-agent-flow
sudo useradd --no-create-home --shell /bin/false grafana-agent-flow
sudo usermod -aG grafana-agent-flow grafana-agent-flow

# 설정 디렉토리 생성
sudo mkdir -p /etc/grafana-agent/
sudo chown -R grafana-agent-flow:grafana-agent-flow /etc/grafana-agent
```

### 2.3 Grafana Agent 설정
```bash
# Grafana Agent 설정 파일 생성
sudo tee /etc/grafana-agent/agent.river << 'EOF'
// 로깅 설정
logging {
  level = "info"
}

// Prometheus 원격 쓰기 설정
prometheus.remote_write "prom" {
  endpoint {
    url = "<프로메테우스_쓰기_엔드포인트_URL>"
    sigv4 {
      region = "<AMP_리전>"
      role_arn = "<ARN>"
    }
    metadata_config {
      send                 = true
      send_interval        = "1m"
      max_samples_per_send = 2000
    }
  }
}

// EC2 인스턴스 발견 설정
discovery.ec2 "ec2" {
  region      = "<EC2_리전>"
  port        = 9100
  filter {
    name   = "tag:Monitoring"
    values = ["true"]
  }
}

// EC2 메타데이터 리라벨링
discovery.relabel "ec2_metadata" {
  targets = discovery.ec2.ec2.targets

  rule {
    source_labels = ["__meta_ec2_instance_id"]
    target_label  = "instance_id"
  }
  rule {
    source_labels = ["__meta_ec2_availability_zone"]
    target_label  = "availability_zone"
  }
  rule {
    source_labels = ["__meta_ec2_instance_type"]
    target_label  = "instance_type"
  }
  rule {
    source_labels = ["__meta_ec2_tag_Name"]
    target_label  = "instance_name"
  }
  rule {
    source_labels = ["__meta_ec2_private_ip"]
    target_label  = "private_ip"
  }
}

// 스크레이핑된 메트릭을 AMP로 전달
prometheus.scrape "instances" {
  targets = discovery.relabel.ec2_metadata.output
  honor_labels = true
  forward_to = [prometheus.remote_write.prom.receiver]
}
EOF

# 설정 파일 권한 설정
sudo chown grafana-agent-flow:grafana-agent-flow /etc/grafana-agent/agent.river
```

### 2.4 Grafana Agent 서비스 설정
```bash
# 서비스 파일 생성
sudo tee /etc/systemd/system/grafana-agent-flow.service << EOF
[Unit]
Description=Grafana Agent Flow
After=network.target

[Service]
Environment="AWS_STS_REGIONAL_ENDPOINTS=regional"
ExecStart=/usr/bin/grafana-agent-flow run /etc/grafana-agent/agent.river
Restart=on-failure
User=grafana-agent-flow
Group=grafana-agent-flow
WorkingDirectory=/etc/grafana-agent

[Install]
WantedBy=multi-user.target
EOF

# 서비스 시작
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-agent-flow
sudo systemctl status grafana-agent-flow
```


### 2.5 메트릭 수집 흐름
1. EC2 Discovery -> 2. 메타데이터 Relabeling -> 3. 메트릭 Scraping -> 4. AMP로 전송

```
EC2 Instance
    ↓
Node Exporter (9100 포트)
    ↓
Grafana Agent
    ├─ EC2 메타데이터 수집 (discovery.ec2)
    ├─ 메타데이터 레이블 변환 (discovery.relabel)
    └─ 메트릭 스크래핑 (prometheus.scrape)
        ↓
AMP (Remote Write)
```

### 2.6 구성 요소 설명

1. **EC2 Discovery (discovery.ec2)**
   ```river
   discovery.ec2 "ec2" {
     region = "<EC2_리전>"
     port   = 9100
     filter {
       name   = "tag:Monitoring"
       values = ["true"]
     }
   }
   ```
   - EC2 인스턴스를 자동으로 발견하는 기능
   - `filter`: 특정 태그("Monitoring=true")가 있는 인스턴스만 선택
   - `port`: Node Exporter가 실행 중인 포트 지정
   - 결과로 `__meta_ec2_*` 형식의 메타데이터 레이블 생성

2. **메타데이터 Relabeling (discovery.relabel)**
   ```river
   discovery.relabel "ec2_metadata" {
     targets = discovery.ec2.ec2.targets
     rule {
       source_labels = ["__meta_ec2_instance_id"]
       target_label  = "instance_id"
     }
     // 추가 규칙들...
   }
   ```
   - EC2 Discovery에서 생성된 메타데이터를 사용자 정의 레이블로 변환
   - `source_labels`: EC2에서 제공하는 원본 메타데이터 레이블
   - `target_label`: 새로 생성할 레이블 이름
   - 변환 예시:
     - `__meta_ec2_instance_id` → `instance_id`
     - `__meta_ec2_availability_zone` → `availability_zone`

3. **메트릭 Scraping (prometheus.scrape)**
   ```river
   prometheus.scrape "instances" {
     targets = discovery.relabel.ec2_metadata.output
     honor_labels = true
     forward_to = [prometheus.remote_write.prom.receiver]
   }
   ```
   - Relabeling된 타겟에서 실제 메트릭을 수집
   - `targets`: Relabeling 과정을 거친 EC2 인스턴스 목록
   - `honor_labels`: 원본 메트릭의 레이블 유지
   - `forward_to`: 수집된 메트릭을 AMP로 전송

4. **Remote Write 설정**
   ```river
   prometheus.remote_write "prom" {
     endpoint {
       url = "<프로메테우스_쓰기_엔드포인트_URL>"
       sigv4 {
         region = "<AMP_리전>"
         role_arn = "<ARN>"
       }
     }
   }
   ```
   - 수집된 메트릭을 AMP로 전송하는 설정
   - AWS SigV4 인증을 사용하여 안전한 전송
   - `metadata_config`: 메타데이터 전송 주기와 양 조절

### 2.7 메타데이터 Relabeling의 필요성
EC2 Discovery를 통해 생성되는 `__meta_*` 형식의 메타데이터 레이블은 내부 레이블로 분류되어 스크래핑 과정 이후에는 자동으로 삭제됩니다. 

```
    메타데이터 레이블(스크래핑 후 삭제됨)     →     영구 레이블(보존됨)
    __meta_ec2_instance_id                →     instance_id
    __meta_ec2_availability_zone          →     availability_zone
    __meta_ec2_instance_type             →     instance_type
```

리레이블링을 하지 않을 경우:
- 인스턴스 식별 정보 손실
- 가용 영역 정보 손실
- 인스턴스 유형 정보 손실
- 기타 EC2 관련 메타데이터 손실

이러한 정보 손실을 방지하고 메트릭과 함께 중요한 컨텍스트 정보를 유지하기 위해 discovery.relabel을 통한 리레이블링이 필수적입니다.


## 3. CloudWatch Agent 설치

### 3.1 운영체제별 설치 방법

#### RHEL 9.3
```bash
# 에이전트 패키지 다운로드 및 설치
wget https://amazoncloudwatch-agent-ap-northeast-2.s3.ap-northeast-2.amazonaws.com/redhat/amd64/latest/amazon-cloudwatch-agent.rpm
sudo rpm -U ./amazon-cloudwatch-agent.rpm
```

#### Amazon Linux 2023
```bash
# yum을 통한 직접 설치
sudo yum install amazon-cloudwatch-agent -y
```

### 3.2 에이전트 설정

```bash
# 설정 파일 생성
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'EOF'
{
    "agent": {
        "metrics_collection_interval": 60,
        "run_as_user": "cwagent"
    },
    "metrics": {
        "append_dimensions": {
            "InstanceId": "${aws:InstanceId}",
            "ImageId": "${aws:ImageId}",
            "InstanceType": "${aws:InstanceType}"
        },
        "aggregation_dimensions": [
            ["InstanceId"],
            ["ImageId", "InstanceId", "InstanceType"]
        ],
        "metrics_collected": {
            "cpu": {
                "measurement": [
                    "cpu_usage_idle",
                    "cpu_usage_user",
                    "cpu_usage_system"
                ],
                "totalcpu": true,
                "metrics_collection_interval": 60
            },
            "mem": {
                "measurement": [
                    "mem_used_percent",
                    "mem_total",
                    "mem_used"
                ],
                "metrics_collection_interval": 60
            },
            "disk": {
                "measurement": [
                    "disk_used_percent",
                    "disk_free",
                    "disk_used"
                ],
                "resources": ["/"],
                "metrics_collection_interval": 60
            },
            "agent": {
                "measurement": [
                    "agent_running"
                ]
            }
        }
    }
}
EOF

# 설정 적용
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

### 3.3 수집 메트릭 상세

1. **CPU 메트릭**
   - cpu_usage_idle: CPU 유휴 상태 비율
   - cpu_usage_user: 사용자 프로세스의 CPU 사용률
   - cpu_usage_system: 시스템 프로세스의 CPU 사용률

2. **메모리 메트릭**
   - mem_used_percent: 메모리 사용률
   - mem_total: 전체 메모리
   - mem_used: 사용 중인 메모리

3. **디스크 메트릭**
   - disk_used_percent: 디스크 사용률
   - disk_free: 여유 디스크 공간
   - disk_used: 사용 중인 디스크 공간

4. **에이전트 상태**
   - agent_running: 에이전트 실행 상태

### 3.4 서비스 관리

```bash
# 서비스 상태 확인
sudo systemctl status amazon-cloudwatch-agent

# 로그 확인
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log

# 설정 검증
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```
