Journey of API Gateway migration
========================


1st Gen
----------
##### Aim problem
1. Reduce TCO of Tyk
2. Compability of URL Proxying

##### Reduce TCO of Tyk 
Early 2023, We decide out api gateway change from tyk to kong, reason why tky's price is to high comapre with last year. 
Tyk Gateway is open source and can use freely until now. they change pricing model for enterprice customer and license of exclude gateway.
2022년과 2023년 사이에 Tyk는 가격 정책을 변경하면서 기존보다 2배이상의 연간 라이선스 비용을 요구하려 했습니다. 
그래서 저희는 tyk를 대체할 수 있는 다른 API Gateway를 찾았고 AWS API Gateway와 KrakenD와 kong을 비교했고, kong을 선택하게 되었는데 이유는 Kong과 Konga를 \
조합하면 적당한 GUI 환경을 사용하면서 Tyk 대비 0원에 가까운 비용으로 유지를 할 수 있다는 계산과 API  인건비는 tyk이든 kong이든 동일하기 때문입니다.

tyk 다이어그램
![tyk_v1.drawio.png](tyk_v1.drawio.png)
Tyk를 사용할 때는 mongodb를 저장소로 사용하는데 여기에 설정이 된 정보를  tyk pump를 이용해서 AWS Elasticache for REDIS (REDIS)에 저장합니다. 
이 REDIS에 저장된 설정을 Tyk Gateway들이 읽어들인 후 서비스 하게 됩니다. 
Tyk는 전면에 나서는 Gateway , 서버를 설정하고 여러 지표를 볼 수 있는 Dashboard, Dashboard와 Gateway 사이 매개체로 쓰이는  Redis에 업데이트를 하는 pump로 
구성이되어 있습니다. 데이터 저장소로는 MongoDB를 사용합니다. 
우린 6대의  m5.xlarge 로 구성된 gateway와 m5.xlarge로 Tyk Dashboard, MongoDB, Tyk Pump, 그리고 m5.xlarge 스펙으로 HA 구성이 된 AWS Elasticache for Redis 를 
사용해서 구성했습니다. 
인스턴스와 Redis를 기준으로 계산되는 예상 비용은 연간 약 $18400($1,533x12) 입니다. 

 | resource | type | Qty | Unit price/month | amount / month |
 | --- | --- | --- | --- | --- |
 | EC2              | m5.xlarge          | 9 | $107 |  $963 |
 | ElastiCache | cache.m5.xlarge | 3 | $190 | $570 |
 |계|||| $1,533|

 (표 1.Tyk 사용시 예상 인스턴스 비용)
 
 
kong 다이어그램
![1st kong diagram](kong_v1.drawio.png)
Tyk에서 Kong 이전을 결정하고 제일먼저 한 일은 Authentication에 대한 정책을 변경했습니다.
 트래픽의 99%가 내부에서 발생하는 트래픽이었기 때문에 별도의 인증을 거치지 않고 방화벽에서 통신을 제어하는 것으로 인증을 대신했습니다.
때문에  Cunsumer와  Credentials 에 대한 부담이 없는 상태로 API Proxy 규칙을 이전하는 코드를 개발 했습니다.

 | resource | type | Qty | Unit price/month | amount / month |
 | --- | --- | --- | --- | --- |
 | EC2              |m5..large          | 5 | $38 | $186|
| RDS               | db.t3.small      | 1 |  $32  | $32|
| 계||||$218|  

하지만 이런 작업의 결과 인프라 비용은 월간 $1,500 에서 $220로 Tyk 대비 -85% 비용 절감 및 연간 $50,000의 라이선스 비용을 아끼게 되었습니다.
그리고 기존 Tyk 장애시엔  Tyk에 대한 모니터링 지점이 dashboard, mongodb, pump, redis,gateway로 총 5곳이었지만 kong으로 전환하면서 
kong gateway, postgresql, 두개로 줄어들어서 모니터링 지점이 줄었고, 장애 시 살펴봐야 하는 요소도 줄었습니다. 
목적이 비용 줄이기였던 만큼 해당 목적은 달성한 것으로 평가합니다. 
tyk의 총 운영비용이 연간 $70,000이었던 것을 연간 $2,600 정도로 절감시켜서 96%의 비용을 절감했습니다. 


2nd Gen
-----------
#### Problem
1. Security Issue (using too old version)
2. Availability Issue (stand alone configuration)
![2nd kong diagram](kong_v2.drawio.png)
