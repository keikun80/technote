Jenkins를 이용한 배치 스케줄러 구축
========================

#### 배경 
사내 통합 스케줄러 서버의 서비스 종료에 따른 신규 스케줄러 구축 소요 제기되었고, 이 때 각 애자일 그룹별 스케줄러 분리 요청에 따른 서버 분리를 요청 받게 되었습니다. 
스케줄러를 3개로 분리(3개의 애자일 그룹)하게 되면 하나의 스케줄러 장애가 3개의 그룹에 영향을 끼치는 것을 방지 할 수 있기 때문에 수용했습니다. 
그리고 RTTM, RTO 개선 (기존에는 사람이 장애를 인지한 시점에서 담당자에게 연락이 취해진 후 담당자에 의해 Standby서버로 스케줄링 작업 개시)요건이 
있었는데 위에 말했 듯 인력에 의한 서버 이전과 다시 failback하는 과정에서도 Standby에서 실행된 모든 기록을 Active 서버로 옮겨야 했는데, 
이 때는 유지보수 업체의 인력과 스케줄 조정이 필수였기 때문에 RTTM이 하루나 이틀씩 걸리는 경우도 있었고, RTO의 경우에는 Standby에서 Active 로 내용을 
동기화를 시작할 때의 데이터로 복원되고 이때가 RTO, 결정적으로 복원을 하는 과정에서도 모든 스케줄러가 중지되어야 한다는 문제가 있었습니다. 

#### 문제
1. 한대의 스케줄러에서 장애 발생 시 3개 애자일 그룹의 배치 작업이 중지 된다. 
2. HA 구성이 수동으로 되어 있다. 
3. RTTM, RTO 그밖에 모니터링 지표가 설정되어 있지 않다. 

#### 설계 
각 그룹별로 스케줄러를 분리하고 RTTM, RTO를 개선하고 장애시간이 길어지지 않도록 설계하는데 중점을 뒀습니다.
기본 요구
1. AWS ASG를 이용해서 Auto Healing 구현
2. AWS EFS를 이용해서 각 인스턴스가 사용하는 Permanent Storage를 제공
3. ASG에 EC2 ,ELB 헬스체크를 이용해서 스케줄러 자체의 상태를 모니터링 하도록 구성 
4. AWS Cloudwatch Alarm을 이용해서 이상 상태 발생 시 담당자에게 알람 전송

![설계 안](scheduler_to_be_2.drawio.png "그림1 설계")

##### 컨테이너 설치
ASG는 Launch Template를 기반으로 만들어지기 때문에 Launch Template에서 사용할 AMI를 만들어야 합니다. 
AMI는 AWS가 제공하는 것을 사용하고 거기에 맞춰서 Userdata를 구성할 수도 있지만 
이 AMI는 스케줄러 전용으로 사용 될 것이기 때문에 전 docker image부터 만들 계획입니다. 
EC2 인스턴스에서 docker와 docker-compose를 사용할 수 있도록 구성합니다. ( appe계정을 이용해서 서버상의 application을 실행시킨다)
```
# Installing Docker
sudo dnf install -y docker
sudo usermode -aG docker appe
sudo systemctl enable --now docker 
```
```
# Installing docker-compose
sudo mkdir -p /usr/local/lib/docker/cli-plugins/
sudo curl -SL "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-$(uname -m)" -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
```
docker, docker-compose를 설치 했으면 적당한 jenkins 버전을 골라서 각자의 환경에 맞는 세팅을 하고 docker image로 build 하면 됩니다. 
사용할 플러그인이나 OS업데이트 정보 등을 구성하고 Dockerfile을 만들어서 docker image를 build 합니다. 
```
#Dockerfile
FROM jenkins/jenkins:2.530.2-jdk21
USER root
RUN apt-get update && apt-get install -y lsb-release
RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
  https://download.docker.com/linux/debian/gpg
RUN echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
  https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
RUN apt-get update && apt-get install -y docker-ce-cli  jq
RUN apt-get install -y graphviz && apt-get install -y fonts-nanum-* && fc-cache -f -v
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean docker-workflow json-path-api"
RUN jenkins-plugin-cli --plugins "depgraph-view:1.0.5"
RUN jenkins-plugin-cli --plugins "extra-columns:1.27"
RUN jenkins-plugin-cli --plugins "extended-timer-trigger:30.vb_fb_7092cccfa_"
RUN jenkins-plugin-cli --plugins "disable-job-button:1.3.vf55949267366"
RUN jenkins-plugin-cli --plugins "jobConfigHistory:1356.ve360da_6c523a_"
```
jenkins 이미지에 추가로 plugin을 설치하는 명령어를 추가한 Dockerfile을 만들고 이것을 기본 이미지로 사용하기로 결정한 다음 
```
sudo docker image build -t jenkins-scheduler . 
```
이렇게 해서 jenkins-scheduler 이미지를 생성한다. 

이것을 AWS의 경우엔  ECR 혹은 다른 docker registry에 업로드 한 후 pull해도 되지만 우린 닫힌 환경에서 정해진 ami로만 할거니까 업로드 하지 않도록 한다. 

