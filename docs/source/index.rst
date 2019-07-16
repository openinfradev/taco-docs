.. taco-docs documentation master file, created by
   sphinx-quickstart on Fri Jun 28 14:18:13 2019.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Welcome to taco documentation!
=====================================
Welcome to the documentation of TACO, the cloud native infrastructure
delivery and lifecycle management technology based on various open source
software projects: Airship, OpenStack-Helm, Kubernetes, Helm, Ansible, etc.
If you visit here for the first time, we recommend that you read the
:ref:`introduction page <doc_about_intro>` to get an overview of what this
documentation has to offer.

The table of contents below and in the sidebar should let you easily access the
documentation for your topic of interest.

.. note:: 현재 TACO는 Latest Stable Version인 v19.02 소스코드들과 이에 대한
          기본적인 문서들이 Apache 2.0 라이선스로 공개되어 있읍니다.
          앞으로, 관심있는 분들이 직접 디자인과 개발에 참여할 수 있도록
          오픈소스 SW 프로젝트화 하는 단계들을 점차적으로 진행해나갈 예정입니다.

The main documentation for the site is organized into the following sections:

.. toctree::
   :maxdepth: 1
   :caption: General
   :name: taco-general

   about/index


.. toctree::
   :maxdepth: 2
   :caption: First Step
   :name: taco-firststep

   intro/aio.rst


.. toctree::
   :maxdepth: 2
   :caption: TACOPLAY
   :name: taco-tacoplay

   tacoplay/tacoplay.rst


.. toctree::
   :maxdepth: 1
   :caption: Sub Project
   :name: taco-subproject

   tacoplay/kubespray.rst
   tacoplay/ceph-ansible.rst
   tacoplay/openstack-helm.rst
   tacoplay/armada.rst


.. toctree::
   :maxdepth: 1
   :caption: TACO 구조 및 기술
   :name: taco-architecture
   :glob:

   architecture/*
