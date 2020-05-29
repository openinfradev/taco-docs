***************************
TACO 2.0 AIO 설치 가이드
***************************

이 매뉴얼은 AWS에서 생성한 EC2 환경을 기준으로 작성되었다.

AWS EC2 Instance 사양
=====================

* OS : CentOS Linux 7.7.1908
* Flavor : m5.2xlarge
* vCPU : 8
* MEM : 32G (24G 이상 권장)
* Root Disk Volume : 200G
* Additional Empty Volume : 50G
* 외부 접근을 위해 개방할 포트 : 80, 30608, 31000

Installing Guide
================

* sudo 권한 설정

OS 설치시 기본으로 생성되는 centos 계정을 사용한다.

centos 계정이 패스워드 없이 sudo를 사용할 수 있도록 아래와 같이 설정한다.

.. code-block:: bash

   $ sudo visudo
   ## Allow root to run any commands anywhere
   root    ALL=(ALL)       ALL
   centos  ALL=(ALL:ALL) NOPASSWD: ALL

|

* 패키지 업데이트

.. code-block:: bash

   $ sudo yum update -y
   $ sudo yum install -y bridge-utils epel-release python3 git
   $ sudo yum update -y 

|

* tacoplay 다운로드

tacoplay는 taco를 설치하는 프로그램이다.

.. code-block:: bash

   $ sudo su - centos
   $ cd ~/
   $ git clone -b tacoplay-2.0 --single-branch https://github.com/openinfradev/tacoplay.git
   $ cd tacoplay/
   $ sudo yum install -y python-pip
   $ sudo pip install --upgrade pip
   $ sudo pip install -r requirements.txt --upgrade --ignore-installed
   $ ./fetch-sub-projects.sh

|

* (오픈스택을 설치하는 경우) 브릿지 네트워크 구성

K8s만 설치하는 경우 브릿지 네트워크 구성은 생략한다.

네트워크 인터페이스가 1개 뿐인 경우를 전제로 한다.

루트 계정에서 구성을 진행하며 명령어 앞에 # 은 루트 계정임을 의미한다.

.. code-block:: bash

   $ sudo su -
   # cd /etc/sysconfig/network-scripts/

|

default 이더넷 인터페이스명을 아래 명령어로 확인한다. AWS EC2 기준 인터페이스명은 "ens5" 이다.

.. code-block:: bash

   # ip a
   ex) ens5
|

network-scripts 디렉토리의 ifcfg-(인터페이스명) 파일을 아래 내용과 같이 수정한다. ex) ifcfg-ens5

.. code-block:: bash

   # vi ifcfg-ens5
   DEVICE=ens5
   ONBOOT=yes
   TYPE=Ethernet
   BRIDGE=br-data
   BOOTPROTO=none
   NM_CONTROLLED=no

|

network-scripts 디렉토리에 ifcfg-br-data 파일을 생성하여 아래 내용을 저장한다.

게이트웨이 주소는 "$ netstat -r" 명령어로 확인할 수 있다.

host ip 주소는 "$ ip a" 명령어로 확인할 수 있다.(netstat -r 혹은 route 명령으로 두개 모두 확인할 수 있는지 확인하고 이미지 첨부 예정)

.. code-block:: bash

   # vi ifcfg-br-data
   BOOTPROTO=none
   DEFROUTE=yes
   DEVICE=br-data
   GATEWAY=(게이트웨이 주소)
   IPADDR=(host ip 주소)
   NETMASK=255.255.255.0
   ONBOOT=yes
   TYPE=Bridge
   STP=no
   NM_CONTROLLED=no

|

설정한 내용을 적용한다.

.. code-block:: bash

   # systemctl restart network
   # brctl addbr br-data
   # brctl addif br-data ens5
   # ip link add veth0 type veth peer name veth1
   # ip link set veth0 up
   # ip link set veth1 up

|

"$ ip a"명령어로 ens5의 MAC address를 확인하여 아래 명령을 수행한다.

.. code-block:: bash

   # brctl addif br-data veth1;ifconfig br-data hw ether (ens5의 MAC address)
   ex) MAC address가 02:ae:fa:xx:88:00이라면, # brctl addif br-data veth1;ifconfig br-data hw ether 02:ae:fa:xx:88:00

|

오픈스택을 위한 브릿지 네트워크 설정을 마치고 centos 계정으로 전환한다.

.. code-block:: bash

   # su - centos

|

* 인벤토리 설정(오픈스택을 설치하는 경우)

설정에 필요한 host_ip_대역은 다음과 같이 확인한다.

