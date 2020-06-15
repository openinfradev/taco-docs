***************************************
TACO 2.0 AIO 설치 가이드 (Kubernetes & Ceph & OpenStack Deployment)
***************************************

이 매뉴얼은 AWS에서 생성한 EC2 환경을 기준으로 작성되었으며, Kubernetes, Ceph, OpenStack을 하나의 VM에 모두 설치하는 AIO (All-In-One) 가이드이다.

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

* 포트 개방 : 80, 30608, 31000 (+LMA 설치시 30001, 30009, 30018, 32000)

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
   $ sudo yum install -y bridge-utils epel-release git
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

브릿지 네트워크 구성
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

AIO node가 보유한 IP 자원을 오픈스택 위에 생성될 VM에게 할당해주기 위해서 브릿지 네트워크를 구성해야 한다. 이를 위해 필요한 로컬 정보를 아래의 방법으로 확인한다.

* { ethernet_interface }, { interface_MAC }, { host_ip }

{ ethernet_interface } : ens5   <--아래 출력된 결과의 7번째 줄, AWS에서 생성한 인스턴스가 아닌 경우 eth0 등의 이름으로 출력될 수 있다.

{ interface_MAC } : 02:ae:fa:f2:88:84   <--아래 출력된 결과의 8번째 줄

{ host_ip } : 172.32.0.81   <--아래 출력된 결과의 9번째 줄

.. code-block:: bash

   $ ip a
   ##(example)
   1  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
   2      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
   3      inet 127.0.0.1/8 scope host lo
   4         valid_lft forever preferred_lft forever
   5      inet6 ::1/128 scope host
   6         valid_lft forever preferred_lft forever
   7  2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
   8      link/ether 02:ae:fa:f2:88:84 brd ff:ff:ff:ff:ff:ff
   9      inet 172.32.0.81/24 brd 172.32.0.255 scope global dynamic ens5
   10        valid_lft 3520sec preferred_lft 3520sec
   11    inet6 fe80::ae:faff:fef2:8884/64 scope link
   12        valid_lft forever preferred_lft forever

|

* { gateway } : 172.32.0.1   <--아래 "ip route"을 통해 알 수 있다.

.. code-block:: bash

   $ ip route
   ##(example)
   default via 172.32.0.1 dev ens5
   172.32.0.0/24 dev ens5 proto kernel scope link src 172.32.0.81
|

* 브릿지 네트워크 생성

위에서 확인한 값을 바탕으로 br-data라는 이름의 브릿지 네트워크 생성을 시작한다. 사용되는 계정은 root 이며, '#'로 시작하는 커맨드는 root 계정에서 수행함을 의미한다. '##'로 시작하는 것은 주석을 의미한다.

.. code-block:: bash

   $ sudo su -
   # cd /etc/sysconfig/network-scripts/

   ##{ } 안에 알맞은 값을 대입하여 아래 설정을 ifcfg-{ ethernet_interface }에 저장한다
   # vi ifcfg-{ ethernet_interface }   ##should be edited
   DEVICE={ ethernet_interface }   ##should be edited
   ONBOOT=yes
   TYPE=Ethernet
   BRIDGE=br-data
   BOOTPROTO=none
   NM_CONTROLLED=no

   ##{ } 안에 알맞은 값을 대입하여 아래 설정을 ifcfg-br-data에 저장한다
   # vi ifcfg-br-data
   BOOTPROTO=none
   DEFROUTE=yes
   DEVICE=br-data
   GATEWAY={ gateway }   ##should be edited
   IPADDR={ host_ip }   ##should be edited
   NETMASK=255.255.255.0
   ONBOOT=yes
   TYPE=Bridge
   STP=no
   NM_CONTROLLED=no
|

위에서 확인한 값을 바탕으로 설정한 내용을 반영한다.

.. code-block:: bash

   # systemctl restart network
   # ip link add veth0 type veth peer name veth1
   # ip link set veth0 up
   # ip link set veth1 up

   ##{ } 안에 알맞은 값을 대입하여 아래 명령을 수행한다. 두 명령을 ';'을 통해 연속적으로 수행하지 않으면 ssh 접속이 끊길 수 있으니 주의한다.
   # brctl addif br-data veth1;ifconfig br-data hw ether { interface_MAC }   ##should be edited

