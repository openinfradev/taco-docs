**********
Kubespray
**********

주요기능
=========

* Can be deployed on AWS, GCE, Azure, OpenStack, vSphere, Oracle Cloud Infrastructure (Experimental), or Baremetal
* Highly available cluster
* Composable (Choice of the network plugin for instance)
* Supports most popular Linux distributions
* Continuous integration tests


주요 Playbook
==============

* cluster.yml : k8s 클러스터 설치
* remove-node.yml : 설치된 k8s 클러스터에서 노드 삭제
* reset.yml : k8s 클러스터 삭제
* scale.yml : 설치된 k8s 클러스터에 노드 추가
* upgrade-cluster.yml : 설치된 k8s 클러스터의 버전 업그레이드


Group
=====

* kube-node : list of kubernetes nodes where the pods will run.
* kube-master : list of servers where kubernetes master components (apiserver, scheduler, controller) will run.
* etcd: list of servers to compose the etcd server. You should have at least 3 servers for failover purpose.


/kubespray/roles/
=================

::

   .
   ├── adduser
   │   ├── defaults
   │   ├── tasks
   │   └── vars
   ├── bastion-ssh-config
   │   ├── tasks
   │   └── templates
   ├── bootstrap-os
   │   ├── defaults
   │   ├── files
   │   ├── tasks
   │   └── templates
   ├── container-engine
   │   ├── cri-o
   │   │   ├── defaults
   │   │   ├── files
   │   │   ├── tasks
   │   │   ├── templates
   │   │   └── vars
   │   ├── defaults
   │   ├── docker
   .
   .
   ├── download
   │   ├── defaults
   │   ├── meta
   │   └── tasks
   .
   .
   ├── kubernetes
   │   ├── client
   │   ├── kubeadm
   │   ├── master
   │   ├── node
   │   ├── preinstall
   │   ├── secrets
   │   └── tokens
   ├── kubernetes-apps
   .
   .
   │   ├── helm
   │   ├── network_plugin
   │   │   ├── calico
   │   │   │   └── tasks
   │   │   ├── canal
   │   │   │   └── tasks
   │   │   ├── cilium
   │   │   │   └── tasks
   │   │   ├── contiv
   │   │   │   └── tasks
   │   │   ├── flannel
   │   │   │   └── tasks
   │   │   ├── kube-router
   │   │   │   └── tasks
   │   │   ├── meta
   │   │   ├── multus
   │   │   │   └── tasks
   │   │   └── weave
   │   │       └── tasks
   │   ├── persistent_volumes
   │   │   ├── meta
   │   │   └── openstack
   │   │       ├── defaults
   │   │       ├── tasks
   │   │       └── templates
   │   ├── registry
   │   │   ├── defaults
   │   │   ├── tasks
   │   │   └── templates
   │   └── rotate_tokens
   │       └── tasks
   ├── network_plugin
   │   ├── calico
   │   ├── canal
   │   ├── cilium
   │   ├── cloud
   │   ├── contiv
   │   ├── flannel
   │   ├── kube-router
   │   ├── meta
   │   ├── multus
   │   └── weave
   ├── remove-node
   │   ├── post-remove
   │   │   └── tasks
   │   └── pre-remove
   │       ├── defaults
   │       └── tasks
   ├── reset
   │   ├── defaults
   │   └── tasks
   ├── upgrade
   │   ├── post-upgrade
   │   │   └── tasks
   │   └── pre-upgrade
   │       ├── defaults
   │       └── tasks
   └── win_nodes
       └── kubernetes_patch
           ├── defaults
           ├── files
           └── tasks


주요 인벤토리 파일
==================

* inventory/sample/group_vars/k8s-cluster/k8s-cluster.yml
* inventory/sample/group_vars/k8s-cluster/k8s-net-calico.yml
* inventory/sample/group_vars/k8s-cluster/addons.yml
* inventory/sample/group_vars/all/all.yml
* inventory/sample/group_vars/all/docker.yml


설치된 k8s 형상
===============

