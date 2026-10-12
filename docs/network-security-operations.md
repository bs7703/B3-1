# 네트워크·보안·운영 설계 설명

## 1. 네트워크 구성 및 라우팅

실습 구성 기준은 VPC `10.0.0.0/16`, Public Subnet `10.0.1.0/24`이다. EC2의 실제 프라이빗 IPv4 `10.0.1.242`는 이 서브넷 주소 범위에 속한다. VPC·IGW 식별정보는 콘솔 증빙으로 확인된다. Public Subnet과 라우팅 테이블의 개별 ID는 보관 자료에 기록되어 있지 않다.

| 구성 요소 | 값 | 역할 |
|---|---|---|
| VPC | my-vpc / vpc-08ee458d33b9e1a4c | 전체 가상 네트워크 공간 |
| Public Subnet 구성 CIDR | 10.0.1.0/24 | EC2를 배치하는 네트워크 구역 |
| 가용 영역 | ap-northeast-2a | EC2 배치 위치 |
| Internet Gateway | my-igw / igw-0c96bf4bc03058ae2 | VPC와 인터넷 연결 |
| EC2 프라이빗 IPv4 | 10.0.1.242 | VPC 내부 통신 주소 |
| EC2 퍼블릭 IPv4 | 3.38.244.69 | 실습 당시 외부 접속 주소 |

### Public Subnet 라우팅 테이블 구성 기준

| 목적지 CIDR | 대상 | 의미 |
|---|---|---|
| 10.0.0.0/16 | local | VPC 내부 목적지로 통신 전달 |
| **0.0.0.0/0** | **igw-0c96bf4bc03058ae2** | 더 구체적인 경로가 없는 인터넷 목적지로 통신 전달 |

라우팅 테이블은 Public Subnet에 연결되며, Internet Gateway는 VPC에 연결된다. IGW를 VPC에 연결하는 작업과 서브넷의 기본 경로를 설정하는 작업은 별개의 설정이다. 외부 접속의 구성 조건은 **EC2 퍼블릭 IPv4 + 해당 서브넷의 0.0.0.0/0→IGW 경로 + HTTP 80 허용 + Nginx 실행**이다.

외부 요청은 퍼블릭 IP를 통해 EC2에 도달하며, IGW가 퍼블릭 IPv4와 프라이빗 IPv4 사이의 주소 변환을 수행한다. EC2에서 시작한 외부 통신도 라우팅 테이블의 기본 경로를 이용한다. 실제 설치 기록에서 패키지 저장소 다운로드가 성공했고, 브라우저 화면에서 외부 요청과 응답이 확인되었다. 이 기록은 통신 성공을 보여주며 라우팅 설정 화면 자체를 대체하는 원본 설정 로그는 아니다.

[AWS 인터넷 게이트웨이 안내](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)

## 2. 보안 그룹의 허용·차단 기준

보안 그룹은 EC2의 네트워크 통신을 제어한다. 다음 표는 실습에 사용한 접근 제어 구성 기준을 정리한 것이다. 삭제 증빙에는 인바운드 규칙 2개가 표시되어 있지만, 각 규칙의 실제 소스 CIDR은 해당 화면에 표시되지 않는다.

| 방향 | 프로토콜·포트 | 소스 또는 목적지 기준 | 목적 |
|---|---|---|---|
| 인바운드 | TCP 80 / HTTP | 소스 0.0.0.0/0 | 누구나 웹페이지 열람 |
| 인바운드 | TCP 22 / SSH | 소스 학습자 공인 IPv4/32 | 학습자 컴퓨터의 관리 접속 |
| 인바운드 | 그 외 포트 | 허용 규칙 없음 | 사용하지 않는 서비스의 접근 차단 |
| 아웃바운드 | 패키지 설치 등 외부 통신 | 외부 저장소 통신 허용 | 설치·업데이트 및 외부 통신 |

`/32`는 IPv4 주소 한 개를 뜻한다. 예를 들어 `203.0.113.10/32`는 한 컴퓨터의 공인 IP를 허용하는 표기 예시이며 실제 학습자 IP가 아니다.