|

브릿지 네트워크 구성을 마쳤다면 centos 계정으로 전환한다.

.. code-block:: bash

   # su - centos

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

* { host_ip } : 172.32.0.81   <-- 아래 출력된 결과의 9번째 줄에서 확인 가능. 브릿지 네트워크를 구성한 경우에는 네트워크 구성 단계에서 확인한 { host_ip }(혹은 br-data가 갖고 있는 ip)를 사용한다.

* { network_cidr } : 172.32.0.0/24   <--아래에서 출력된 9번째 줄에서 확인 가능한 172.32.0.81/24의 네 번째 옥텟을 0으로 바꾼 값. 브릿지 네트워크를 구성한 경우에는 네트워크 구성 단계에서 확인한 { host_ip }(혹은 br-data가 갖고 있는 ip)를 통해 구한다.

.. code-block:: bash

   $ ip a
   ##(example)br-data 구성하기 전
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

브릿지 네트워크를 구성하였다면 아래와 비슷한 결과가 출력될 것이다. 이때 { host_ip }와 { network_cidr }은 br-data의 것을 참고한다.

.. code-block:: bash

   $ ip a
   ##(example)br-data 구성한 후
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
       inet 127.0.0.1/8 scope host lo
          valid_lft forever preferred_lft forever
       inet6 ::1/128 scope host
          valid_lft forever preferred_lft forever.
   2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq master br-data state UP group default qlen 1000
       link/ether 02:ae:fa:f2:88:84 brd ff:ff:ff:ff:ff:ff
       inet6 fe80::ae:faff:fef2:8884/64 scope link
          valid_lft forever preferred_lft forever
   3: br-data: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
       link/ether 02:ae:fa:f2:88:84 brd ff:ff:ff:ff:ff:ff
       inet 172.32.0.81/24 brd 172.32.0.255 scope global br-data
          valid_lft forever preferred_lft forever
       inet6 fe80::ae:faff:fef2:8884/64 scope link
          valid_lft forever preferred_lft forever
   4: veth1@veth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-data state UP group default qlen 1000
       link/ether 1e:88:df:ce:3a:43 brd ff:ff:ff:ff:ff:ff
       inet6 fe80::1c88:dfff:fece:3a43/64 scope link
          valid_lft forever preferred_lft forever
   5: veth0@veth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
       link/ether 06:12:1a:0a:65:25 brd ff:ff:ff:ff:ff:ff
       inet6 fe80::412:1aff:fe0a:6525/64 scope link
          valid_lft forever preferred_lft forever

|

* 인벤토리 설정

제공된 sample extra-vars.yml 파일에서 아래와 같이 5가지 항목의 value를 수정한다.

.. code-block:: bash

   ##{ } 안에 알맞은 값을 대입하여 아래 설정을 extra-vars.yml에 저장한다.
   $ vi ~/tacoplay/inventory/sample/aio/extra-vars.yml
   taco_apps: ["openstack"]
   monitor_interface: br-data
   public_network: { network_cidr }   ##should be edited
   cluster_network: { network_cidr }   ##should be edited
   lvm_volumes:
     - data: /dev/{ Additional_Empty_Volume }   ##should be edited

|


* (optional) LMA (Logging, Monitoring, Alerting) 설치를 위한 인벤토리 설정

LMA를 설치하면 TACO가 관리하는 리소스의 로그와 사용 현황을 확인할 수 있는 대쉬보드가 제공된다. LMA를 구성하는 소프트웨어 컴포넌트는 다음과 같다.

   * Prometheus
   * Grafana
   * Fluentbit
   * Elasticsearch
   * Kibana

제공된 샘플 extra-vars.yml 에서 아래와 같이 1가지 항목의 value를 수정한다.

.. code-block:: bash

   ##taco_apps의 value에 "lma"를 추가하면 자동으로 LMA를 설치한다.
   $ vi ~/tacoplay/inventory/sample/aio/extra-vars.yml
   taco_apps: ["openstack","lma"]

|

제공된 샘플 lma-manifest.yaml 에서 총 14개의 FIXME 주석이 표시된 항목들을 알맞게 수정해준>다.