.. code-block:: bash

   [taco@centos01 ~]$ kubectl get all -n kube-system
   NAME                                          READY   STATUS    RESTARTS   AGE
   pod/calico-kube-controllers-df465b84f-nvhxh   1/1     Running   0          44h
   pod/calico-node-6ndj6                         1/1     Running   0          44h
   pod/calico-node-g8nth                         1/1     Running   0          44h
   pod/calico-node-vlvn6                         1/1     Running   0          44h
   pod/coredns-788d98cc7b-hc8vs                  1/1     Running   0          44h
   pod/coredns-788d98cc7b-hvfcd                  1/1     Running   0          44h
   pod/dns-autoscaler-6bd55f77d4-9bx6s           1/1     Running   0          44h
   pod/kube-apiserver-centos01                   1/1     Running   0          44h
   pod/kube-controller-manager-centos01          1/1     Running   0          44h
   pod/kube-proxy-s764l                          1/1     Running   0          44h
   pod/kube-proxy-wnlbl                          1/1     Running   0          44h
   pod/kube-proxy-zbhbj                          1/1     Running   0          44h
   pod/kube-scheduler-centos01                   1/1     Running   0          44h
   pod/kubernetes-dashboard-5db4d9f45f-rm82h     1/1     Running   0          44h
   pod/nginx-proxy-centos02                      1/1     Running   0          44h
   pod/nginx-proxy-centos03                      1/1     Running   0          44h
   pod/rbd-provisioner-86cbb58748-4qjmh          1/1     Running   0          44h
   pod/tiller-deploy-bf6884cdb-qsfsh             1/1     Running   0          44h
   
   NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
   service/coredns                ClusterIP   10.233.64.3     <none>        53/UDP,53/TCP,9153/TCP   44h
   service/kubernetes-dashboard   ClusterIP   10.233.99.64    <none>        443/TCP                  44h
   service/tiller-deploy          NodePort    10.233.103.53   <none>        44134:32134/TCP          44h
   
   NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
   daemonset.apps/calico-node   3         3         3       3            3           <none>                        44h
   daemonset.apps/kube-proxy    3         3         3       3            3           beta.kubernetes.io/os=linux   44h
   
   NAME                                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
   deployment.apps/calico-kube-controllers   1         1         1            1           44h
   deployment.apps/coredns                   2         2         2            2           44h
   deployment.apps/dns-autoscaler            1         1         1            1           44h
   deployment.apps/kubernetes-dashboard      1         1         1            1           44h
   deployment.apps/rbd-provisioner           1         1         1            1           44h
   deployment.apps/tiller-deploy             1         1         1            1           44h
   
   NAME                                                DESIRED   CURRENT   READY   AGE
   replicaset.apps/calico-kube-controllers-df465b84f   1         1         1       44h
   replicaset.apps/coredns-788d98cc7b                  2         2         2       44h
   replicaset.apps/dns-autoscaler-6bd55f77d4           1         1         1       44h
   replicaset.apps/kubernetes-dashboard-5db4d9f45f     1         1         1       44h
   replicaset.apps/rbd-provisioner-86cbb58748          1         1         1       44h
   replicaset.apps/tiller-deploy-bf6884cdb             1         1         1       44h


Supported Linux Distributions
==============================

* Container Linux by CoreOS
* Debian Buster, Jessie, Stretch, Wheezy
* Ubuntu 16.04, 18.04
* CentOS/RHEL 7
* Fedora 28
* Fedora/CentOS Atomic
* openSUSE Leap 42.3/Tumbleweed


Supported Components ( 2019.02.20 master )
==========================================

* Core

   * `kubernetes <https://github.com/kubernetes/kubernetes>`_ v1.13.3
   * `etcd <https://github.com/etcd-io/etcd>`_ v3.2.24
   * `docker <https://www.docker.com/>`_ v18.06 (see note)
   * `rkt <https://github.com/rkt/rkt>`_ v1.21.0 (see Note 2)
   * `cri-o <https://cri-o.io/>`_ v1.11.5 (experimental: see `CRI-O Note <https://github.com/kubernetes-sigs/kubespray/blob/master/docs/cri-o.md>`_. . Only on centos based OS) 

* Network Plugin

   * `calico <https://github.com/projectcalico/calico>`_ v3.4.0
   * `canal <https://github.com/projectcalico/canal>`_ (given calico/flannel versions)
   * `cilium <https://github.com/cilium/cilium>`_ v1.3.0
   * `contiv <https://github.com/contiv/install>`_ v1.2.1
   * `flanneld <https://github.com/coreos/flannel>`_ v0.11.0
   * `kube-router <https://github.com/cloudnativelabs/kube-router>`_ v0.2.1
   * `multus <https://github.com/intel/multus-cni>`_ v3.1.autoconf
   * `weave <https://github.com/weaveworks/weave>`_ v2.5.0

* Application

   * `cephfs-provisioner <https://github.com/kubernetes-incubator/external-storage>`_ v2.1.0-k8s1.11
   * `cert-manager <https://github.com/jetstack/cert-manager>`_ v0.5.2
   * `coredns <https://github.com/coredns/coredns>`_ v1.2.6
   * `ingress-nginx <https://github.com/kubernetes/ingress-nginx>`_ v0.21.0


vs kubeadm
===========

*Kubeadm provides domain Knowledge of Kubernetes clusters' life cycle management, including self-hosted layouts, dynamic discovery services and so on.
Had it belonged to the new operators world, it may have been named a "Kubernetes cluster operator".
Kubespray however, does generic configuration management tasks from the "OS operators" ansible world, plus some initial K8s clustering (with networking plugins included)
and control plane bootstrapping.
Kubespray strives to adopt kubeadm as a tool in order to consume life cycle management domain knowledge from it and offload generic OS configuration things from it,
which hopefully benefits both sides.*


apiserver client connection
============================

.. figure:: _static/taco-kubespray-api.png