.. code-block:: bash

   $ ip a
   ex) 확인된 ip가 101.101.101.11/24 라면 host_ip_대역은 101.101.101.0/24 이다.

|

설정에 필요한 additional_empty_volume은 다음과 같이 확인한다.

.. code-block:: bash

   $ lsblk
   ex) nvme1n1

|

샘플 extra-vars.yml 파일에서 아래와 같이 5가지 항목을 수정한다.

.. code-block:: bash

   $ vi ~/tacoplay/inventory/sample/extra-vars.yml
   taco_apps: ["openstack"]
   monitor_interface: br-data
   public_network: host_ip_대역(ex. 101.101.101.0/24)
   cluster_network: host_ip_대역(ex. 101.101.101.0/24)
   lvm_volumes:
     - data: /dev/addtional_empty_volum (ex. /dev/nvme1n1)

|

샘플 openstack-manifest.yaml 파일에서 아래와 같이 3가지 항목을 수정한다.

.. code-block:: bash

   $ vi ~/tacoplay/inventory/sample/openstack-manifest.yaml
   1138             physical_interface_mappings: "provider:veth0"
   1320         host_interface: br-data
   1322         live_migration_interface: br-data

|

* 인벤토리 설정(K8s만 설치하는 경우)

설정에 필요한 host_ip_대역은 다음과 같이 확인한다.

.. code-block:: bash

   $ ip a
   ex) 확인된 ip가 101.101.101.11/24 라면 host_ip_대역은 101.101.101.0/24 이다.

|

설정에 필요한 additional_empty_volume은 다음과 같이 확인한다.

.. code-block:: bash

   $ lsblk
   ex) nvme1n1

|

샘플 extra-vars.yml 파일에서 아래와 같이 5가지 항목을 수정한다.

.. code-block:: bash

   $ vi ~/tacoplay/inventory/sample/extra-vars.yml
   taco_apps: [""]
   monitor_interface: ens5
   public_network: host_ip_대역(ex. 101.101.101.0/24)
   cluster_network: host_ip_대역(ex. 101.101.101.0/24)
   lvm_volumes:
     - data: /dev/addtional_empty_volum (ex. /dev/nvme1n1)

|


* tacoplay 실행

위의 설정을 모두 했다면 한 번에 모든 구성을 배포해도 되지만, 이슈 발생을 대비하여 ceph - k8s - openstack 순서로 배포한다.

.. code-block:: bash

   $ cd ~/tacoplay/
   ##1. ceph 배포
   $ ansible-playbook -b -i inventory/sample/hosts.ini -e @inventory/sample/extra-vars.yml site.yml --tags setup-os,ceph,ceph-post-install --skip-tags k8s,lma,openstack,deploy
   
|

"$ ceph status" 명령을 통해 ceph이 잘 배포되었는지 확인하고, K8s를 배포한다.

.. code-block:: bash

   ##2. K8s 배포
   $ ansible-playbook -b -i inventory/sample/hosts.ini -e @inventory/sample/extra-vars.yml site.yml --tags ceph-post-install,k8s --skip-tags setup-os,ceph,lma,openstack,deploy 
   
|

"$ kubectl get pods -n kube-system" 명령을 통해 K8s가 잘 배포되었는지 확인한다. 오픈스택을 설치하는 경우 이어서 오픈스택을 배포한다.

.. code-block:: bash

   ##3. 오픈스택 배포
   $ ansible-playbook -b -i inventory/sample/hosts.ini -e @inventory/sample/extra-vars.yml site.yml --skip-tags setup-os,ceph,lma,k8s

|

테스트 환경 사양에 따라 배포 완료 시간이 40분에서 2시간까지 달라질 수 있다. 오픈스택 배포 중인 경우 별도의 터미널에 watch 명령을 사용하여 Completed나 Running 상태가 아닌 파드들을 모니터링할 수 있다.

.. code-block:: bash

   $ watch 'kubectl get pods -n openstack | grep -v Comp | grep -v Runn'

|

ansible-playbook이 성공적으로 종료되었다면 <host_ip>:31000 으로 웹 접속하여 openstack를 사용할 수 있다.
(로그인 정보: default / admin / password)

* ansible 로그 확인 방법
1. /tmp/ansible.log를 확인한다.
2. ansible-playbook 명령시 -vvvv 옵션을 추가하면 더 구체적인 로그가 기록된다.
3. ansible-playbook 명령시 > example_ansible_log_0 옵션을 추가하면 로그가 터미널에 출력되지 않고 파일에 기록된다. "$ tail -f example_ansible_log_0"로 모니터링 할 수 있다.

