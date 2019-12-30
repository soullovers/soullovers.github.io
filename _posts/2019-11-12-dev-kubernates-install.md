---
layout: post
title: "쿠버네티스 소개"
categories:
  - dev
tags:
  - kubernates
---



# Kubernetes on GCE

https://console.cloud.google.com/
https://cloud.google.com/sdk/docs/quickstarts?hl=ko

* python과 curl이 먼저 설치가 되어 있어야 한다.

```
## python 버전 확인(2.7이상 필수)
$ python -V
$ brew install python


$ curl -V
$ brew install curl
```


#### cloud 설치
```

$ curl https://sdk.cloud.google.com | bash


## 쉘 다시 시작
$ exec -l $SHELL


## 환경 초기화
$ gcloud init


## 설정 확인
$ gcloud config list
```


#### gcp계정 정보 설정
```
$ gcloud auth login
```

#### 기본프로젝트 확인
```
$ gcloud config list project
```



#### 기본프로젝트 설정
```
$ gcloud config set project <프로젝트 아이디>
```


#### 프로젝트 아이디 조회
```
$ gcloud alpha projects list 
```



#### 쿠버네티스 설치
```
$ mkdir gsw-k8s-3 
$ cd gsw-k8s-3 
$ curl -sS https://get.k8s.io | bash
```



#### 쿠버네티스 실행
```
$ kubernetes/cluster/kube-up.sh 
$ gcloud container clusters create kubia --num-nodes 3 --machine-type f1-micro
```




#### openssl 에러 발생시
```
## open ssl설치
$ brew update
$ brew install openssl
$ echo 'export PATH="/usr/local/opt/openssl/bin:$PATH"' >> ~/.bash_profile
$ source ~/.bash_profile
$ brew link --force openssl
```




#### 컴포넌트 부족 에러 발생시
```
... Starting cluster in us-central1-b using provider gce
... calling verify-prereqs
missing required gcloud component "beta"
Try running `gcloud components install beta`


## 부족한 컴포넌트 확인
$ gcloud components list 


## 컴포넌트 설치
$ gcloud components install bata
Your current Cloud SDK version is: 272.0.0
Installing components from version: 272.0.0


┌─────────────────────────────────────────────┐
│     These components will be installed.     │
├──────────────────────┬────────────┬─────────┤
│         Name         │  Version   │   Size  │
├──────────────────────┼────────────┼─────────┤
│ gcloud Beta Commands │ 2019.05.17 │ < 1 MiB │
└──────────────────────┴────────────┴─────────┘

For the latest full release notes, please visit:
  https://cloud.google.com/sdk/release_notes

Do you want to continue (Y/n)?  Y


╔════════════════════════════════════════════════════════════╗
╠═ Creating update staging area                             ═╣
╠════════════════════════════════════════════════════════════╣
╠═ Installing: gcloud Beta Commands                         ═╣
╠════════════════════════════════════════════════════════════╣
╠═ Creating backup and activating new installation          ═╣
╚════════════════════════════════════════════════════════════╝

Performing post processing steps...done.                                                                                                                                                                                                                                                                                     

Update done!



$ gcloud components install alpha
$ gcloud components install kubectl

```




#### 설정
```
## 기본 컴퓨팅 영역을 설정
$ gcloud config set compute/zone {zone} 
$ gcloud config set compute/zone us-central1-a 


## kubeconfig 항목 생성
$ gcloud container clusters get-credentials {클러스터이름}
$ gcloud container clusters get-credentials kubia




## gcloud 최신버전으로 업데이트
$ gcloud components update
```



#### ssh 설정
```
$ sudo gcloud compute config-ssh
$ ssh gke-kubia-default-pool-1fe57469-0pg2.us-central1-a.gke-soullovers
```


#### 마스터에 접속해서 실행 중인 docker 컨테이너 출력
```
$ gcloud compute ssh --zone us-central1-a gke-kubia-default-pool-1fe57469-frmq
$ docker container ls --format 'table {{.Image}}\t{{.Status}}'
```



