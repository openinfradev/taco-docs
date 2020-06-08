.. taco-docs documentation master file, created by
   sphinx-quickstart on Fri Jun 28 14:18:13 2019.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Welcome to taco documentation!
=====================================

.. toctree::
   :maxdepth: 2
   :caption: First Step

   intro/aio-k8s.rst
   intro/aio-ops.rst

.. toctree::
   :maxdepth: 2
   :caption: TACOPLAY

   tacoplay/tacoplay.rst

.. toctree::
   :maxdepth: 2
   :caption: Admin guide
   :glob:

   operation/*

.. toctree::
   :maxdepth: 1
   :caption: Sub-project

   tacoplay/kubespray.rst
   tacoplay/ceph-ansible.rst
   tacoplay/openstack-helm.rst
   tacoplay/armada.rst

.. toctree::
   :maxdepth: 1
   :caption: TACO 구조 및 기술
   :glob:

   architecture/*

.. toctree::
   :maxdepth: 1
   :caption: Contribute
   

   contribute.rst