보안 그룹은 허용 규칙을 사용하는 방식이다. 별도의 Deny 규칙을 추가하는 대신 필요한 통신만 허용하고, 허용되지 않은 신규 인바운드 통신은 차단한다. 여러 보안 그룹이 연결되어 있으면 각 그룹의 허용 규칙을 함께 고려한다.

### 0.0.0.0/0의 용도와 위험

`0.0.0.0/0` 자체가 위험한 설정인 것은 아니다. 라우팅 테이블에서는 인터넷 기본 경로의 목적지 범위로 사용한다. 보안 그룹에서는 모든 IPv4 주소를 의미하므로 적용 포트에 따라 위험이 달라진다.

- HTTP 80 전체 허용: 공개 웹 서비스 제공 목적에 맞는 설정이다.
- SSH 22 전체 허용: 모든 외부 주소에서 인증 시도가 가능해져 스캔·무차별 로그인 시도에 노출된다. 개인 IP/32로 제한한다.
- 모든 포트 전체 허용: 불필요한 서비스까지 외부에 노출하므로 실습 구성에서 제외한다.

Security Group은 상태를 추적하므로 허용된 연결의 응답 통신을 위해 별도의 반대 방향 인바운드 규칙을 추가하지 않는다.

[AWS 보안 그룹 안내](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)

## 3. IAM과 Security Group의 차이

| 구분 | IAM | Security Group |
|---|---|---|
| 제어 대상 | AWS API·콘솔 작업 권한 | EC2 네트워크 통신 |
| 판단 기준 | 사용자/Role, Action, Resource, Condition | 프로토콜, 포트, 소스·목적지 |
| 예시 | EC2 생성·종료 권한 | SSH 22번을 개인 IP에 허용 |
| 제한 효과 | 허용되지 않은 AWS 작업을 막음 | 허용되지 않은 접속을 막음 |

IAM 사용자는 AWS 콘솔 관리 계정이고, SSH 명령의 `ubuntu`는 EC2 운영체제 사용자이다. Nginx 패키지 설치는 Linux의 `sudo` 권한으로 실행되며 IAM 정책이 직접 결정하는 작업이 아니다.

### 실습용 최소권한 설계

권한 설계 범위를 실습용 EC2와 VPC 구성 작업으로 한정한다. 아래는 필요한 작업과 대응 권한 예시이며, 실제 연결된 정책의 내보내기 자료와는 구분된다.

| 작업 | IAM Action 예시 |
|---|---|
| 리소스 조회 | ec2:DescribeInstances, DescribeVpcs, DescribeSubnets, DescribeRouteTables, DescribeSecurityGroups, DescribeVolumes |
| EC2 생성·종료 | ec2:RunInstances, TerminateInstances |
| VPC·서브넷 관리 | ec2:CreateVpc, DeleteVpc, CreateSubnet, DeleteSubnet |
| IGW 관리 | ec2:CreateInternetGateway, AttachInternetGateway, DetachInternetGateway, DeleteInternetGateway |
| 라우팅 관리 | ec2:CreateRouteTable, CreateRoute, AssociateRouteTable, DisassociateRouteTable, DeleteRouteTable |
| 보안 그룹 관리 | ec2:CreateSecurityGroup, AuthorizeSecurityGroupIngress, RevokeSecurityGroupIngress, DeleteSecurityGroup |
| 태그·키 페어 | ec2:CreateTags, CreateKeyPair, DeleteKeyPair |
| 잔여 볼륨 정리 | ec2:DeleteVolume |
| 할당한 EIP 정리 | ec2:DisassociateAddress, ReleaseAddress |

Action 이름은 모두 `ec2:` 접두사를 사용한다. VPC 구성도 EC2 API 권한 영역에 포함된다. 허용 범위는 서울 리전 `ap-northeast-2`로 제한하고, 지원되는 작업은 실습 리소스 ARN으로 제한한다. `Describe*` 조회 작업은 리소스 단위 제한이 지원되지 않아 Resource `*`가 필요하다. 실행 권한까지 Resource `*`로 허용하면 서울 리전의 다른 리소스도 영향을 받을 수 있으므로 동작 범위와 리소스 범위를 구분한다.

### 권한의 점진 확대 원칙