#### 마스터에서 실행되는 서비스
- fluented-gcp : 클러스터 로그 파일을 모아 구글 클라우드 로깅 서비스로 전송하는 컨테이너
- node-problem-detector : 모든 노드에서 실행되며 하드웨어 및 커널 계층에서 생기는 문제를 감지하는 데몬
- rescheduler : 중요한 컴포넌트가 항상 실행 중인지를 확인하는 애드온 컨테이너. 리소스가 모자랄 경우, 덜 중요한 파드를 제거해 공간을 확도할수도 있다.
- glbc : 인그레스(ingress) 기능을 사용해 구글 클라우드 레이어 7 로드밸런싱을 제공하는 쿠버네티스 애드온 컨테이너
- kube-addon-manager : 쿠버네티스를 확장하는 핵심 컴포넌트. 변경 사항을 주기적으로 /etc/kubernetes/addons 디렉터리에 적용
- etcd-empty-dir-cleanup : etcd에 있는 빈(empty) 키를 정리하는 유틸리티
- kube-controller-manager : 여러가지 클러스터 기능을 제어하는 컨트롤러 관리자, 최신 상태로 리플리케이션을 보장해줌, 새로운 노드를 모니터링, 관리, 서비스 엔드포인트를 관리하고 업데이트
- kube-apiserver : API 서버를 실행, RESTful API를 통해 쿠버네티스 클러스터의 다양한 컴포넌트를 생성, 조회, 업데이트, 삭제
- kube-scheduler : 스케줄링되지 않은 파드를 현재 사용 중인 스케줄링 알고리즘에 따로 노드에 바인드하는 스케줄러
- etcd : CoreOS에서 만든 분산 키/값 저장소인 etcd 소프트웨어를 실행, 쿠버네티스 클러스터의 상태가 저장
- pause : 파드 인프라 컨테이너, 각 파드의 네트워킹 네임스페이스와 리소스 제한을 설장하고 유지하는데 사용


#### 노드에서 실행되는 서비스
- kubedns : 쿠버네티스에 있는 서비스와 엔드포인트 리소스를 모니터링하며 변경 사항을 DNS 룩업에 동기화 시킨다.
- kube-dnsmasq : DNS 캐싱을 제공하는 또 다른 컨테이너
- dnsmasq-metrics : 클러스터 DNS 서비스의 메트릭 리포팅을 제공
- l7-defaultbackend : GCE L7 로드 밸런서와 인그레스를 핸들링하는 기본 백엔드
- kube-proxy : 클러스터의 네트워크 및 서비스 프록시. 클러스터에서 워크로드가 실행되는 곳으로 서비스 트래픽이 전달되도록 한다.
- heapster : 모니터링과 분석을 위한 컨테이너
- addon-resizer : 컨테이너를 스케일링하는 클러스터 유틸리티
- heapster_grafana : 리소스 사용량을 모니터링
- heapster_influxdb : 힙스터 데이터를 저장하는 시계열(time-series) 데이터베이스
- cluster-proportional-autoscaler : 클러스터 크기에 비례해 컨테이너를 스케일링하는 클러스터 유틸리티
- exechealthz : 파드에 헬스체크 수행




#### 클러스터해제
```
$ kubernetes/cluster/kube-down.sh 

$ gcloud container clusters delete kubia
```




#### sdk 삭제
```
## 설치 디렉토리 찾기
$ gcloud info --format='value(installation.sdk_root)'

## 구성 디렉토리 찾기
$ gcloud info --format='value(config.paths.global_config_dir)'


## 두 디렉토리 삭제
```


# 2 Kubernetes on AWS
console.aws.amazon.com

kube-aws https://github.com/kubernetes-incubator/kube-aws
kops https://github.com/kubernetes/kops/blob/master/docs/install.md

#### kops설치
```
$ brew install kops
```




#### AWS CLI설치

https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/install-macos.html
```
$ pip3 --version 


## pip3 설치 스크립트를 다운로드하고 실행 
$ curl -O https://bootstrap.pypa.io/get-pip.py 
$ python3 get-pip.py --user 


## pip3를 사용하여 AWS CLI를 설치 
$ pip3 install awscli --upgrade --user 


## 버전확인 
$ aws --version 
```




#### AWS CLI 구성
https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/cli-chap-configure.html
보안자격증명 > 액세스 키(액세스 키 ID 및 비밀 액세스 키) > 새 액세스 키 만들기 > 발급 받은 키 등록
```
$ aws configure 
AWS Access Key ID [None]: {AWS_ACCESS_KEY_ID} 
AWS Secret Access Key [None]: {AWS_SECRET_ACCESS_KEY} 
Default region name [None]: {REGION_NAME} 
Default output format [None]: {FORMAT}
```




