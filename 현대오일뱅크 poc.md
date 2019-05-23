현대오일뱅크 poc

서버구성

172.18.21.148~158

node01 148

node02 149

VM01 150

VM09 158

배스쳔(VM09): 172.18.21.158

- bastion01.oilbank.co.kr

- DNS, NFS, Docker Registry, Ansible (기타...)

마스터(VM01): 172.18.21.150

- master01.oilbank.co.kr

라우터01(VM02): 172.18.21.151

- router01.oilbank.co.kr

- 라우터 (HA Proxy)

인프라01(로깅,모니터링)(VM03): 172.18.21.152

- infra01.oilbank.co.kr
- EFK, Prometheus

앱01(VM04): 172.18.21.153

- app01.oilbank.co.kr

앱02(VM05): 172.18.21.154

- app02.oilbank.co.kr

앱03(VM06): 172.18.21.155

- app03.oilbank.co.kr





hostnamectl set-hostname app01.oilbank.co.kr



AIR SVN: http://61.251.161.213:8080/svn/repos_mobile/trunk/src/was/

AIR 개발기 서버:172.18.200.22 (hdomobile / ahqkdlf51!!)

Matoms: 211.56.68.144 - Matoms/Docker (root / oilbank!)

http://gogs.poc-infra.svc.cluster.local:3000/hyundai/air-tomcat.git





docker load & tag & push

```
#master 서버에서

sudo cat oraclelinux_webuser_mobiledis_20190521.tar | docker load

sudo docker tag oraclelinux:mobilecust6 docker-registry.default.svc:5000/openshift/mobilecust6:1.0

sudo docker tag oraclelinux:mobiledis8 docker-registry.default.svc:5000/openshift/mobiledis8:1.0

oc whoami -t
#토큰 복사
13uhx-e_RK-uZXN4pSj2_D2L-f_URzC-phvXEnJAJts


docker login -u admin -p 13uhx-e_RK-uZXN4pSj2_D2L-f_URzC-phvXEnJAJts docker-registry.default.svc:5000

docker push docker-registry.default.svc:5000/openshift/mobilecust6:1.0




```



컨테이너에 루트 권한 취득

```
oc adm policy add-scc-to-user anyuid -z default
```

jenkins token:

eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJwb2MtaW5mcmEiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlY3JldC5uYW1lIjoiamVua2lucy10b2tlbi04cW4yaiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJqZW5raW5zIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiN2FkMDJiMGUtN2M1ZC0xMWU5LThiMzQtNTI1NDAwZmZiYjc5Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OnBvYy1pbmZyYTpqZW5raW5zIn0.n2ganJTfXB0fthaIjB82TbGdorM5p2gYs-TAYQ9fmf_KJYKY4eCyNSMlZ8-qGiVPAEFYs9c7PPEeobdQkXsyDidDobvM1FtIsqcZM2Ye1tWhd_uMMx9ock5YzavJ-Hxu9rTbllJaPIV-wMvSFldwzT8mRbEBJyUkHxyXYEkg8m-O77cmkDGWBCqzG-HHxu43JUbHVvhyqlzSvKq7tfTfPmZIOphZbRTznWCKoUMJM_USo58GD2VLjDmv3Y-o0iV3s4jQqHkKgFhIg6W8Boet8cOK2o0sI8RoSZLW3lBQhdEVoe6gnqK1idmmY1M326Y3LdxcpA7meaToq-WWi8KmTA



```
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: hdo-airtomcat-jenkins
spec:
  runPolicy: Serial
  source:
    git:
      ref: master
      uri: "http://gogs.poc-infra.svc.cluster.local:3000/hyundai/air-tomcat.git"
    type: Git
  strategy:
    jenkinsPipelineStrategy:
      jenkinsfilePath: jenkinsfile
    type: JenkinsPipeline

```





현대오일뱅크 poc 시나리오

목요일.

1. s2i build (console에서 배포시)

   - web console에서 tomcat 배포

   . Browse catalog > Runtimes & Frameworks > JBoss Web Server 3.1 Apache Tomcat 7 (no https)

   - 정보 입력

   .Application Name=hdo-air

   .Git Repository URL=http://gogs.poc-infra.svc.cluster.local:3000/hyundai/air-tomcat.git

   .Git Reference=master

   .Context Directory=/

   .JWS Admin Username=admin

   .JWS Admin Password=admin

   .ImageStream Namespace=openshift

   .ARTIFACT_DIR=/deployments

   - Create 버튼 클릭

2. s2i build (binary build, Terminal에서)

   - bastion에서 아래 명령어 실행 (build config 생성)

     ```
     oc new-build --image-stream=jboss-webserver31-tomcat7-openshift:1.2 --name=hdo-airtomcat --binary=true
     ```

   - build config 기반으로 실행 (Application root에서)

     ``` 
     cd /root/hdopoc/air-tomcat
     oc start-build hdo-airtomcat --from-dir=. --follow --wait
     oc new-app hdo-airtomcat
     oc expose svc/hdo-airtomcat
     ```

   - 이미지 교체 후 재빌드

     ```
     cd /root/hdopoc/air-tomcat/WebContent/images
     cp main_title_openshift.png main_title.png
     cd /root/hdopoc/air-tomcat
     ant -f build.xml
     oc start-build hdo-air --from-dir=. --Follow=true --wait=true
     ```

