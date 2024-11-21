# Single EC2 Test: Public EC2 Monitoring Setup

이 문서는 퍼블릭 EC2 인스턴스를 사용하여 Prometheus, Grafana, Grafana Agent, Node Exporter의 설정 및 작동을 검증하는 테스트 과정을 설명합니다.

---

## 테스트 구성 요소

- **Prometheus**  
  - 시스템 및 애플리케이션의 다양한 메트릭을 수집하고 저장하는 오픈 소스 모니터링 솔루션입니다.
  - 서버의 상태와 성능 실시간 모니터링

- **Grafana**  
  - Prometheus 등에서 수집한 메트릭 데이터를 시각화하고, 대시보드를 통해 모니터링 환경을 제공합니다.
  - 또한, 특정 조건에 따른 알림 설정을 통해 시스템 이상을 빠르게 감지하고 대응할 수 있도록 지원합니다.

- **Grafana Agent**  
  - 경량화된 데이터 수집기로, EC2 인스턴스의 메타데이터와 Node Exporter에서 수집한 메트릭을 Prometheus 서버로 전송합니다.
  - 중앙 집중식 모니터링 지원

- **Node Exporter**  
  - 리눅스 시스템의 CPU 사용률, 메모리 사용량 등 다양한 하드웨어 및 운영 체제 수준의 메트릭을 수집하는 도구입니다. 
  - 시스템의 상태를 상세하게 모니터링

---

## 테스트 환경

- **AWS EC2**: 퍼블릭 IP가 할당된 Amazon Linux 2 인스턴스.
- **Prometheus 버전**: 2.55.1
- **Grafana 버전**: 11.1.3
- **Grafana Agent 버전**: 0.43.3
- **Node Exporter 버전**: 1.8.2

---