#### IAM 설정 - 권한부여 설정
```
$ aws iam create-group --group-name kops 


$ aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops 
$ aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops 
$ aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops 
$ aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops 
$ aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops 


$ aws iam create-user --user-name kops 
$ aws iam add-user-to-group --user-name kops --group-name kops 
$ aws iam create-access-key --user-name kops
```





#### 클러스터 생성
https://github.com/kubernetes/kops/blob/master/docs/getting_started/aws.md
```

$ export NAME=gswk8s3.k8s.local 
$ export KOPS_STATE_STORE=s3://gsw-k8s-3-state-store 


## 형상정보 저장할 버킷 생성 (kops가 버킷 위치를 강제하고 있어서 버킷은 us-east-1 리전에 만들어야 한다.)
$ aws s3api create-bucket --bucket gsw-k8s-3-state-store2 --region us-east-1 


## 버전관리 (문제가 발생하면 이전상태로 롤백)
$ aws s3api put-bucket-versioning --bucket gsw-k8s-3-state-store2 --versioning-configuration Status=Enabled 


## 리전을 볼수 있는지 확인 
$ aws ec2 describe-availability-zones --region us-east-2 


## 쿠버네티스 설치 예행연습 (aws에 구축하려는 내역을 미리보고 편집할수 있다)
$ kops create cluster --zones us-east-2a ${NAME} 

```


#### 
```

## list clusters with: 
$ kops get cluster 


## 클러스터 생성
$ kops update cluster --name gswk8s3.k8s.local --yes 


$ kops validate cluster

```


#### 노드리스트
```

$ kubectl get nodes --show-labels
```


#### ssh설정
```
$ ssh -i ~/.ssh/id_rsa admin@<마스터IP>


## ssh키에 문제가 있을 경우 kops으로 시크릿을 생성해서 클러스터에 수작업으로 추가
$ kops create secret --name gswk8s3.k8s.local sshpulickey admin -i ~/.ssh/id_rsa.pub 
$ kops update cluster --yes 




## 롤링업데이트 실행
$ kops rolling-update cluster --name gswk8s3.k8s.local

```


#### 컴포넌트 상태 확인 
```
$ kubectl get componentstatuses
```

#### 컴포넌트 삭제
```
$ kops delete cluster --name ${name} --yes 
## yes 플래그를 생략하면 실제 삭제 하지 않고 실행결과를 미리 볼수 있다
```


# Kubernetes on Local Cluster
https://github.com/kubernetes/minikube
https://kubernetes.io/ko/docs/setup/learning-environment/minikube/


#### 설치
```
## install minikube 
$ brew install minikube 


## HyperKit installation 
$ brew install hyperkit 


## Start a cluster using the hyperkit driver: 
$ minikube start --vm-driver=hyperkit 
$ minikube start --vm-driver=virtualbox


## To make hyperkit the default driver: 
$ minikube config set vm-driver hyperkit

```




#### Examples
```
## Access the Kubernetes Dashboard running within the minikube cluster: 
$ minikube dashboard 


## Once started, you can interact with your cluster using kubectl, just like any other Kubernetes cluster. For instance, starting a server: 
$ kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.4 


## Exposing a service as a NodePort 
$ kubectl expose deployment hello-minikube --type=NodePort --port=8080 


## minikube makes it easy to open this exposed endpoint in your browser: 
$ minikube service hello-minikube 


## Start a second local cluster (note: This will not work if minikube is using the bare-metal/none driver): 
$ minikube start -p cluster2 


## Stop your local cluster: 
$ minikube stop 


## Delete your local cluster: 
$ minikube delete 


## Delete all local clusters and profiles 
$ minikube delete --all
```

## 쿠버네티스 UI
```
## 대시보드 UI 배포 
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta6/aio/deploy/recommended.yaml 


## 대시보드가 실행가능하도록 하는 proxy 커맨드 
$ kubectl proxy --port=8001 


## 토큰정보 확인 
$ kubectl config view | grep token
```

http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/ 접속