3. Git trigger

   - hdo-air Pod 갯수 세개로 증가

   - builds > hdo-app > Configuration

   - Triggers > Generic Webhook URL을 복사

   - gogs 접속

   - <http://gogs.oilbank.co.kr/hyundai/air-tomcat>

   - gogs 열어서 설정 > Webhooks > 추가(Git) > gogs > 페이로드 URL에 URL 입력

   - Add Webhook 버튼 클릭

   - paste url 된 걸 payload url에 복사

   - Content Type 은 application/x-www-form-urlencoded

   - 웹훅 > 모든것이 필요합니다.

   - 이미지 교체 후 재빌드하여 git push
   
     ```
     cd /root/hdopoc/air-tomcat
     git pull
     cd /root/hdopoc/air-tomcat/WebContent/images
     cp main_title_asis.png main_title.png
     cd /root/hdopoc/air-tomcat
     ant -f build.xml
     git add .
     git commit -m "source change"
     git push origin master
     ```

4. mobilecluster/mobiledist 배포

   - project 생성

     ```
     oc new-project hdo-mobile
     oc adm policy add-scc-to-user anyuid -z default
     ```

   - console에서 deploy image

     openshift>mobilecust6>1.0

     openshift>mobiledis8>1.0

   - pods> terminal 들어가서 확인

5. gcafe-php build

   ```
   oc new-project hdo-gcafe
   oc new-app php-55-rhel7~http://gogs.poc-infra.svc.cluster.local:3000/hyundai/gcafe.git gcafe
   
   oc new-app jboss-webserver31-tomcat7-openshift:1.2~http://gogs.poc-infra.svc.cluster.local:3000/hyundai/gcafe-tomcat.git
   ```

6. Pipeline build

   ```
   oc project hdo-jenkins
   #jenkins가 default 권한을 가지므로, 권한 업데이트 해줘야 함
   oc policy add-role-to-user edit -z default
   #Build > Pipeline 확인
   ```

   -------------------------------------------------------------------------------------------------------------------------------------------------------------

PoC 시연