.. code-block:: bash

   $ vi ~/tacoplay/inventory/sample/aio/lma-manifest.yaml
   #Prometheus의 볼륨 크기를 정해준다.
   273                   storage: 200Gi # FIXME

   #K8s-master의 host_ip, 위에서 사용한 { host_ip }를 사용한다.
   278       - ENTER_YOUR_MASTER1_IP    # FIXME

   #Federated master Prometheus의 볼륨 크기를 정해준다.
   466                   storage: 500Gi # FIXME

   #Elasticsearch 인스턴스 갯수를 정해준다.
   607         count: 1 # FIXME

   #Elasticsearch의 볼륨 크기를 정해준다.
   618                 storage: 2000Gi          # FIXME

   #Grafana의 볼륨 크기를 정해준다.
   686       size: 10G  # FIXME

   #Elasticsearch URL, AIO 설치에선 변경하지 않고 그대로 사용한다.
   867         host: "https://taco-elasticsearch-es-http:9200"          # FIXME

   #Datasource Prometheus URL, AIO 설치에선 변경하지 않고 그대로 사용한다.
   874         hosts: ["lma-prometheus-fed-master-prometheus.fed.svc.cluster.local:9090"]       # FIXME

   #Ceph-monitor의 host_ip, 위에서 사용한 { host_ip }를 사용한다.
   891         - ip: ENTER_YOUR_CEPH_MONITOR_IP # FIXME

   #Elasticsearch URL, AIO 설치에선 변경하지 않고 그대로 사용한다.
   926         host: "https://taco-elasticsearch-es-http.lma.svc.cluster.local:9200"    # FIXME

   #Kibana URL, AIO 설치에선 변경하지 않고 그대로 사용한다.
   930         host: "taco-kibana-dashboard-kb-http.lma.svc.cluster.local:5601"         # FIXME

   #Datasource Prometheus URL, AIO 설치에선 변경하지 않고 그대로 사용한다.
   933         hosts: ["lma-prometheus-fed-master-prometheus.fed.svc.cluster.local:9090"]       # FIXME

   #K8s-master의 host_ip, 위에서 사용한 { host_ip }를 사용한다.
   956         address: ENTER_YOUR_MASTER1_IP   # FIXME

   #Kibana URL, AIO 설치에선 변경하지 않고 그대로 사용한다.
   970       url: "http://taco-kibana-dashboard-kb-http.lma.svc.cluster.local:5601"     # FIXME

|

tacoplay 실행
^^^^^^^^^^^^

위의 설정을 모두 마쳤다면 tacoplay를 실행한다.

.. code-block:: bash

   $ cd ~/tacoplay/
   $ ansible-playbook -b -i inventory/sample/aio/hosts.ini -e @inventory/sample/aio/extra-vars.yml site.yml

|

테스트 환경 사양에 따라 배포 완료 시간이 40분에서 2시간까지 달라질 수 있다. 오픈스택 배포가 시작되면 "TASK [taco-apps/deploy : deploy apps using 'armada apply']"에서 한동안 ansible 로그가 출력되지 않는데, 별도의 터미널에서 watch 명령을 사용하면 배포 중인 파드들을 모니터링할 수 있다. LMA의 배포를 모니터링하는 것도 마찬가지이다.

.. code-block:: bash

   $ watch 'kubectl get pods -n openstack'   ##openstack을 구성하는 모든 파드를 모니터링
   $ watch 'kubectl get pods -n openstack | grep -v Comp'   ##Completed 상태인 파드를 제외하고 모니터링
   $ watch 'kubectl get pods -n openstack | grep -v Comp | grep -v Runn'   ##Completed 혹은 Running 상태인 파드를 제외하고 모니터링

   $ watch 'kubectl get pods -n lma'   ##lma를 구성하는 모든 파드를 모니터링
   $ watch 'kubectl get pods -n fed'   ##fed(fedaration) 관련 파드를 모니터링
   
   $ watch 'kubectl get pods -A'   ##모든 K8s 파드를 모니터링(kube-system, openstack, lma, fed)

|

