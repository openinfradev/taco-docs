**************
Airship-armada
**************

개념
====

* 오픈소스 툴을 연동하여 인프라를 구성하는 `Airship프로젝트 <https://openinfradev.github.io/AirshipIntro/>`_ 의 Subproject
* 여러 개의 helm chart를 그룹으로 묶어 설치 및 업그레이드 할 수 있도록 만든 orchestrator
* 하나의 armada manifest 파일로 여러 차트와 관련된 설정들을 관리
* 복잡한 마이크로 서비스 형태로 구성된 오픈스택 서비스들을 openstack-helm으로 패키징 하고, 이를 선언적으로 관리하고 오케스트레이션 할 수 있도록 함
* https://github.com/openstack/airship-armada

기능
====

* apply: armada manifest에 정의된 차트 배포
* delete: armada manifest에 정의된 차트 삭제
* rollback: armada manifest에 정의된 차트 롤백
* test: armada manifest 파일로 helm test 호출 
* tiller: release 목록 조회 및 tiller 정보
* validate : armada manifest 검사

armada apply
------------

우리가 주로 사용할 기능은 "armada apply"로, "helm install"을 통해서 각각 배포하던 chart들을 한꺼번에 배포 가능

"armada apply"를 사용할 때 배포하려는 chart가 기존에 존재하는 경우

* 기존의 helm chart를 "helm upgrade" 명령을 통해서 chart를 upgrade 하며
* 기존에 배포된 helm chart와 배포하려는 armada-manifest 내의 chart를 비교하여, chart에 수정사항이 존재하지 않는 경우에는 upgrade 하지 않음

결국 우리는 1) armada manifest yaml 파일을 잘 만들어서, 2) armada apply를 통해 배포되어야 할 모든 chart를 배포하고, 3) 테스트를 수행

Armada manifest
===============

주요 구조
---------

* schema

  * chart: armada manifest의 기본 단위로서 helm chart 1개를 의미 
  * chartGroup: chart들을 group별로 모아놓은 내용
  * Manifest

* chartGroup data

  * description
  * sequenced: chart들을 순서대로 배포할 것인지, 한꺼번에 다 배포할 것인지 결정, 순서대로 배포하면 1개의 chart가 모두 Running 상태가 되고 나서 다음 chart를 배포
  * chart_group: 해당 chartGroup에 포함될 chart의 목록

* chart data

  * chartname
  * namespace
  * install: "helm install" 명령과 관련된 내용을 기술
  * upgrade: "helm upgrade" 명령과 관련된 내용을 기술
  * values: chart의 values.yaml 중 수정한 내용
  * source: 배포할 chart의 타입, 위치에 대한 정보
  * dependencies

예시)

.. code-block:: yaml

   chart_name: glance
   release: glance
   namespace: openstack
   install:
     no_hooks: false
   upgrade:
     no_hooks: false
     pre:
       delete:
         - name: glance-bootstrap
           type: job
           labels:
             application: glance
             component: bootstrap
         - name: glance-storage-init
           type: job
           labels:
             application: glance
             component: storage-init
   ...
   values:
     storage: rbd
     images:
       tags:
         test: admin:5000/pike/ubuntu-source-rally:3.6.0
         glance_storage_init: admin:5000/ceph-config-helper:v1.10.3
         db_init: admin:5000/pike/ubuntu-source-heat-engine:3.6.0
   ...
   source:
     type: local
     location: "/home/centos/tacoplay/charts/openstack-helm"
     subpath: glance
   dependencies:
     - helm-toolkit

armada in TACO
==============

TACO armada-manifest 주요 구조
------------------------------

* chartGroup

  * openstack-infra

    * ceph-provisioners
    * ingress
    * etcd
    * rabbitmq
    * memcached
    * mariadb

  * openstack-services

    * libvirt
    * openvswitch
    * keystone
    * glance
    * cinder
    * heat
    * nova
    * neutron
    * horizon

  * logging-infra

    * ldap
    * elasticsearch

  * monitoring-infra

    * grafana
    * prometheus
    * prometheus-alertmanager
    * prometheus-kube-state-metrics
    * prometheus-node-exporter
    * prometheus-openstack-exporter

armada apply with tacoplay
--------------------------

.. code-block:: shell

   $ ansible-playbook -b -v -i inventory/new_env/hosts.ini -e @inventory/new_env/extra-vars.yml armada-apply.yml

로그 확인
---------

.. code-block:: shell

   $ cat ~/armada.log
