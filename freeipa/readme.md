## freeipa로 자체 IdP 구축하기


##### 발단
IDP 도입은 공용계정사용금지(공용 PEM키)와 서버 접근 제어 도입이라는 점검 지적사항에서 시작되었습니다.
제 생각에 이런 지적에 대한 부분은 클라우드 환경에서 가상서버를 사용하고 있는 조직들이 대부분 겪을 수 있는 문제라고 여겨집니다.
특별하게 클라우드 자원을 전사 차원에서 관리하는 조직을 빼고 대부분은 가상 서버 인스턴스를 생성하고 난 뒤 용도에 맞게 여러가지 서버 어플리케이션을 설치하고 환경을 구성하고 테스트도 수행을 
해야하는데 이 과정에서 보통은 CSP가 제공한 ssh 접속용 PEM키를 쓰는 경우가 많습니다. 
이 PEM키는 root의 권한을 갖고 있는 인스턴스 사용자를 위한 key이다. (root가 원격으로 불허하는게 보안에서는 표준으로 되어 있기 때문에 
일반 사용자로 접속해서 su 명령으로 root로 변신하던가. sudo 명령으로 작업을 수행하게 됩니다. 
 
 ##### 우리는…. 그런데 
 이 PEM이 하나이고 공용으로 사용되는 것이 문제의 발생 원인입니다. 이 키를 어딘가의 jump 호스트에 저장하고 공용으로 사용하던가 혹은 각 담당자의 PC에 보관되는 경우도 왕왕 있습니다. 
물론 여러 암호 관리 해결책도 많고 다른 서버 관리 자동화 도구도 많지만, 저희의 환경은 그런 것과는 거리가 멀었습니다. 
이미 프로덕션 환경에서 운영되고 있는 서비스들과 적은 수의 관리자(2명)가 새롭게 무엇인가를 도입해서 시스템에 적용하기엔 기존에 할당된 다른 업무도 줄 서 있는 상태에서 
새롭게 해결책을 도입하고 자동화를 하기엔 무리가 있다고 판단했기에 두 가지 지적을 하나로 묶어서 처리하기로 했습니다. 

##### 개선을 위한 방안 고려
공용 계정을 사용하지 않고 로컬 계정을 사용하기 위해서 서버에 계정을 만드는 것부터 검토를 시작했습니다. 
먼저 계정을 서버별로 생성하자는 의견이 나왔는데 이렇게 되면 구성원이 변경되는 경우 모든 서버에서 계정을 생성해 줘야 합니다 
한 300대 정도 되는 서버는 많다면 많고 적다면 적은 대수인데, 하루 날 잡고 300개의 계정을 만들기 위해서 300번의 로그인을 할 엄두가 나지 않았습니다.
물론 AWS systems manager의 run command 사용이다. ansible 구축 후 일괄 처리와 같은 여러 방법을 이용할 수 있지만 
조직의 특성상 개발과 운영이 공존하고 있고 개발자들이 들어가는 서버와 인프라 담당자들이 들어가는 서버가 약간씩 차이가 나기 때문에 
매번 서버별로 계정을 생성하고 권한을 분리하는데 업무를 너무 과중하게 만든다고 생각했습니다.
그래서 IdP를 사용하기로 했지만, 전사적으로 사용하고 있는 okta에 우리 팀의 서비스를 위해서 새롭게 뭔가를 구성해달라고 하는 것은 
힘든 일이기 때문에 팀 자체적으로 우리 팀만을 위한 idP 를 구성하기로 하고 LDAP + kerberos로 만들면 좋겠다는 아이디어를 발판으로 FreeIPA를 사용하기로 했습니다.

##### 설치
먼저 Fedora42 인스턴스 2대를 준비하고 각각 Master/Slave로 설정했습니다. (Master는 service internal only network, slave는 company wide only network)
관리하는 리눅스 서버들은 모두 Amazon Linux 2023으로 업그레이드를 완료한 상태였지만 오래된 버전은 freeipa 패키지가 없어서 OS 패치 버전을 통일시키는 작업부터 진행했습니다. 
```
# sudo dnf upgrade --releasever=2023.1.20230628 
```
사용하기 위해서 먼저 DNS를 설정하고 설치를 진행했습니다.
그리고 서버의 설치는 아래의 명령으로 설치 합니다.
```
# sudo dnf install -y freeipa-server freeipa-server-dns
# sudo ipa-server-install
```
명령은 두줄 이지만 설치 중간에 질문이 몇 개 나오는데 답을 입력하면 설치가 완료됩니다. 
이 과정에서는 아래에 있는 문서들이 도움이 될 수 있습니다.
- [RedHat Identity Management installation](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/10/html/installing_identity_management/index) 
- [Quick Start Guide](https://www.freeipa.org/page/Quick_Start_Guide)
- [Deployment Recommendations](https://www.freeipa.org/page/Deployment_Recommendations)
- [How Tos](https://www.freeipa.org/page/HowTos)

저는 주로 RedHat의 공식 문서를 주로 참고했습니다.
 - [RedHat Identity Management ](https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/10#Identity%20Management)

그리고 클라이언트는 아래의 명령으로 설치할 수 있다. (위에 링크된 매뉴얼에 더 자세한 내용이 있습니다.)
```
# sudo dnf install -y freeipa-client  
# sudo ipa-client-install
```


서버 설치 후 계정이 추가된 모습 (계정을 추가하고 지정하는 건 매뉴얼을 보시기 바랍니다.)
![WEBUI](ipa-web.png)

##### 구성
구성이 복잡하진 않지만 FreeIPA를 모든 서버들과 연동을 하는 작업은 서버별로 직업 접속해서 최초 설정을 해줘야 합니다.
AWS에서 AMI등으로 백업 후 새로운 인스턴스를 만드는 경우엔 ipa-client-install을 다시 실행해야 합니다. 

일단 서버들이 IPA에 연동이 되면 ssh는 sssd 를 통해서 IPA로부터 Kerberos 인증을 받은 사용자가 로그인 할 수 있게 됩니다.
자세한 설명: https://docs.redhat.com/ko/documentation/red_hat_single_sign-on/7.6/html/server_administration_guide/sssd

IdP를 도입한 후 계정별 패스워드 생명주기, 패스워드 복잡도 등을 한곳에서 제어할 수 있게 되었고, 계정별로 호스트 단위에 맞는 권한을 줄 수 있고, Permission, Privileges, Role을 조합해서 RBAC을 구현할 수도 있습니다.
많은 수의 서버 인스턴스가 변경되는 경우엔 프로젝트 단위 혹은 팀 단위로 내부에서 사용할 IdP를 구축하는 것을 고민해 볼만한 가치가 있다고 생각됩니다. 
 
 
 ##### 만난 문제
 AWS ECS에서 새롭게 추가된 EC2 인스턴스들의 자동 등록 문제가 있는데 Auto Scale out으로 추가된 인스턴스의 경우 원본 마스터 AMI에 ipa client가 설치되어 있었어도 IPA 서버에 
호스트가 추가가 안 되어 있고 인증서도 자신의 IP/Hostname과 불일치하니 IPA를 사용할 수 없었습니다. 
 
 ##### 우회
 AWS ECS에서 Auto Scale out으로 추가된 인스턴스와 신규 설치되는 EC2 인스턴스들의 경우 동일한 문제가 발생했기 때문에 
 일차적으로 ECS의 경우엔 트러블 슈팅을 위해서 SSH를 사용하는 비율이 적다는 결론이 나왔고 
 신규로 설치되는 EC2 인스턴스들은 어차피 모든 설정을 위해서 접근제어를 사용하기 이전에 설정되어야 하는 내용이 있기 때문에  
 EC2 Serial console을 이용하기로 했습니다. 
 이것은 IDC에서도 직접 KVM을 연결하는 것과 동일한 방법이기 때문에 문제가 없다는 결론에 도달했습니다. 
 접근제어 솔루션 장애 시 시스템에 문제가 생기면 접속하는 방안으로 사용되고 있습니다. 
 
 ##### 결론
 IdP를 도입하고 사용자별로 계정과 권한을 부여한 뒤 업무에는 변화가 없습니다. 오히려 계정 관리가 한 곳에서 되기 때문에 
 계정관리는 쉬워졌습니다. 서버별로 패스워드 복잡도나 생명주기를 신경 쓰지 않아도 됩니다. 
 아마도  IdP를 사용하지 않았다면 서버가 생성될 때마다 계정을 만들어야 했고 이건 사용자마다 접속할 수 있는 서버가 다르기 때문에 
 서버별로 해야 했을 것입니다. 그런데 idP가 관리에 대한 시간을 줄여줬기 때문에 관리자들은 더 많은 자유 시간을 갖게 되었습니다. 
 물론 그 시간에 쌓인 티켓을 처리해야 하는 건 똑같습니다. 