1. 실습 작업 목록을 먼저 작성하고 조회·생성·설정·삭제에 필요한 권한을 선정한다.
2. 지정 사용자/Role로 해당 작업을 실행해 허용 여부를 확인한다.
3. AccessDenied 발생 시 거부된 Action·Resource와 호출 목적을 확인한다.
4. 필요한 권한만 추가하고 같은 작업을 재검증한다.
5. 임시 권한과 불필요한 정책을 제거하고 연결된 정책 전체를 검토한다.

AdministratorAccess와 실습에 무관한 S3/RDS 관리 권한은 실습용 권한 설계에서 제외한다. 정책을 추가할 때 기존 관리자 정책이 남아 있는지도 확인한다. 최소권한 설계 설명과 실제 정책 적용 증빙은 서로 다른 확인 항목이다.

[AWS IAM 최소권한 원칙](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html), [EC2 콘솔 권한 예시](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-policies-ec2-console.html)

## 4. 리소스 명명 및 태그 규칙

식별 이름은 공통 접두사 `my-`와 리소스 역할을 결합한 형식으로 정리하였다.

| 리소스 | 관찰된 이름 | 의미 |
|---|---|---|
| EC2 | my-web | 웹 서버 |
| VPC | my-vpc | 실습 네트워크 |
| IGW | my-igw | 인터넷 게이트웨이 |
| 보안 그룹 | my-web-sg | 웹 서버 접근 제어 |
| 키 페어 | my-key | SSH 인증 키 |

태그 표기 예시는 `Name:my-web`, `Name:my-vpc`, `Name:my-igw`이다. 콘솔의 Name 열에서 이름이 확인된다. 보안 그룹 이름과 키 페어 이름은 자체 이름 필드도 있으므로 이름이 보인다는 이유만으로 각각 별도의 Name 태그까지 있다고 단정하지 않는다. Project·Owner·Environment 태그를 추가했다고 기록하지 않았다.

## 5. 확장·병목·비용 대응 설계

본 실습의 구현 범위는 EC2 1대의 HTTP 웹 서비스이다. 다음은 규모가 커질 때 적용하는 운영 설계이며 ALB나 자동 확장을 구현했다는 결과가 아니다.

| 지표 또는 상황 | 판단 및 대응 |
|---|---|
| CPUUtilization 지속 상승 | 요청 증가·프로세스 부하를 분석하고 서버 크기 변경 또는 수평 확장 판단 |
| t3 CPU 크레딧 감소 | CPUCreditBalance 및 버스트 관련 지표로 지속 부하 확인 |
| NetworkIn / NetworkOut 증가 | 전송량, 응답 크기와 트래픽 원인 분석 |
| StatusCheckFailed | 호스트 또는 인스턴스 상태 장애 구분 |
| Nginx 오류 및 지연 | access.log·error.log로 상태 코드·요청 패턴 분석; 시간 필드를 설정하면 지연 추적 가능 |
| 여러 서버에 분산 또는 가용성 확보 필요 | ALB + 복수 EC2/Auto Scaling 설계 검토 |

ALB는 서버 여러 대로 HTTP 요청을 분산하고 상태 검사를 수행하는 용도이다. 단일 서버의 단순 실습에는 필요하지 않으며 추가 리소스와 비용이 생기므로 요구사항에 맞춰 선택한다.

비용 추적은 Billing/Cost Explorer에서 서비스·리전별 사용 내역을 비교하고, AWS Budgets로 예산 알림을 설정하는 방식이다. EC2 실행, EBS·스냅샷, 퍼블릭 IPv4, NAT Gateway, ALB와 데이터 전송 등이 점검 대상이다. 예상 밖의 사용량이 발견되면 해당 리전의 잔여 리소스와 실행 시간을 확인한다. 알림 설정 자체가 리소스를 자동으로 삭제하는 것은 아니다.

정리 후 비용이 0이라고 단정하지 않는다. 과거 사용 내역은 남고 반영 지연도 있을 수 있으므로 리소스 상태와 비용 내역을 구분한다. 실제 종료·삭제 결과는 [정리 기록](cleanup-checklist.md)에 제시하였다.

[AWS EC2 모니터링](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-cloudwatch.html), [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html)