jenkins가 완성이 되었으면 모니터링을 위한 나머지 요소들까지 포함한 docker-compose.yaml을 만들어야 한다. 
```
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    user: "2000:2000"
    environment:
      - TZ=Asia/Seoul
    restart: always
    networks:
      - scheduler-net
    ports:
      - 9090:9090
    volumes:
      - ./prometheus/data:/prometheus
      - ./prometheus/etc:/etc/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.enable-lifecycle'
    healthcheck:
      test: ["CMD", "wget", "--tries=1", "--spider", "http://localhost:9090/-/healthy"]
      interval: 10s
      timeout: 5s
      retries: 3
  grafana:
    image: grafana/grafana:latest
    user: "2000:2000"
    container_name: grafana
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    restart: always
    ports:
      - 3000:3000
    networks:
      - scheduler-net
    volumes:
      - ./grafana/data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=true
      - TZ=Asia/Seoul
  jenkins:
    image: jenkins-scheduler:latest
    container_name: scheduler
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    restart: always
    user: root
    environment:
      - TZ=Asia/Seoul
    ports:
      - "8080:8080"
      - "50000:50000"
    networks:
      - scheduler-net
    volumes:
      - ./jenkins_home:/var/jenkins_home
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - 9100:9100
    networks:
      - scheduler-net
  cadvisor:
    container_name: cadvisor
    image: gcr.io/cadvisor/cadvisor:latest
    networks:
      - scheduler-net
    ports:
      - 9080:8080
    volumes:
      - "/:/rootfs"
      - "/var/run:/var/run"
      - "/sys:/sys"
      - "/var/lib/docker/:/var/lib/docker"
      - "/dev/disk/:/dev/disk"
    devices:
      - "/dev/kmsg"

networks:
  scheduler-net:
    driver: bridge
```
여기까지 설정하면 이제 docker-compose start 혹은 up 명령으로 모든 컨테이너를 구동시키고 데이터가 생성되는 것을 볼 수 있다. 

* grafana docker : http://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/
* prometheus docker : https://prometheus.io/docs/prometheus/latest/installation/
* node exporter : https://hub.docker.com/r/prom/node-exporter
* ref : https://medium.com/@sohammohite/docker-container-monitoring-with-cadvisor-prometheus-and-grafana-using-docker-compose-b47ec78efbc 

##### 2. AWS EFS를 이용해서 각 인스턴스가 사용하는 Permanent Storage를 제공
AWS EFS(Elastic File System)은 NFSv4를 기반으로 하는 네트워크 파일 시스템이다. 
젠킨스나 프로메테우스등이 파일을 기반으로 서비스 되기 때문에 ASG에 의해서 EC2 인스턴스가 변경되는 경우에도 모든 파일이 지속적으로 
인계되어야 한다. 이 경우 EBS를 사용하면 비용은 절감되지만, 파일을 동기화 시키는데 어려움이 따른다. 이 경우 사용하는 것이 EFS이다. 
AWS EFS로 각 서버가 사용할 파일 시스템을 만들어서 EC2인스턴스에 연결해주면 작업은 끝난다. 
우리는 이것을 구성할 때 스케줄러에 필요한 모든 파일을 이 EFS에 저장했다. 
Dockerfile부터 docker-compose용 yaml과 jenkins_home, prometheus_home 등을 모두 하나의 EFS에 저장하고 EC2는 이 EFS에 있는 명령만 userdata를 통해서 
실행시키도록 구성했다. 
```
[root@scheduler scheduler]# ls -al
total 2076
-rw-r--r--.  1 appexec appexec       2537 Jan 29 12:06 compose.yml
drwxr-xr-x.  3 appexec appexec    6144 Jan 29 07:25 grafana
drwxr-xr-x.  2 appexec appexec    6144 Jan 29 07:56 jenkins
drwxr-xr-x. 16 appexec appexec    6144 Feb 23 05:32 jenkins_home
drwxr-xr-x.  4 appexec appexec    6144 Jan 29 07:26 prometheus
```
* AWS EFS 파일 시스템 생성 : https://docs.aws.amazon.com/ko_kr/efs/latest/ug/creating-using-create-fs.html
##### 3. ASG에 EC2 ,ELB 헬스체크를 이용해서 스케줄러 자체의 상태를 모니터링 하도록 구성 
AWS ALB에서 health check 옵션을 다음과 같이 세팅 한다. 
젠킨스 타겟그룹,  그라파나 타겟그룹, 프로메테우스 타겟그룹을 생성하고 각 타겟그룹의 특성에 맞는 health check 파라메터를 입력한다. 
* Health checks for Application Load Balancer target groups: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html
##### 4. AWS Cloudwatch Alarm을 이용해서 이상 상태 발생 시 담당자에게 알람 전송
AWS CloudWatch SNS을 이용해서 SMS(문자메시지)를 받기 위해선 리전에 따라 다르지만 우리가 사용하는 리전은 SMS발송이 안되기 때문에 
부득이하게 다른 리전을 사용하게 되면서 발생한 문제가 AWS CloudWatch Alarm을 이용해서 SNS에 직업 호출하지 못한다는 문제가 있다. 
따라서 AWS CloudWatch에서 AWS Lambda를 호출하고 이 Lambda가 다른 리전에 있는 SNS에 메시지를 발송할 수 있도록 해야 한다. 
* Invoke a Lambda function from an alarm https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarms-and-actions-Lambda.html


##### 5. 예상비용
예상비용 
개발계 t3.large x 3 EA = 227.76 USD/월
운영계 m5.2xlarge x 2 EA = 543.12 USD/월 
EFS x 6EA = 267.40  x 6 =  1602 USD/월
합계 2372 USD/월 

