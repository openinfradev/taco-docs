***********************
TACO install - aio node
***********************

ceph 용 볼륨 추가
=================

.. code-block:: bash

  $ lsblk

tacoplay 설정
=============

* Git 받아오기

.. code-block:: bash

   $ sudo yum install -y git
   $ git clone https://tde.sktelecom.com/stash/scm/oreotools/tacoplay.git
   $ cd tacoplay/

* tacoplay/VERSION 파일 수정

.. code-block:: bash

   $ vi VERSIONS
   armada https://github.com/openinfradev/armada.git taco-v19.02
   kubespray https://github.com/kubernetes-sigs/kubespray.git v2.10.3
   charts/openstack-helm https://github.com/openinfradev/openstack-helm.git taco-v19.02
   charts/openstack-helm-infra https://github.com/openinfradev/openstack-helm-infra.git taco-v19.02
   ceph-ansible https://github.com/openinfradev/ceph-ansible.git stable-3.2
   charts/helm https://github.com/helm/charts.git master

* 하위 프로젝트 fetch
  
.. code-block:: bash

   $ ./fetch-sub-projects.sh

* ceph-ansible site.yml 생성

.. code-block:: bash

   $ cp ceph-ansible/site.yml.sample ceph-ansible/site.yml

* host.ini & extra-vars.yml 수정  (경로: tacoplay/inventory/sample)
   
   * host.ini
      * instance's ip 확인 필요
      * https://tde.sktelecom.com/stash/projects/TACODEV/repos/openinfraday2019/browse/inventories/aio/host.ini
   * extra-vars.yml
      * monitor_interface, public_network, cluster_network, ceph_monitors, lvm_molumes 확인 필요
      * https://tde.sktelecom.com/stash/projects/TACODEV/repos/openinfraday2019/browse/inventories/aio/extra-vars.yml

OS 설정
=======

* 호스트 파일 설정: admin 노드

..code-block:: bash

   $ sudo vi /etc/hosts
   ## TACO ClusterInfo
   127.0.0.1   taco-aio

TACO 설치
=========

* TACO playbook 실행에 필요한 패키지 설치 : admin 노드

..code-block:: bash

   # admin 노드에서 실행
   cd ~/tacoplay
   sudo yum install -y selinux-policy-targeted
   sudo yum install -y bridge-utils
   sudo yum install -y epel-release
   sudo yum install python-pip -y
   sudo pip install --upgrade pip==9.0.3
   sudo pip install -r ceph-ansible/requirements.txt
   sudo pip install -r kubespray/requirements.txt --upgrade
   sudo pip install -r requirements.txt --upgrade

* armada-manifest 받아오기

..code-block :: bash

   $ cd ~/tacoplay
   $ git clone https://tde.sktelecom.com/stash/scm/tacodev/openinfraday2019.git manifest
   $ cp manifest/inventories/aio/* inventory/sample

* host.ini & extra-vars.yml 수정  (경로: tacoplay/inventory/sample)
   * host.ini
      * instance's ip 확인 필요
      * https://tde.sktelecom.com/stash/projects/TACODEV/repos/openinfraday2019/browse/inventories/aio/hosts.ini
   * extra-vars.yml
      * monitor_interface, public_network, cluster_network, ceph_monitors, lvm_molumes 확인 필요
      * https://tde.sktelecom.com/stash/projects/TACODEV/repos/openinfraday2019/browse/inventories/aio/extra-vars.yml

OS 설정
=======

* 호스트 파일 설정: admin 노드

..code-block :: bash
   
   cd ~/tacoplay
   pip install yq
   sudo yum install -y selinux-policy-targeted
   sudo yum install -y bridge-utils
   sudo yum install -y epel-release
   sudo yum install python-pip -y
   sudo pip install --upgrade pip==9.0.3
   sudo pip install -r ceph-ansible/requirements.txt
   sudo pip install -r kubespray/requirements.txt --upgrade
   sudo pip install -r requirements.txt --upgrade

* 네트워크 생성 
  
..code-block :: bash

   #!/bin/bash
   # Virtual Interfaces
   ip link add veth0-ex type veth peer name veth1-ex
   ip link add veth0-int type veth peer name veth1-int
   ip link add veth0-tun type veth peer name veth1-tun
   ip addr add 10.10.10.1/24 dev veth0-ex
   ip addr add 172.30.1.30/24 dev veth0-int
   ip addr add 192.168.20.1/24 dev veth0-tun
   # NAT
   iptables -A FORWARD -o veth0-ex -j ACCEPT
   iptables -A FORWARD -o bond0 -j ACCEPT
   iptables -t nat -A POSTROUTING -o bond0 -j MASQUERADE

이 후, armada-manifest에 veth0-ex, veth0-tun 추가

* Taco 설치

..code-block :: bash

   $ cd ~/tacoplay
   $ ansible-playbook -b -i inventory/sample/hosts.ini -e @inventory/sample/extra-vars.yml site.yml

TACO 설치 확인
==============

* Key 생성

..code-block :: bash

   $ ssh-keygen -t rsa

* 설치 확인

..code-block :: bash

   $ cd ~/tacoplay
   $ tests/taco-test.sh

Trouble Shoothing
=================

* openstack client version issue

..code-block :: bash

   $ sudo pip install --ignore-installed python-openstackclient==3.14.3
   $ . tacoplay/tests/adminrc

* openstack cannot import decorate

..code-block :: bash

   $ sudo pip install --upgrade decorator

* openstack client version 

tacoplay/roles/openstack/client/task/main.yml

..code-block :: bash

   - name: install python-openstackclient
     pip:
       name: "{{ item.name }}"
       version: "{{ item.version }}"
       state: present
     loop:
       - { name: 'pbr', version: '5.1.1' }
       - { name: 'python-openstackclient', version: '3.14.3' }
       - { name: 'python-cinderclient', version: '3.5.0' }
       - { name: 'python-glanceclient', version: '2.10.1' }
       - { name: 'python-keystoneclient', version: '3.15.0' }
       - { name: 'python-novaclient', version: '10.1.0' }
       - { name: 'python-neutronclient', version: '6.7.0' }

* Missing value auth-url required for auth plugin password

   로그아웃하고 다시 로그인하자.

* image queued issue (TACO 설치 후 openstack image를 확인했을 때 queued 상태의 이미지가 있을 때)

..code-block :: bash

   #!/bin/bash
   rbd ls images
   rbd -p images snap unprotect --image 201084fc-c276-4744-8504-cb974dbb3610 --snap snap
   rbd snap purge 201084fc-c276-4744-8504-cb974dbb3610 -p images
   rbd -p images rm 201084fc-c276-4744-8504-cb974dbb3610
