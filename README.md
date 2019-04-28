# Ansible playbooks for OpenShift Libvirt + BYOR installation

## Table of Contents

- [OpenShift Installer Ansible](#openshift-installer-ansible)
- [Features](#features)
- [Quick Start](#quick-start)

## OpenShift Installer Ansible

The OpenShift Installer Ansible is a set of ansible playbooks and roles to automate the installation of a OpenShift cluster with libvirt VMs and scale the cluster with RHEL worker nodes.

## Features

- Spin up a OpenShift cluster with libvirt VMs.
- Scale the OpenShift cluster with RHEL worker nodes.

## Quick Start

### Install Libvirt Cluster

- Prerequisites

  - One Baremetal server installed with CentOS-7.6, Memory > 32G, CPU > 16
  - SSH login without password to the Baremetal server (ansible requirement)

- Configuration

  - ``pull_secret`` is used for pulling OpenShift images, it can be configured in ``inventory/group_vars/all.yml``, make sure the pull_secret string is quoted with single quote.
  - ``hypervisor`` host is the IP address of target Baremetal server, it can be configured in ``inventory/inventory``

- Deployment

  - Run ansible playbook ``playbooks/deploy.yml`` against Baremetal server. The playbook configures components(haproxy, NetworkManager etc), downloads images needed for bringing up a libvirt cluster and generates the ``start-vm.sh`` script.
```
ansible-playbook -i inventory/inventory playbooks/deploy.yml
```

- Start the cluster

  - Login the Baremetal server, run script ``/root/env/start-vm.sh`` which starts bootstrap, master-0, worker-0 nodes.

- Approve CSR

  - Manually approve all the csr
```
oc get csr -o name | xargs -n 1 oc adm certificate approve
```

### Scale the cluster with RHEL worker node

Scaling of new RHEL worker node is done via OpenShift-Ansible. Below steps are to prepare the necessary network and inventory file for running openshift-ansible scale script.

- Prerequisites

  - Assume the libvirt cluster is deployed and started successfully
  - Assume new worker node is installed with RHEL and accessable via ssh from Baremetal server
  - Assume new worker node has been configured with proper subscription to RHEL and OpenShift repos, packages such as openshift-clients, openshift-hyperkube, cri-o etc will be downloaned during scaling.
  - Disable selinux on new worker node
  - Add port 10250/tcp in firewalld on new worker node
  - Configure dns for api.{{ cluster_domain }} on new worker node


- Configuration (on local host)

  - ``new_worker_ip`` is IPv4 address of the new RHEL worker node to be added. It can be configured in ``inventory/group_vars/all.yml``

- Prepare network and inventory file for running openshift-ansible (on local host)

  - Run ansible playbook ``playbook/scale.yml`` against Baremetal server. The playbook configures network(haproxy etc) for new worker node to be added and generates the inventory file for running openshift-ansible.
```
ansible-playbook -i inventory/inventory playbooks/scale.yml
```

Inventory file for openshift-ansible will be generated under ``/root/env/scale/inventory.scale`` on Baremetal server, openshift-ansible will be downloaded under ``/root/env/scale/openshift-ansible``


- Scale the worker node with openshift-ansible (on remote Baremetal server)

  - Run openshift-ansible playbooks with generated inventory.scale

```
ansible-playbook -i inventory.scale openshift-ansible/playbooks/scaleup.yml
```

- Approve CSR

  - Manually approve all the csr
```
oc get csr -o name | xargs -n 1 oc adm certificate approve
