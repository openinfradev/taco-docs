###########################
Adding nodes with Kubespray
###########################

새로운 노드 추가 절차
*********************

/etc/hosts 업데이트
===================

Ansible 호스트의 /etc/hosts에 새로운 노드의 ansible SSH 접속 IP를 추가한다. 보통 매니지먼트 네트워크와 동일하다.
아래는 com-3, com-4, com-5 노드를 추가하는 예시다.

.. code-block:: shell

   [centos@deployer tacoplay]$ sudo sed -i -e "\$a 172.80.0.17 com-3" /etc/hosts
   [centos@deployer tacoplay]$ sudo sed -i -e "\$a 172.80.0.18 com-4" /etc/hosts
   [centos@deployer tacoplay]$ sudo sed -i -e "\$a 172.80.0.19 com-5" /etc/hosts

SSH key 복사
============

Ansible 호스트에서 새로 추가된 노드로 password 없이 ssh 접근이 가능하도록 설정한다.

.. code-block:: shell

   [centos@deployer tacoplay]$ ssh-copy-id com-3
   [centos@deployer tacoplay]$ ssh-copy-id com-4
   [centos@deployer tacoplay]$ ssh-copy-id com-5

인벤토리 파일 업데이트
======================

인벤토리 파일에 새로 추가된 노드 정보를 넣어 준다.

.. code-block:: shell

   [centos@deployer tacoplay]$ vi inventory/sample/hosts.ini
   # ## Configure 'ip' variable to bind kubernetes services on a
   # ## different ip than the default iface
   master-1 ip=172.80.0.9
   master-2 ip=172.80.0.11
   master-3 ip=172.80.0.6
   ctrl-1 ip=172.80.0.10
   ctrl-2 ip=172.80.0.5
   ctrl-3 ip=172.80.0.4
   com-1 ip=172.80.0.7
   com-2 ip=172.80.0.8
   # 추가할 노드들의 이름과 매니지먼트 IP 추가
   com-3 ip=172.80.0.17
   com-4 ip=172.80.0.18
   com-5 ip=172.80.0.19
    
   [kube-node]
   ctrl-1
   ctrl-2
   ctrl-3
   com-1
   com-2
   # 추가할 노드의 이름 추가
   com-3
   com-4
   com-5
    
   [compute-node]
   com-1
   com-2
   # 추가할 노드의 이름 추가
   com-3
   com-4
   com-5
    
   # 추가할 노드 수가 많은 경우 추가할 노드들로 구성된 그룹 추가
   # scale.yml playbook 실행 시, limit 인자 값으로 새로 추가된 노드들만 넘길 때 유용하다
   [new-nodes]
   com-3
   com-4
   com-5

Scale playbook 실행
===================

.. code-block:: shell

   [centos@deployer tacoplay]$ ansible-playbook -b -i inventory/sample/hosts.ini scale.yml --limit etcd,new-nodes


