****************************************************
TACO 2.0 AIO 설치 가이드 (Kubernetes & Ceph Deployment)
****************************************************

이 매뉴얼은 AWS에서 생성한 EC2 환경을 기준으로 작성되었으며, Kubernetes, Ceph을 하나의 VM에 모두 설치하는 AIO (All-In-One) 가이드이다.

.. contents::
  :local:

설치 환경
=========

Single Host 사양
^^^^^^^^^^^^^^^^

* Instance : AWS EC2
* OS : CentOS Linux 7.7.1908
* Flavor : m5.2xlarge
* vCPU : 8
* MEM : 32G   <--24G 이상 권장

볼륨 디스크
^^^^^^^^^^^

* Root Disk Volume : 200G   <--160G 이상 권장
* Additional Empty Volume : 50G   <--50G 이상 권장

네트워크 설정
^^^^^^^^^^^^

* 포트 개방 : 80, 30608 (+LMA 설치시 30001, 30009, 30018, 32000)

Installing Guide
================

계정 설정
^^^^^^^^^

이 매뉴얼에서 사용하는 계정은 centos 이며, '$'로 시작하는 커맨드는 centos 계정에서 수행함을 의미한다.

* sudo 권한 부여

centos 계정에서 패스워드 없이 sudo를 사용할 수 있도록 아래와 같이 설정한다.

.. code-block:: bash

   $ sudo visudo
   ## Allow root to run any commands anywhere
   root    ALL=(ALL)       ALL
   centos  ALL=(ALL:ALL) NOPASSWD: ALL

|

* ssh 패스워드 설정

패스워드를 통해 ssh 로그인을 할 수 있도록 아래 변수를 yes로 바꿔준다.

.. code-block:: bash

   $ sudo vi /etc/ssh/sshd_config
   PasswordAuthentication yes

|

패스워드를 설정하고 변경된 sshd 설정을 반영시킨다.

.. code-block:: bash

   $ sudo passwd centos
   $ sudo systemctl restart sshd

|

Pre-installation
^^^^^^^^^^^^^^^^

TACO 설치에 필요한 패키지와 소스 코드를 다운로드한다.

* 패키지 업데이트 및 다운로드

.. code-block:: bash

   $ sudo yum update -y
   $ sudo yum install -y epel-release git
   $ sudo yum install -y python-pip
   $ sudo yum update -y 

|

* tacoplay 다운로드

tacoplay는 ansible playbook 모음을 이용하여 TACO를 자동으로 설치하는 프로그램이다.

.. code-block:: bash

   $ git clone -b taco-v20.05 --single-branch https://github.com/openinfradev/tacoplay.git ~/tacoplay
   $ cd $_

|

tacoplay에 필요한 패키지와 소스 코드를 다운로드한다.

.. code-block:: bash

   $ sudo pip install --upgrade pip
   $ sudo pip install -r requirements.txt --upgrade --ignore-installed
   $ ./fetch-sub-projects.sh

|

인벤토리 수정
^^^^^^^^^^^^

인벤토리 설정을 위해 필요한 로컬 정보를 아래의 방법으로 확인한다.

* { Additional_Empty_Volume } : nvme1n1   <--추가한 50G 빈 볼륨

.. code-block:: bash

   $ lsblk
   ##(example)
   nvme0n1     259:0    0  200G  0 disk
   └─nvme0n1p1 259:1    0  200G  0 part /
   nvme1n1     259:2    0   50G  0 disk

|

* { host_ip } : 172.32.0.81   <-- 아래 출력된 결과의 9번째 줄에서 확인 가능.

* { network_cidr } : 172.32.0.0/24   <--아래에서 출력된 9번째 줄에서 확인 가능한 172.32.0.81/24의 네 번째 옥텟을 0으로 바꾼 값.

.. code-block:: bash

   $ ip a
   ##(example)
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
       inet 127.0.0.1/8 scope host lo
          valid_lft forever preferred_lft forever
       inet6 ::1/128 scope host
          valid_lft forever preferred_lft forever
   2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
       link/ether 02:ae:fa:f2:88:84 brd ff:ff:ff:ff:ff:ff
       inet 172.32.0.81/24 brd 172.32.0.255 scope global dynamic ens5
          valid_lft 3520sec preferred_lft 3520sec
      inet6 fe80::ae:faff:fef2:8884/64 scope link
          valid_lft forever preferred_lft forever

|

* 인벤토리 설정

제공된 샘플 extra-vars.yml 에서 아래와 같이 5가지 항목의 value를 수정한다.

.. code-block:: bash

   ##{ } 안에 알맞은 값을 대입하여 아래 설정을 extra-vars.yml에 저장한다.
   $ vi ~/tacoplay/inventory/sample/aio/extra-vars.yml
   taco_apps: [""]
   monitor_interface: { ethernet_interface }   ##should be edited
   public_network: { network_cidr }   ##should be edited
   cluster_network: { network_cidr }   ##should be edited
   lvm_volumes:
     - data: /dev/{ Addtional_Empty_Volume }   ##should be edited

|

* (optional) LMA (Logging, Monitoring, Alerting) 설치를 위한 인벤토리 설정

LMA를 설치하면 TACO가 관리하는 리소스의 로그와 사용 현황을 확인할 수 있는 대쉬보드가 제공된다.

제공된 샘플 extra-vars.yml 에서 아래와 같이 1가지 항목의 value를 수정한다.

.. code-block:: bash

   ##taco_apps의 value에 "lma"를 추가하면 자동으로 LMA를 설치한다.
   $ vi ~/tacoplay/inventory/sample/aio/extra-vars.yml
   taco_apps: ["lma"]