## 설치 및 구성
```bash
# 시스템 업데이트 및 필수 패키지 설치
sudo yum update -y
sudo yum install -y wget
```
### 1. Prometheus 
- [EC2_SD_CONFIG](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#ec2_sd_config)
```bash
# prometheus 사용자 및 그룹 생성
# 홈 디렉토리를 생성하지 않고, 로그인 불가능한 시스템 계정 'prometheus' 생성
sudo useradd --no-create-home --shell /bin/false prometheus

# 프로메테우스 설치
wget https://github.com/prometheus/prometheus/releases/download/v2.55.1/prometheus-2.55.1.linux-amd64.tar.gz
tar xvfz prometheus-*.tar.gz
sudo mv prometheus-2.55.1.linux-amd64 /usr/local/bin/prometheus

# 실행 파일 권한 설정
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chmod +x /usr/local/bin/prometheus

# 설정 파일 디렉토리 생성 및 작성
sudo mkdir -p /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus

# 데이터 저장 디렉토리 생성 및 권한 설정
sudo mkdir -p /var/lib/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus

# 구성 파일
cat <<EOF | sudo tee /etc/prometheus/prometheus.yaml
global:
scrape_interval: 60s
evaluation_interval: 60s

scrape_configs:
- job_name: 'prometheus'
    static_configs:
    - targets:
        - 'localhost:9090'

- job_name: 'grafana-agent'
    ec2_sd_configs:
    - region: "us-east-1" # AWS 리전
        port: 9100              # Node Exporter가 사용하는 포트
        filters:
        - name: "tag:Monitoring"
            values: ["true"]    # "Monitoring=true" 태그가 있는 인스턴스만 탐지

    relabel_configs:
    # EC2 Instance ID를 라벨로 추가
    - source_labels: [__meta_ec2_instance_id]
        target_label: instance_id

    # Private IP를 라벨로 추가
    - source_labels: [__meta_ec2_private_ip]
        target_label: private_ip

    # Public IP를 라벨로 추가
    - source_labels: [__meta_ec2_public_ip]
        target_label: public_ip

    # Custom Tag (예: Environment) 추가
    - source_labels: [__meta_ec2_tag_Environment]
        target_label: environment

    # AWS Account 정보 추가
    - source_labels: [__meta_ec2_owner_id]
        target_label: aws_account

    # EC2 Region 추가
    - source_labels: [__meta_ec2_region]
        target_label: region
EOF

# 서비스 파일 작성
sudo tee /etc/systemd/system/prometheus.service <<EOF
[Unit]
Description=Prometheus Monitoring System
Documentation=https://prometheus.io/docs/
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus/prometheus \
    --config.file=/etc/prometheus/prometheus.yaml \
    --storage.tsdb.path=/var/lib/prometheus \
    --web.console.templates=/usr/local/bin/prometheus/consoles \
    --web.console.libraries=/usr/local/bin/prometheus/console_libraries 
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# 서비스 등록 및 시작
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status -l prometheus
```

### 2. Grafana 
```bash
### 그라파나 설치
sudo yum install -y https://dl.grafana.com/enterprise/release/grafana-enterprise-11.1.3-1.x86_64.rpm
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

### 3. Grafana agent flow mode
- 그라파나 에이전트 Flow 모드
    - 내부 주소의 기본값은 `agent.internal:12345`입니다. 
    - 이 주소가 네트워크의 실제 대상과 충돌하는 경우 실행 명령에서 `--server.http.memory-addr` 플래그를 사용하여 고유한 주소로 변경하세요.
    - [Prometheus 메트릭 수집 및 전달](https://grafana.com/docs/agent/latest/flow/tasks/collect-prometheus-metrics/)
    - [components - prometheus.remote_write](https://grafana.com/docs/agent/latest/flow/reference/components/prometheus.remote_write/)
    - [EC2 메타데이터 관련 설정 - Service Discovery 개념](https://grafana.com/docs/agent/latest/flow/reference/components/discovery.ec2/)
        - [기본적으로 내보내는 필드](https://grafana.com/docs/agent/latest/flow/reference/components/discovery.ec2/#exported-fields)
        - [필터 블록](https://grafana.com/docs/agent/latest/flow/reference/components/discovery.ec2/#filter-block)
        - [EC2 API 필터](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/APIReference/API_Filter.html)
```bash
# 1. Grafana Agent Flow .rpm 파일 다운로드
sudo curl -LO https://github.com/grafana/agent/releases/download/v0.43.3/grafana-agent-flow-0.43.3-1.amd64.rpm

# 2. .rpm 파일 설치
sudo rpm -ivh grafana-agent-flow-0.43.3-1.amd64.rpm

# 3. grafana-agent-flow 그룹/사용자 생성
sudo groupadd grafana-agent-flow
# sudo useradd --no-create-home --shell /bin/false grafana-agent-flow  # 로그인 불가 및 홈 디렉토리 없는 사용자 생성
sudo usermod -aG grafana-agent-flow grafana-agent-flow

# 4. Grafana Agent Flow 설정 파일 생성 및 소유권 변경
sudo mkdir -p /etc/grafana-agent/  # 디렉토리 생성
sudo chown -R grafana-agent-flow:grafana-agent-flow /etc/grafana-agent  # 디렉토리 소유권 변경

# 설정 파일 생성
PromIP="172.31.17.179"  # Prometheus 서버 IP 주소

cat <<EOF | sudo tee /etc/grafana-agent/agent.river
logging {
    level = "info"
}

prometheus.remote_write "prom" {
endpoint {
    url = "http://$PromIP:9090/api/v1/write"
    metadata_config {
        send                 = true         // 메타데이터 전송 (기본값: true)
        send_interval        = "1m"         // 메타데이터 전송 간격 (기본값: 1분)
        max_samples_per_send = 2000         // 전송당 최대 샘플 수 (기본값: 2000)
        }
    }
}

discovery.ec2 "ec2" {
    region      = "us-east-1"  // AWS 리전
    port        = 9100         // Node Exporter 포트

    // filters 블록을 사용하여 현재 인스턴스만 필터링
    filter {
        name   = "tag:Monitoring"
        values = ["true"]
    }
}

prometheus.scrape "instances" {
    targets = discovery.ec2.ec2.targets
    forward_to = [prometheus.remote_write.prom.receiver]
}
EOF


# 설정 파일 소유권 변경
sudo chown grafana-agent-flow:grafana-agent-flow /etc/grafana-agent/agent.river

# 5. Grafana Agent Flow 서비스 파일 생성
sudo tee /etc/systemd/system/grafana-agent-flow.service <<EOF
[Unit]
Description=Grafana Agent Flow
After=network.target

[Service]
ExecStart=/usr/bin/grafana-agent-flow run /etc/grafana-agent/agent.river
Restart=on-failure
User=grafana-agent-flow
Group=grafana-agent-flow
WorkingDirectory=/etc/grafana-agent
EnvironmentFile=-/etc/default/grafana-agent-flow

[Install]
WantedBy=multi-user.target
EOF

# 6. 서비스 데몬 재로드 및 서비스 활성화
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-agent-flow
sudo systemctl status grafana-agent-flow
```

### 4. Node Exporter
- Node Exporter 
    - 메트릭 수집용, 그라파나 에이전트는 EC2 메타데이터 제공용도.
    - 그라파나 에이전트가 노드익스포터를 통해 메트릭을 수집하고 이를 메타데이터와 같이 가공하여 제공하는 것.
```bash
sudo wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
sudo tar xvfz node_exporter-*.tar.gz
sudo mv node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/
sudo useradd -rs /bin/false node_exporter

# 서비스 데몬 등록 및 서비스 등록
cat <<EOF | sudo tee /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter
[Install]
WantedBy=default.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
sudo systemctl status node_exporter
```