# Pocket Manila Kube

This "pocket" manila kube is based on the original [manila kube](https://github.com/tombarron/manila-kube), but
whereas the original deploys Kubernetes nodes on VMs running in an OpenStack cloud,
this pocket version uses [kind](https://kind.sigs.k8s.io) to deploy on a single target machine.  Currently
a single VM is also deployed on that target machine, using Vagrant with the libvirt VM, for a minimal
devstack.

After the deployment the target machine has a *kind* based Kubernetes cluster with a single
control plane node and three workers:

    [stack@c7 ~]$ kind get clusters
    manila-kube
    [stack@c7 ~]$ kubectl get nodes
    NAME                        STATUS   ROLES    AGE   VERSION
    manila-kube-control-plane   Ready    master   15h   v1.16.3
    manila-kube-worker          Ready    <none>   15h   v1.16.3
    manila-kube-worker2         Ready    <none>   15h   v1.16.3
    manila-kube-worker3         Ready    <none>   15h   v1.16.3

Manila-csi plugins and the partner NFS csi node plugin are deployed:

    [stack@c7 ~]$ kubectl get all
    NAME                                              READY   STATUS    RESTARTS   AGE
    pod/csi-nodeplugin-nfsplugin-ft8rl                1/1     Running   0          22h
    pod/csi-nodeplugin-nfsplugin-n885l                1/1     Running   0          22h
    pod/csi-nodeplugin-nfsplugin-pdfkm                1/1     Running   0          22h
    pod/sharedstorage-manila-csi-controllerplugin-0   3/3     Running   0          22h
    pod/sharedstorage-manila-csi-nodeplugin-8tfzg     2/2     Running   0          22h
    pod/sharedstorage-manila-csi-nodeplugin-fcppm     2/2     Running   0          22h
    pod/sharedstorage-manila-csi-nodeplugin-pdt2f     2/2     Running   0          22h

    NAME                                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
    service/kubernetes                                  ClusterIP   10.96.0.1       <none>        443/TCP     22h
    service/sharedstorage-manila-csi-controllerplugin   ClusterIP   10.100.218.24   <none>        12345/TCP   22h

    NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
    daemonset.apps/csi-nodeplugin-nfsplugin              3         3         3       3            3           <none>          22h
    daemonset.apps/sharedstorage-manila-csi-nodeplugin   3         3         3       3            3           <none>          22h

    NAME                                                         READY   AGE
    statefulset.apps/sharedstorage-manila-csi-controllerplugin   1/1     22h

The *kubectl* and *kind* commands as used above are installed as well as *manila* and *openstack* clients.
To use the latter source the *adminrc* or *demorc* credential files 
stored under *~/manila-kube-workspace/Vagrant* on the target
machine:

    [stack@c7 ~]$ source manila-kube-workspace/Vagrant/adminrc
    [stack@c7 ~]$ manila service-list
    +----+------------------+----------------------------+---------------+---------+-------+----------------------------+
    | Id | Binary           | Host                       | Zone          | Status  | State | Updated_at                 |
    +----+------------------+----------------------------+---------------+---------+-------+----------------------------+
    | 1  | manila-share     | ubuntu-devstack@cephfsnfs1 | manila-zone-0 | enabled | up    | 2020-01-07T01:07:35.000000 |
    | 2  | manila-scheduler | ubuntu-devstack            | nova          | enabled | up    | 2020-01-07T01:07:34.000000 |
    | 3  | manila-data      | ubuntu-devstack            | nova          | enabled | up    | 2020-01-07T01:07:36.000000 |
    +----+------------------+----------------------------+---------------+---------+-------+----------------------------+
    [stack@c7 ~]$ source manila-kube-workspace/Vagrant/demorc
    [stack@c7 ~]$ manila list
    +--------------------------------------+--------+------+-------------+-----------+-----------+-----------------+------+-------------------+
    | ID                                   | Name   | Size | Share Proto | Status    | Is Public | Share Type Name | Host | Availability Zone |
    +--------------------------------------+--------+------+-------------+-----------+-----------+-----------------+------+-------------------+
    | 8ae07748-ce94-42a9-852b-92dba7d0cf9e | share1 | 1    | NFS         | available | False     | cephfsnfstype   |      | manila-zone-0     |
    +--------------------------------------+--------+------+-------------+-----------+-----------+-----------------+------+-------------------+
    [stack@c7 ~]$

These clients interact with a minimal
[devstack](https://docs.openstack.org/devstack/latest/) in which only
Manila and the services it requires (like Keystone) are deployed.  (If
you want to run a fuller OpenStack, edit
*roles/setup_devstack_vm/files/local.conf* and update
*roles/setup_devstack_vm/files/Vagrantfile* to allocate more resources
to the devstack VM).

 ## How To Deploy

1. Copy *inventory.yml.sample* to *inventory.yml*

2. Edit *inventory.yml* to indicate the
target machine or machines where you want manila kube deployed, the
non-root user there who will own the deployment, the target machine
workspace, etc.

3. Run **ansible-playbook site.yml** to deploy manila kube.

## Working with Manila Kube

You can create pvcs and pods using them using manifests like those in the cloud provider
openstack source under *examples/manila-csi-plugin/nfs/dynamic-provisioning/*

Source for the cloud provider openstack repository is cloned under
*~/manila-kube-workspace/src/* on the deployment target:

    [stack@c7 ~]$ cd manila-kube-workspace/src/k8s.io/cloud-provider-openstack/
    [stack@c7 cloud-provider-openstack]$ git remote -v
    origin	https://github.com/kubernetes/cloud-provider-openstack (fetch)
    origin	https://github.com/kubernetes/cloud-provider-openstack (push)

You can modify/patch the manila-csi source code here and run
**ansible-playbook redeploy-manila_csi.yml** to rebuild the manila-csi
image with the changes and redeploy the manila-kube k8s cluster with
the updated image.  We re-deploy the K8s cluster rather than just 
reloading the image in order to avoid test bugs due to state in the 
cluster prior to the image change.  It's pretty fast with a *kind* 
cluster -- takes a bit over 5 minutes on my laptop.

To patch or modify manila itself, login to the VM running devstack manila:

    [stack@c7 ~]$ cd manila-kube-workspace/Vagrant
    [stack@c7 Vagrant]$ vagrant ssh
    Last login: Mon Jan  6 02:00:42 2020 from 192.168.18.1
    vagrant@ubuntu-devstack:~$ ls /opt/stack/manila
    api-ref    bindep.txt  CONTRIBUTING.rst  doc  HACKING.rst  lower-constraints.txt  manila.egg-info  rally-jobs  releasenotes      setup.cfg  test-requirements.txt  tox.ini
    babel.cfg  contrib     devstack          etc  LICENSE      manila                 playbooks        README.rst  requirements.txt  setup.py   tools
    vagrant@ubuntu-devstack:~$

On the VM, you can modify the manila source code at /opt/stack/manila
or cherry-pick pending reviews, etc.  Run **sudo systemctl restart
devstack@m\*** on the devstack VM to run with your changes.

If you need to tear down the devstack VM itself and rebuild you can
run **ansible-playbook redeploy-devstack-vm.yml**.

## Pre-requisites

To run these playbooks as is you need to be able to ssh to target
machines as a regular, non-root user with passwordless sudo
privileges.  This is the *regular_user* defined in inventory.yml for
each target host defined there, and may or may not be the same as the
user running the playbook.  (To run the bootstrap the user running the
playbook must be able to ssh to the target, of course, and will need
to be able to sudo on the target, but does not need to be in wheel
group, etc.)

This user can be set up by running

    ansible-playbook bootstrap.yml --extra-vars "ansible_become_pass=SECRETPASSWORD"

when the *regular_user* defined in inventory.yml is not the same as the
user running the playbook and/or when that user is not yet set up on the
remote machine with appropriate privileges.

## Note on deployment targets

I've deployed against CentOS 7 and Fedora 29 and Fedora 31 targets.
More RAM on the target is better, but it worked for me on an 8GB RAM
CentOS 7 machine.

*kind* does not yet work with *podman* yet, so we need to install
*docker*.  Fedora 31 ships with cgroups v2, which is
currently docker-incompatible.  So with Fedora 31 and higher you need to
revert to cgroups v1 on the target machine before running this playbook,
as explained in the
[Fedora project wiki](https://fedoraproject.org/wiki/Common_F31_bugs#Docker_package_no_longer_available_and_will_not_run_by_default_.28due_to_switch_to_cgroups_v2.29).

If you want to deploy on other targets, open a github issue on this
repository and I'll try to help make it happen.

## Note on packaging and idempotence

Devstack installs a lot of software via *pip*.  In order to avoid
polluting the target machine, we install devstack in a VM rather than on
the target machine itself.

devstack's setup script is not idempotent.  If you have failures you
may need to run vagrant to blow away the VM and re-run the deployment.
See the convenience tags in site.yml.

golang executables and shell scripts don't have library dependencies
the so are deployed directly on the target machine at */usr/local/bin*
or *~/bin* so they don't conflict or overwrite system installs.

We install the python manilaclient and openstack client on the target
machine for convenience but do *user* pip installs so everything is
under *~/.local* and does not overwrite any system installed software.

## TODO

* Try deploying Ceph outside of devstack via Rook/Ceph (probably in an independent but kind-deployed K8s cluster, de-coupled from the manila-kube k8s cluster).

* Eliminate the devstack VM, incrementally

  * Run Manila and  Keystone in containers, ideally in their own K8s cluster

  * Maybe run the OpenStack database and rabbitmq in containers in the K8s cluster with Manila and Keystone, maybe on the target host (deployed via rpms).