|

tacoplay 실행
^^^^^^^^^^^^

위의 설정을 모두 마쳤다면 tacoplay를 실행한다.

.. code-block:: bash

   $ cd ~/tacoplay/
   $ ansible-playbook -b -i inventory/sample/aio/hosts.ini -e @inventory/sample/aio/extra-vars.yml site.yml

|

테스트 환경 사양에 따라 배포 완료 시간이 30분 정도에서 1시간 정도까지 달라질 수 있다.

Taco apps 설치
^^^^^^^^^^^^^^

* LMA 설치
Tacoplay를 통한 LMA 등의 taco_apps는 [Decapod](https://github.com/openinfradev/decapod-flow.git)을 사용한다. 
아래 메뉴얼은 Argo CLI를 사용하는 방법이다. Argo UI(http://{ host_ip }:30004/)를 통해서도 실행할 수 있다.
.. code-block:: bash

   $ git clone https://github.com/openinfradev/decapod-flow.git
   $ cd decapod-flow/workflows
   $ argo submit --from wftmpl/prepare-manifest -n argo
   $ argo list -n argo
   // prepare-manifest-XXX workflow가 완료될 때 까지 기다린다.

   $ argo submit lma-federation-wf.yaml

|

* (Optional) LMA 커스터마이징
위에서 설치한 LMA에 대한 configuration을 보거나 수정하고 싶다면,
[decapod-base-yaml](https://github.com/openinfradev/decapod-base-yaml.git)과 [decapod-site-yaml](https://github.com/openinfradev/decapod-site-yaml.git)을 참조하여 자신의 site-yaml을 만들어야 한다.

1. [decapod-site-yaml](https://github.com/openinfradev/decapod-site-yaml.git)을 fork한다.
2. decapod-site-yaml/lma/site/hanu-deploy-apps/site-values.yaml 의 값들을 바꾸고 commit한다.
3. Argo CLI로 prepare-manifest를 다시 실행한다.
.. code-block:: bash

   $ argo submit --from wftmpl/prepare-manifest -p site_yaml_url=https://github.com/{ your_repo }/decapod-site-yaml.git

|

4. LMA를 재 배포한다.
.. code-block:: bash

   $ argo submit lma-federation-wf.yaml

|

* LMA 접속

LMA를 설치한 경우 아래 접속 정보를 참고하여 웹 브라우저로 접속해본다.

   * Kibana: http://{ host_ip }:30001/
   아이디 / 패스워드 : elastic / tacoword
   
   * Grafana : http://{ host_ip }:30009/
   아이디 / 패스워드 : admin / password


Trouble Shooting
================

* ansible 로그 확인 방법
1. 디폴트로 생성되는 로그는 /tmp/ansible.log를 확인한다. 로그를 별도로 관리하고자 한다면 '> example_file.log_0' 옵션을 붙여 로그를 원하는 파일에 생성할 수 있다.
2. ansible-playbook 명령 시 -vvvv 옵션을 추가하면 더 구체적인 로그가 기록된다.

* ansible 설치 중에 문제가 발생하여 재설치할 때 tag를 이용하여 일부 role만 수행하는 방법
tacoplay 실행 시 tacoplay/site.yml에 작성되어 있는 role의 순서대로 설치가 진행된다. 설치는 크게 보았을 때 ceph - K8s - taco_app(LMA) 순으로 진행된다. 이를 부분적으로 설치하고 싶다면 아래 명령을 수행하면 된다.

.. code-block:: bash

   ##1. 초기 세팅 및 ceph의 설치를 진행하는 커맨드(ceph이 이미 설치된 경우 에러가 발생할 수 있으니 주의한다.)
   $ ansible-playbook -b -i inventory/sample/aio/hosts.ini -e @inventory/sample/aio/extra-vars.yml site.yml --tags setup-os,ceph,ceph-post-install --skip-tags k8s
   
|

.. code-block:: bash

   ##2. ceph이 정상적으로 설치되었을 때, K8s를 설치하는 커맨드(ceph을 중복으로 설치하게 되면 문제가 발생하여 스킵해준다)
   $ ansible-playbook -b -i inventory/sample/aio/hosts.ini -e @inventory/sample/aio/extra-vars.yml site.yml --tags ceph-post-install,k8s,taco-clients --skip-tags ceph

|

.. code-block:: bash

   ##3. K8s까지 정상적으로 설치되었을 때, taco_app(LMA)의 배포 혹은 남은 role을 수행하는 커맨드
   $ ansible-playbook -b -i inventory/sample/aio/hosts.ini -e @inventory/sample/aio/extra-vars.yml site.yml --skip-tags ceph,k8s

|

* K8s 설치 관련 문제 발생 시
1. kube-system 네임스페이스를 갖는 K8s 리소스들이 잘 작동 중인지 확인한다.

.. code-block:: bash

   $ kubectl get pods -n kube-system
   $ kubectl get services -n kube-system
   $ kubectl get deployments -n kube-system

|

2. "The connection to the server localhost:8080 was refused - did you specify the right host or port?"와 같은 문구가 발생한다면

.. code-block:: bash

   $ mkdir -p $HOME/.kube
   $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   $ sudo chown $(id -u):$(id -g) $HOME/.kube/config

|

위 명령을 순차적으로 수행한다. root 계정에서는 K8s 클러스터에 접근할 수 있으나 centos와 같은 user 계정에서 접근하지 못할 때 발생한다.(참고: https://snowdeer.github.io/kubernetes/2018/02/13/kubernetes-can-not-use-kubectl/)