모든 Running 파드가 ready 상태가 되면 ansible은 곧 종료된다. 만약 openstack 네임스페이스의 horizon 파드가 ready 되지 못하고 restart가 반복된다면 해당 문서의 Trouble Shooting을 참고한다.


* 오픈스택 설치 확인

웹 브라우저를 통해 { host_ip }:31000 접속하여 openstack 웹 콘솔이 나타나는지 확인한다.

.. figure:: _static/horizon.png

| domain : default
| id : admin
| pw : password

|

* LMA 접속

LMA를 설치한 경우 아래 접속 정보를 참고하여 웹 브라우저로 접속해본다.

   * Kibana : http://{ host_ip }:30001/
   아이디 / 패스워드 : elastic / tacoword

   * Grafana : http://{ host_ip }:30009/
   아이디 / 패스워드 : admin / password
   

* 오픈스택 VM 생성을 위한 네트워크 토폴로지 구성

네트워크를 구성해주어야 오픈스택에서 인스턴스를 생성할 수 있다. 다음은 centos 계정에 생성된 os_client를 통해 예시 네트워크를 구성하는 절차이다.

.. code-block:: bash

   $ sh /home/centos/os_client.sh
   ~$ openstack network create private-net
   ~$ openstack subnet create --network private-net --subnet-range 172.30.1.0/24 --dns-nameserver 8.8.8.8 private-subnet

   ~$ openstack network create --external --share --provider-network-type flat --provider-physical-network provider provider-net
   ~$ openstack subnet create --network provider-net --subnet-range 192.168.97.0/24 --dns-nameserver 8.8.8.8 provider-subnet --allocation-pool 192.168.97.122=192.168.97.122,192.168.97.46=192.168.97.46,192.168.97.231=192.168.97.231,192.168.97.115=192.168.97.115


   ~$ openstack network create --external --share --provider-network-type flat --provider-physical-network provider provider-net
   ~$ openstack subnet create --network provider-net --subnet-range 192.168.97.0/24 --dns-nameserver 8.8.8.8 provider-subnet--allocation-pool 192.168.97.91=192.168.97.91,192.168.97.70=192.168.97.70,192.168.97.31=192.168.97.31,192.168.97.182=192.168.97.182

   ~$ openstack network create --external --share --provider-network-type flat --provider-physical-network provider provider-net
   ~$ openstack subnet create --network provider-net --subnet-range 192.168.97.0/24 --dns-nameserver 8.8.8.8 provider-subnet--allocation-pool 192.168.97.52=192.168.97.52,192.168.97.206=192.168.97.206,192.168.97.192=192.168.97.192,192.168.97.13=192.168.97.13  


   ~$ openstack router create admin-router
   ~$ openstack router add subnet admin-router private-subnet
   ~$ openstack router set --external-gateway provider-net admin-router
   ~$ openstack router show admin-router

   ~$ openstack security group list --project admin | grep default | awk '{print $2}'
   ##출력되는 값을 { security_group } 이라고 하자

   ##{ }안에 알맞은 값을 대입하여 명령을 실행한다.
   ~$ openstack security group rule create --proto tcp --remote-ip 0.0.0.0/0 --dst-port 1:65535 --ingress { security_group }
   ~$ openstack security group rule create --protocol icmp --remote-ip 0.0.0.0/0 { security_group }
   ~$ openstack security group rule create --protocol icmp --remote-ip 0.0.0.0/0 --egress { security_group }


|

네트워크를 구성했다면 { host_ip }:31000 으로 접속하여 Compute > 인스턴스 탭에서 인스턴스를 추가할 수 있다. 제공되는 cirros 이미지를 사용하여 인스턴스를 생성했다면, 인스턴스명을 클릭하여 콘솔탭으로 접근한다. cirros의 default 로그인 정보는 cirros / gocubsgo 이다.(콘솔이 정상적으로 열리지 않는다면 웹페이지 새로고침을 반복한다.)

Trouble Shooting
================

* ansible 로그 확인 방법
1. 디폴트로 생성되는 로그는 /tmp/ansible.log를 확인한다. 로그를 별도로 관리하고자 한다면 '> example_file.log_0' 옵션을 붙여 로그를 원하는 파일에 생성할 수 있다.
2. ansible-playbook 명령 시 -vvvv 옵션을 추가하면 더 구체적인 로그가 기록된다.

