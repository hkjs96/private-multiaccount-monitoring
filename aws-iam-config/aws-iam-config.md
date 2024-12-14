# AWS Account IAM Configuration Guide

이 문서는 프라이빗 멀티 어카운트 AWS 환경에서 Grafana, Amazon Managed Prometheus (AMP), CloudWatch 및 Grafana Agent를 활용한 모니터링 시스템 구축을 위한 IAM(Identity and Access Management) 설정 방법을 안내합니다.

## 목차
- [1. 계정 구성 개요](#1-계정-구성-개요)

- [2. AMP 계정 Role 설정](#2-amp-계정-role-설정)
  - [2.1 AMP_Assume_Role](#21-amp_assume_role)
    - [Policy: AMP_Assume_Role_Policy](#policy-amp_assume_role_policy)
    - [Trust Relationship](#trust-relationship)

- [3. Target 계정 Role 설정](#3-target-계정-role-설정)
  - [3.1 Target_Agent_EC2_Instance_Profile](#31-target_agent_ec2_instance_profile)
    - [Policy: Target_Agent_AMP_Policy](#policy-target_agent_amp_policy)
    - [Policy: EC2_Metadata_Policy](#policy-ec2_metadata_policy)
  - [3.2 Cloudwatch_Assume_Role](#32-cloudwatch_assume_role)
    - [Policy: Cloudwatch_Log_To_Grafana_Policy](#policy-cloudwatch_log_to_grafana_policy)
    - [Policy: Cloudwatch_Metric_To_Grafana_Policy](#policy-cloudwatch_metric_to_grafana_policy)
    - [Trust Relationship](#trust-relationship-1)

- [4. Grafana 계정 Role 설정](#4-grafana-계정-role-설정)
  - [4.1 Grafana_Server_EC2_Instance_Profile](#41-grafana_server_ec2_instance_profile)
    - [Policy: Grafana_Server_AMP_Policy](#policy-grafana_server_amp_policy)
    - [Policy: EC2_Metadata_Policy](#policy-ec2_metadata_policy-1)

- [참고 사항](#참고-사항)


## 1. 계정 구성 개요

모니터링 시스템은 다음과 같은 AWS 계정들로 구성됩니다:
- AMP 계정: Amazon Managed Prometheus 워크스페이스 호스팅
- Target 계정: 모니터링 대상 리소스 호스팅
- Grafana 계정: Grafana 서버 호스팅

## 2. AMP 계정 Role 설정

### 2.1 AMP_Assume_Role

- **Role Name**: `AMP_Assume_Role`
- **Purpose**: AMP 메트릭 수집 및 쿼리 조회
- **신뢰 관계**: 
  - Grafana 서버 계정 - Role
  - Target 서버 계정 - Role

> ❗ 신뢰 관계를 설정하기 위해서는 [3.1 Target Agent - Role](#31-target-agent---role) 및 [4.1 Grafana Server - Role](#41-grafana_server---role) 설정이 선행되어야 합니다.

#### Policy: AMP_Assume_Role_Policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "aps:DescribeWorkspace",
                "aps:RemoteWrite",
                "aps:QueryMetrics",
                "aps:GetLabels",
                "aps:GetSeries",
                "aps:GetMetricMetadata"
            ],
            "Resource": "arn:aws:aps:<region>:<AMP_account_id>:workspace/<workspace_id>"
        }
    ]
}
```

> ❗ AMP Workspace 생성 시 받은 정보를 참고하여 `<region>`, `<AMP_account_id>`, `<workspace_id>`를 입력하세요.

#### Trust Relationship
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::<Target_Agent_account_id>:role/Target_Agent_EC2_Instance_Profile",
                    "arn:aws:iam::<Grafana_Server_account_id>:role/Grafana_Server_EC2_Instance_Profile"
                ]
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

## 3. Target 계정 Role 설정

### 3.1 Target_Agent_EC2_Instance_Profile

- **Role Name**: `Target_Agent_EC2_Instance_Profile`
- **Required Policies**:
  - AWS 관리형 정책:
    - `CloudWatchAgentServerPolicy`
  - 사용자 정의 정책:
    - `Target_Agent_AMP_Policy`
    - `EC2_Metadata_Policy`
- **신뢰 관계**: EC2 - Role

#### Policy: Target_Agent_AMP_Policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sts:AssumeRole"
            ],
            "Resource": [
                "arn:aws:iam::<AMP_account_id>:role/<AMP_Assume_Role>"
            ]
        }
    ]
}
```

#### Policy: EC2_Metadata_Policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeTags"
            ],
            "Resource": "*"
        }
    ]
}
```

### 3.2 Cloudwatch_Assume_Role

- **Role Name**: `Cloudwatch_Assume_Role`
- **Purpose**: CloudWatch 메트릭 및 로그 수집
- **신뢰 관계**: Grafana 서버 계정 - Role

#### Policy: Cloudwatch_Log_To_Grafana_Policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowReadingLogsFromCloudWatch",
            "Effect": "Allow",
            "Action": [
                "logs:DescribeLogGroups",
                "logs:GetLogGroupFields",
                "logs:StartQuery",
                "logs:StopQuery",
                "logs:GetQueryResults",
                "logs:GetLogEvents"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowReadingTagsInstancesRegionsFromEC2",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeTags",
                "ec2:DescribeInstances",
                "ec2:DescribeRegions"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowReadingResourcesForTags",
            "Effect": "Allow",
            "Action": "tag:GetResources",
            "Resource": "*"
        }
    ]
}
```

#### Policy: Cloudwatch_Metric_To_Grafana_Policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowReadingMetricsFromCloudWatch",
            "Effect": "Allow",
            "Action": [
                "cloudwatch:DescribeAlarmsForMetric",
                "cloudwatch:DescribeAlarmHistory",
                "cloudwatch:DescribeAlarms",
                "cloudwatch:ListMetrics",
                "cloudwatch:GetMetricData",
                "cloudwatch:GetInsightRuleReport"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowReadingTagsInstancesRegionsFromEC2",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeTags",
                "ec2:DescribeInstances",
                "ec2:DescribeRegions"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowReadingResourcesForTags",
            "Effect": "Allow",
            "Action": "tag:GetResources",
            "Resource": "*"
        },
        {
            "Sid": "AllowReadingResourceMetricsFromPerformanceInsights",
            "Effect": "Allow",
            "Action": "pi:GetResourceMetrics",
            "Resource": "*"
        }
    ]
}
```

#### Trust Relationship
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "TrustRelationShipGrafanaRole",
            "Effect": "Allow",
            "Principal": {
                "AWS": [
                    "arn:aws:iam::<Grafana_Server_account_id>:role/Grafana_Server_EC2_Instance_Profile"
                ]
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

## 4. Grafana 계정 Role 설정

### 4.1 Grafana_Server_EC2_Instance_Profile

- **Role Name**: `Grafana_Server_EC2_Instance_Profile`
- **Required Policies**:
  - 사용자 정의 정책:
    - `Grafana_Server_AMP_Policy`
    - `EC2_Metadata_Policy`
- **신뢰 관계**: EC2 - Role

#### Policy: Grafana_Server_AMP_Policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sts:AssumeRole"
            ],
            "Resource": [
                "arn:aws:iam::<AMP_account_id>:role/<AMP_Assume_Role>",
                "arn:aws:iam::<Target_Agent_account_id>:role/<Cloudwatch_Assume_Role>"
            ]
        }
    ]
}
```

#### Policy: EC2_Metadata_Policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeTags"
            ],
            "Resource": "*"
        }
    ]
}
```

## 참고 사항

1. **신뢰 관계 설정**
   - 각 역할 간의 신뢰 관계는 정확한 ARN을 사용하여 설정해야 합니다
   - 잘못된 신뢰 관계 설정은 권한 위임 실패의 주요 원인이 됩니다

2. **최소 권한 원칙**
   - 각 정책은 필요한 최소한의 권한만을 부여하도록 설계되었습니다
   - 필요에 따라 추가 권한을 신중하게 고려하여 부여할 수 있습니다

3. **변수 대체**
   - 문서 내의 모든 `<variable>` 형태의 플레이스홀더는 실제 환경 값으로 대체해야 합니다
   - 예: `<region>`, `<AMP_account_id>`, `<workspace_id>` 등

추가 참고 자료:
- [Amazon CloudWatch data source](https://grafana.com/docs/grafana/latest/datasources/aws-cloudwatch/)
