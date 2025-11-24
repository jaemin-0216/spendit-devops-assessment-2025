**a. 클라우드 아키텍처 다이어그램**
- 포함 요소: VPC, Subnet, Load Balancer, Container, Database, 모니터링 등
첨부한 Draw.io 확인 부탁 드립니다.

**b. 기술 선택 및 설계 근거**
- 왜 이러한 아키텍처를 설계했는지 논리적으로 설명
- 특정 AWS 서비스를 선택한 이유 (또는 GCP, Azure)
- 주어진 요구사항(고가용성, Auto-scaling, 비용 효율성)을 어떻게 충족시켰는지

- lambda를 선택한 이유
    - lambda는 트리거링 시 실행 되며, 실행 되지 않아 비용을 최적화 할 수 있습니다.
    - 평소 시간 당 100건, 피크 타임 시간 당 1000건 발생에도 대응이 가능하도록 sqs를 사용했습니다.
    - 이에 컨테이너 비용 및 장애에 대한 관리 포인트가 줄게 됩니다.
    - 추가적으로 트리거링으로만 실행되기에 고가용성, auto-scailing에 대해서 만족하는 아키텍처가 구성 됩니다.
    - Lambda 서비스는 수만건/일에 대해서는 문제 없이 처리가 가능한 서버리스 서비스이기에 선택하게 되었습니다.
    
- RDS proxy를 선택한 이유
    - lambda를 선택하면 DB connection pool에 대해 반드시 고려해야하며, 이에 대한 방안으로는 AWS에서 제공하는 서비스인    
      RDS-proxy를 사용 했습니다. RDS-proxy를 사용해도 ECS 사용보다 비용이 적으며, 다른 서비스에도 사용할 수 있으므로 추가적인 비용 절감효과도 가져갈 수 있기에 선택 했습니다.

- 고가용성, auto-scaling, 비용 효율성에 논리
    - 위 고가용성, auto-scaling은 lambda에 대한 설명으로 가능하다고 판단하여 비용 효율성에 대한 논리를 설명 합니다.
    - ECS와 lambda-RDS-proxy의 비용 차이
      - ECS - 총 $46/월
        - 평소 1tast 당 0.25 vcpu, 0.5GB memory 사용, 피크 기간 3일 간 4 tast 추가 사용 시 $21
        - 추가 ALB 비용 $25 
      - Lambda-RDS-proxy 사용 - 총 $24/월
        - Lambda는 100만 요청 건 까지 무료, ENI 사용 비용 약 $1, 실행 비용 약 $1
        - RDS-proxy 약 2 ACU 사용 가정 하여 약 $22 

**c. 운영 자동화 및 장애 대응 계획**
- **모니터링 전략**:
  - lambda로 실행하기에 주요 리소스 모니터링은 필요 없으며, ERROR에 대한 알람을 받을 수 있도록 cloudwatch alarm을 활용하여 아키텍처를 구성 했습니다.
  - Cloudwatch Alarm 아키텍처는 Metic, SNS, Lambda Slack의 순서로 특정 장애 로그가 Cloudwatch에 적재 시 Alarm을 발생해 시켜 SNS에 토픽에 적재하여 Lambda를 트리거 시켜 Slack 채널로 전송 시키는 구조를 구성했습니다. 

- **무중단 배포 전략**: Blue/Green, Rolling 등 구체적 방법
   - lambda로 구성 시 사용 이미지를 특정 버전으로 실행하게 구성하여 Blue/Green 배포를 할 수 있습니다.

- **장애 대응 시나리오**:
  - DB 연결 실패 시
    - 제공되는 health API가 DB연결까지 확인하고 있으므로, 해당 API의 응답을 이용하여 알람을 받아, RDS 장애 시  Multi-AZ 구성 또는 Prod 환경에 대해서는 미리 Multi-AZ 환경을 구성해 놓음으로 써 장애에 대한 대응을 합니다.
    - 추가 대응으로는 RDS Proxy에서 Multi-AZ Failover 시 30~60초 간 커넥션 끊김에 대해 버퍼링을 두어 재연결을 하여 서비스 중단에 대응 합니다.

  - 특정 API 서버 장애 시
    - Lambda는 트리거 되어 실행 시 장애가 발생한 순간, 새로운 환경이 실행 되어 대체 됩니다.
    - 실행 전 연계 시스템 장애과 연결 체크 로직을 추가 및 알람 시스템을 구현하여 연계 시스템 장애에 대해 알람을 받도록 합니다.

  - 트래픽 급증 시
    - Lambda, RDS-Proxy를 구성하여 트래픽 급증에 대해 대응 되도록 아키텍처 구성을 했습니다.

#### ▶ Option 2: 운영 시나리오 기반 심층 분석
  **3. 보안 강화 시나리오**
- **상황**: 보안 감사에서 IAM 권한이 과도하게 설정되어 있다는 지적
- **요구사항**:
  - 최소 권한 원칙(Principle of Least Privilege) 적용 계획
    - 팀 or 담당 업무에 대한 Role을 정의 합니다.
    - 해당 Role에 대한 최소 권한을 환경 별(Dev, Staging, Prod 등)로 정의 하여 권한을 부여하고 해당 Role IAM User에 부여합니다.
  
  - Secrets 관리 개선안
    - 보안 팀 또는 Devops 이외 사용자에 대해서는 Get, List의 권한만 허용
    - Secrets에 대해 등록 및 사용 권한에 대해서는 별도의 신청 시스템으로 신청을 받아 이력 관리 및 최소 권한에 대해 관리합니다.

  - 네트워크 보안 강화 방안
    - AWS의 경우 NACL은 최소한의 대역(가능한 Host 단위)으로 Inbound ACL을 정의 합니다.
      - 3tier(Public, Private, Data) 서브넷을 나누어 Public 서브넷에서는 Data 서브넷에 직접 접근하지 못하도록 하여 데이터 접근을 막습니다.
      - Private에서 접근하는 대역에 대해서만 Data 서브넷으로 접근하게 합니다.

    - Securty-Group
      - 1:1 매칭으로 인스턴스 별 Security Group을 가지게 하여 Security Group을 주체로 인바운드 인입을 허용시킵니다.
      - Outbound에 대해서도 가능한 외부와의 통신에 대해서는 특정 서버(Oubound Proxy 등)를 통하여 통신하도록 하여 이외 인스턴스에서는 외부와의 접속을 차단합니다.


  