* ansible 설치 중에 문제가 발생하여 재설치할 때 tag를 이용하여 일부 role만 수행하는 방법
tacoplay 실행 시 tacoplay/site.yml에 작성되어 있는 role의 순서대로 설치가 진행된다. 설치는 크게 보았을 때 ceph - K8s - taco_app(오픈스택 및 LMA) 순으로 진행된다. 이를 부분적으로 설치하고 싶다면 아래 명령을 수행하면 된다.

.. code-block:: bash

   ##1. 초기 세팅 및 ceph의 설치를 진행하는 커맨드(ceph이 이미 설치된 경우 에러가 발생할 수 있으니 주의한다.)
   $ ansible-playbook -b -i inventory/sample/aio/hosts.ini -e @inventory/sample/aio/extra-vars.yml site.yml --tags setup-os,ceph,ceph-post-install --skip-tags k8s
   
|

.. code-block:: bash

   ##2. ceph이 정상적으로 설치되었을 때, K8s를 설치하는 커맨드(ceph을 중복으로 설치하게 되면 문제가 발생하여 스킵해준다)
   $ ansible-playbook -b -i inventory/sample/aio/hosts.ini -e @inventory/sample/aio/extra-vars.yml site.yml --tags ceph-post-install,k8s,taco-clients --skip-tags ceph

|

.. code-block:: bash

   ##3. K8s까지 정상적으로 설치되었을 때, taco_app(오픈스택 및 LMA)의 배포 혹은 남은 role을 수행하는 커맨드
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

* 오픈스택 설치 관련 문제 발생 시
1. 설치 로그는 /tmp/openstack-deployment.log를 확인한다.
2. openstack 네임스페이스를 갖는 K8s 리소스들이 잘 작동 중인지 확인한다.

.. code-block:: bash

   $ kubectl get pods -n openstack
   
|

3. 문제가 생긴 파드가 있다면 events 및 log를 살핀다.

.. code-block:: bash

   $ kubectl get pods -n openstack   ##문제가 생긴 pod의 이름을 확인한다.
   $ kubectl describe pods -n openstack example_pod_name
   $ kubectl logs -n openstack example_pod_name

|

4. helm 설치가 정상적인지 확인한다. helm의 설치는 tacoplay/kubespray/roles/kubernetes-apps/helm/tasks/main.yml 에서 진행된다.

.. code-block:: bash

   $ helm version
   Client: &version.Version{SemVer:"v2.16.1", GitCommit:"bbdfe5e7803a12bbdf97e94cd847859890cf4050", GitTreeState:"clean"}
   Server: &version.Version{SemVer:"v2.16.1", GitCommit:"bbdfe5e7803a12bbdf97e94cd847859890cf4050", GitTreeState:"clean"} 

|

5. helm chart가 정상적으로 배포되었는지 확인한다.

.. code-block:: bash

   $ helm list -a

|

* horizon 파드가 ready 상태가 되지 못하고 restart가 반복될 때

사용된 single host 인스턴스의 사양이 낮은 경우 horizon 파드 내 compress 관련 role이 늦게 처리되어 문제가 발생할 수 있다. 아래 명령을 통해 horizon deployment 설정에 접근하여 'initialDelaySeconds' 항목 2개를 찾아 value를 180으로, 'periodSeconds' 항목 2개를 찾아 value를 600으로 변경해준다.

.. code-block:: bash

   $ kubectl edit deployment -n openstack horizon
     86         livenessProbe:
     87           failureThreshold: 3
     88           httpGet:
     89             path: /
     90             port: 80
     91             scheme: HTTP
     92           initialDelaySeconds: 180   ##HERE
     93           periodSeconds: 600   ##HERE
     94           successThreshold: 1
     95           timeoutSeconds: 5

    101         readinessProbe:
    102           failureThreshold: 3
    103           httpGet:
    104             path: /
    105             port: 80
    106             scheme: HTTP
    107           initialDelaySeconds: 180   ##HERE
    108           periodSeconds: 600   ##HERE
    109           successThreshold: 1
    110           timeoutSeconds: 1

|