1. Management & Monitoring

   - Admin console login (master01.oilbank.co.kr)

     - 왼쪽 Catalog 설명 / 오른쪽 App이 올라가는 Project를 볼 수 있다. 
     - 상단에서 Cluster console로 이동
     - Home > Status 설명 (Projects > All Project )
     - pro
     - Workloads > Pod 설명 (검색 > hdo-airtomcat) > Pod 메트릭
     - Builds > Image Streams 설명
     - Administration > Node 설명 > Node 클릭 > Node 메트릭

   - Pod Control : Log

     - 상단에서 Application Console로 이동
     - Project 선택 (hdopoc) > Route 클릭해서 화면 보여줌

     ```
     oc project hdopoc
     ```

     - Pod 원 클릭
     - Pod information 설명 > Pod 목록에서 하나 클릭
     - Details 설명
     - Logs 설명 
     - Terminal 설명 (ls -al , ps -ef)
     - Application > Deployments > hdo-air 클릭 > 이력 설명
     - 그 전의 Deployments를 클릭해서 Rollback
     - Rollback 클릭 후 Overview 화면으로 이동

   - Monitoring  (Prometheus / Grafana)

     - 상단에서 Cluster Console 로 이동
     - 왼쪽 메뉴바의 Monitoring > metrics
     - 프로메테우스 쿼리창에 node 타이핑 (instance:node_cpu:rate:sum)
     - Console 설명 / Graph 설명
     - 하단에 Add graph > 쿼리창에 node 타이핑 (instance:node_network_receive_bytes:rate:sum)
     - Console 설명 / Graph 설명
     - 다시 돌아가서, 왼쪽 메뉴바의 Monitoring > dashboard
     - Grafana 접속
     - 왼쪽 상단의 Home > K8s / Compute Resources / Cluster 선택 후 설명
     - 중앙 상단에 Add Panel 아이콘 클릭 > Graph 클릭
     - Panel Title의 Edit 
     - Metric의 Datasource를 Promethues로 변경
     - A에서 Query 입력 (node:node_cpu_saturation_load1:) 다시 A 버튼 클릭
     - 오른쪽 상단의 Back to Dashboard

   - Logging (EFK)
     - 상단의 Application Console로 이동
     - 검색창에 logging 타이핑 하여 openshift-logging 프로젝트 선택
     - kibana 서비스 선택하여 라우트 클릭해서 접속
     - Add a filter 밑에 .all로 변경
     - Kibana 상단의 Add a filter 클릭 
     - kubernetes.container_name > is > hdo 후 save
     - hdo 관련된 app의 로그를 확인 가능함

   - Project 백업 & 복구

     - cluster의 모든 project 보기

     ```
     oc get project
     ```

     - 백업할 project 선택

     ```
     oc project hdo-backup
     ```

     - 백업하기 & 지우기

     ```
     oc get -o yaml --export all > project.yaml
     oc delete project hdo-backup
     ```

     - Project 지우기 & 복구하기

     ```
     oc new-project hdo-backup
     oc create -f project.yaml
     ```

   - Auto-Scaling & Auto Healing

     - auto scaling 할 project로 이동
     - 콘솔에서 hdo-autoscaling project 들어가고 아래 cli 명령어 입력

     ```
     ssh master01
     oc project hdo-autoscaling
     ```

     - 수동 scaling 하여 2개로 늘림 (화살표)
     - Pod 원형 눌러서 목록 보여주고

     ```
     oc delete pod <pod name>
     ```

     - 재성성 되는 것을 확인
     - application > deployments
     - hdo-air 선택
     - Action > Add Autoscaler
     - Min: 1 Max: 10 CPU target :20%

     ```
     oc run web-load --restart='OnFailure' --image=siamaksade/siege -- -c300 -d1 -t5M http://hdo-air.hdo-autoscaling.svc.cluster.local:8080/ui/mo1000/mo1900ui.jsp
     oc get hpa -w
     oc delete job/web-load
     ```

   - S2i

     - Project 선택

     ```
     oc project hdo-s2i
     ```

     s2i build (console에서 배포시)

     - web console에서 tomcat 배포

     . Browse catalog > Runtimes & Frameworks > JBoss Web Server 3.1 Apache Tomcat 7 (no https)

     . Search catalog > tomcat > JBoss Web Server 3.1 Apache Tomcat 7 (no https)

     - 정보 입력

     .Application Name=hdo-air2

     .Git Repository URL=http://gogs.poc-infra.svc.cluster.local:3000/hyundai/air-tomcat.git

     .Git Reference=master

     .Context Directory=/

     .JWS Admin Username=admin

     .JWS Admin Password=admin

     .ImageStream Namespace=openshift

     .ARTIFACT_DIR=/deployments

     - Create 버튼 클릭

   - Binary Build

     - Project 선택

     ```
     oc project hdo-binary
     ```

     - Build Config 만들기

     ```
     # Bastion에서..
     oc new-build --image-stream=jboss-webserver31-tomcat7-openshift:1.2 --name=hdo-binary --binary=true
     # 실제 시연 시에는..
     cd /root/hdopoc/air-tomcat
     oc start-build hdo-binary --from-dir=. --follow --wait
     oc new-app hdo-binary
     oc expose svc/hdo-binary
     ```

   - Image Build

     ```
     oc project hdo-image
     ```

     - Console에서 Add to Project > Deploy Image
     - Image Stream Tag에서 openshift > mobilecust6 > 1.0
     - Name: mobilecust6-v3
     - Image Stream Tag에서 openshift > mobiledis8 > 1.0
     - Name: mobiledis8-v3

   - Git trigger

     - project 선택

       ```
       oc project hdo-s2i
       ```

     - hdo-air Pod 갯수 두개로 증가
     - builds > hdo-air > Configuration
     - Triggers > Generic Webhook URL을 복사
     - gogs 접속
     - <http://gogs.oilbank.co.kr/hyundai/air-tomcat>
     - gogs 열어서 설정 > Webhooks > 추가(Git) > gogs > 페이로드 URL에 URL 입력
     - Add Webhook 버튼 클릭
     - paste url 된 걸 payload url에 복사
     - Content Type 은 application/x-www-form-urlencoded
     - 웹훅 > 모든것이 필요합니다.
     - 소스 수정

     ```
     #bastion에서..
     cd /root/hdopoc/air-tomcat/WebContent/images
     cp main_title_openshift.png main_title.png
     cd /root/hdopoc/air-tomcat
     ant -f build.xml
     git add .
     git commit -m "image change"
     git push origin master
     (Username: hyundai / Password: openshift)
     ```

     - 다시 화면 보기 (빌드 트리거 확인)
     - 나중에 다 배포되면, (새 시크릿창을 열어서 보여주기)
     - Rollback
     - Application > Deployments > hdo-air 클릭 > 이력 설명
     - 그 전의 Deployments를 클릭해서 Rollback

   - CI/CD Pipeline

     ```
     oc project hdo-jenkins
     ```

     - Jenkins 배포하는법 대략 보여주기 (catalog에서 jenkins 검색)
     - Builds > pipeline > start pipeline

   - Canary 배포

     ```
     oc project hdo-canary
     ```

     - Applications > Routes

     - hdo-air-v2 눌러서 보여주기

     - Actions > Edit해서 한쪽을 v2를 100으로 하기

     - 새 시크릿창 열어서 보여주기

       

     

   

   

   