* K8s 배포 확인 방법
1. kube-system 네임스페이스를 갖는 K8s 리소스들이 잘 작동중인지 확인한다.

.. code-block:: bash

   $ kubectl get pods -n kube-system
   $ kubectl get services -n kube-system
   $ kubectl get deployments -n kube-system

|

* 오픈스택 배포 확인 방법
1. /tmp/openstack-deployment.log를 확인한다.
2. openstack 네임스페이스를 갖는 K8s 리소스들이 잘 작동중인지 확인한다.

.. code-block:: bash

   $ kubectl get pods -n openstack
   $ kubectl get pods -n openstack | grep -v Comp | grep -v Runn
   
|

3. helm chart가 정상적으로 배포되었는지 확인한다.

.. code-block:: bash

   $ helm list -a

|

오픈스택 네트워크 토폴로지 구성
=====================

* provider/private network 구성 및 라우팅

To Be Added


Trouble Shooting
================

* 오픈스택 파드가 정상적이지 못한 경우 확인 방법

1. helm 설치가 정상적인지 확인한다. helm의 설치는 tacoplay/kubespray/roles/kubernetes-apps/helm/tasks/main.yml 에서 진행된다.

.. code-block:: bash

   $ helm version
   Client: &version.Version{SemVer:"v2.16.1", GitCommit:"bbdfe5e7803a12bbdf97e94cd847859890cf4050", GitTreeState:"clean"}
   Server: &version.Version{SemVer:"v2.16.1", GitCommit:"bbdfe5e7803a12bbdf97e94cd847859890cf4050", GitTreeState:"clean"} 

|

2. 문제가 생긴 파드의 events 및 log를 살핀다.

.. code-block:: bash

   $ kubectl get pods -n openstack #문제가 생긴 pod의 이름을 확인한다.
   $ kubectl describe pods -n openstack example_pod_name
   $ kubectl logs -n openstack example_pod_name

|

* K8s의 deployment 설정이 변경되어 반영이 필요한 경우

.. code-block:: bash

   $ kubectl get deploy -n openstack ## 재배포를 원하는 deployment명 확인
   $ kubectl scale deploy -n openstack example_deploy_name --replicas=0
   $ kubectl scale deploy -n openstack example_deploy_name --replicas=1

|

* 오픈스택 파드 중 문제가 된 차트를 지우고 재배포 하는 방법

배포되는 오픈스택 helm charts는 네 개의 그룹으로 이루어져 아래와 같이 순차적으로 배포된다.

.. code-block:: bash

   ## Chart_Group : openstack-infra
   ceph-provisioners
   ingress
   memcached
   rabbitmq
   mariadb

   ## Chart_Group : openstack-base
   keystone
   glance
   cinder

   ## Chart_Group : openstack-compute-kit
   libvirt
   openvswitch
   nova
   neutron

   ## Chart_Group : openstack-addon
   horizon
   heat

|

같은 chart_group 내에 있는 차트들은 서로 디펜던시를 갖기 때문에 문제가 된 차트를 지우고 새로 배포하기 위해선 해당 차트가 속한 그룹 이후에 설치되는 모든 차트를 지워야 한다.

.. code-block:: bash

   $ kubectl get pods -n openstack ## 문제가 된 파드가 어떤 차트에 속하는지 확인한다. ex) nova
   $ helm list -a ## 지워야할 helm chart의 이름을 확인한다. ex) single-nova
   $ helm delete single-libvirt single-openvswitch single-nova single-neutron single-horizon single-heat --purge ## nova에서 문제가 생겼다면 nova를 포함하는 차트 그룹 이후의 모든 차트를 삭제해준다.
 
|

openstack-manifest.yaml을 위에서 삭제한 Chart_Group만 배포되도록 아래와 같이 수정하고 ansible을 실행한다.

.. code-block:: bash

   ex) infra와 base 차트 그룹은 설치가 되어 있는 경우
   $ vi ~/tacoplay/inventory/sample/openstack-manifest.yaml
   849   chart_groups:
   850   #- openstack-infra ## 설치하지 않도록 #으로 주석처리한다.
   851   #- openstack-base  ## 설치하지 않도록 #으로 주석처리한다.
   852   - openstack-compute-kit
   853   - openstack-addon

   $ ansible-playbook -b -i inventory/sample/hosts.ini -e @inventory/sample/extra-vars.yml site.yml --skip-tags setup-os,ceph,lma,k8s

|

* horizon 파드가 ready 상태가 되지 못하고 restart가 반복될 때

To Be Added

