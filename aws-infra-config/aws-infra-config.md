# AWS 계정 인프라 구성 가이드

이 문서는 AWS 계정에서 프라이빗 네트워크 환경의 모니터링 인프라를 설정하는 방법을 설명합니다.

## 목차
- [1. Amazon Managed Prometheus (AMP) 워크스페이스 구성](#1-amazon-managed-prometheus-amp-워크스페이스-구성)
- [2. VPC 엔드포인트 구성](#2-vpc-엔드포인트-구성)
- [3. 네트워크 연결 구성](#3-네트워크-연결-구성)
- [4. 보안 그룹 설정](#4-보안-그룹-설정)

## 1. Amazon Managed Prometheus (AMP) 워크스페이스 구성

### 1.1 AMP 워크스페이스 생성
1. AMP Account 의 AWS Management Console에서 Amazon Managed Service for Prometheus 서비스로 이동
2. '워크스페이스 생성' 버튼 클릭
3. 다음 설정으로 워크스페이스 구성:
   - 워크스페이스 이름: `AMP_WorkSpace` (또는 원하는 이름)
   - 별칭: 원하는 별칭 입력
   - 태그: 필요한 태그 추가
4. '워크스페이스 생성' 버튼 클릭하여 완료

## 2. VPC 엔드포인트 구성

### 2.1 Target 계정 VPC 엔드포인트
메트릭을 수집하는 계정의 VPC에 다음 엔드포인트를 생성:

1. AMP 관련:
   ```
   com.amazonaws.<region>.aps-workspaces  # AMP 워크스페이스와 통신
   com.amazonaws.<region>.aps             # 메트릭 수집
   ```

2. IAM 관련:
   ```
   com.amazonaws.<region>.sts  # 크로스 어카운트 IAM Role 사용
   ```

### 2.2 Grafana 서버 계정 VPC 엔드포인트
Grafana가 설치된 계정의 VPC에 다음 엔드포인트를 생성:

1. AMP 관련:
   ```
   com.amazonaws.<region>.aps-workspaces  # AMP 워크스페이스 접근
   com.amazonaws.<region>.aps             # 메트릭 조회
   ```

2. CloudWatch 관련:
   ```
   com.amazonaws.<region>.monitoring      # CloudWatch 메트릭 조회
   com.amazonaws.<region>.logs           # CloudWatch 로그 조회
   ```

3. IAM 관련:
   ```
   com.amazonaws.<region>.sts            # AssumeRole 등 IAM 역할 수임
   ```

### 2.3 AMP 계정
AMP가 설치된 계정에는 별도의 VPC 엔드포인트 구성이 필요하지 않습니다.

### 2.4 엔드포인트 설정 방법
1. AWS Console의 VPC 서비스로 이동
2. '엔드포인트' 메뉴 선택
3. 각 서비스별로:
   - '엔드포인트 생성' 클릭
   - 서비스 카테고리에서 해당 서비스 선택
   - VPC 선택
   - 프라이빗 서브넷이 있는 가용영역 선택
   - 보안 그룹 연결
   - Route Table 선택 확인

## 3. 네트워크 연결 구성

### 3.1 Transit Gateway 구성 (멀티 어카운트 환경)
1. Transit Gateway 생성:
   - 이름 태그 지정
   - ASN 번호 설정
   - DNS 지원 활성화
   - VPC 연결 자동 수락 설정

2. Transit Gateway VPC 연결:
   - 각 계정의 VPC 연결 생성
   - 적절한 서브넷 선택
   - 라우팅 테이블 연결

### 3.2 라우팅 테이블 설정
1. 프라이빗 서브넷의 라우팅 테이블에 다음 추가:
   ```
   대상: 타 계정 VPC CIDR
   타겟: Transit Gateway ID
   ```

## 4. 보안 그룹 설정

### 4.1 VPC 엔드포인트 보안 그룹
```
인바운드 규칙:
- TCP 443: HTTPS (VPC 내부 통신용)
  소스: VPC CIDR

아웃바운드 규칙:
- 기본값 유지 (전체 허용)
```

### 4.2 Grafana 서버 보안 그룹
```
인바운드 규칙:
- TCP 3000: Grafana Web Interface
- TCP 22: SSH (관리용)

아웃바운드 규칙:
- 기본값 또는 필요한 경우 VPC 엔드포인트로의 통신 허용
```

### 4.3 모니터링 대상 보안 그룹
```
인바운드 규칙:
- TCP 9090: Prometheus
- TCP 9100: Node Exporter
- TCP 9091: Pushgateway (사용시)

아웃바운드 규칙:
- AMP 워크스페이스 엔드포인트로의 통신 허용
```

### 4.4 보안 그룹 생성 방법
1. VPC 콘솔에서 '보안 그룹' 메뉴 선택
2. '보안 그룹 생성' 클릭
3. 이름 및 설명 입력
4. VPC 선택
5. 인바운드 및 아웃바운드 규칙 구성:
   - VPC 엔드포인트 보안 그룹의 경우:
     - 인바운드 규칙에 TCP 443 포트만 허용
     - 소스는 VPC CIDR 범위로 제한
6. 태그 추가 (선택사항)
7. 생성 완료

