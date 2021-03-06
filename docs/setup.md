# TiDB Operator Setup

## Requirements

Before deploying the TiDB Operator, make sure the following requirements are satisfied:

* Kubernetes v1.10 or later
* [DNS addons](https://kubernetes.io/docs/tasks/access-application-cluster/configure-dns-cluster/)
* [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
* [RBAC](https://kubernetes.io/docs/admin/authorization/rbac) enabled (optional)
* [Helm](https://helm.sh) v2.8.2 or later

> **Note:** Though TiDB Operator can use network volume to persist TiDB data, it is highly recommended to set up [local volume](https://kubernetes.io/docs/concepts/storage/volumes/#local) for better performance. Because TiDB already replicates data, network volume will add extra replicas which is redundant.

## Kubernetes

TiDB Operator runs on top of Kubernetes cluster, you can use one of the methods listed [here](https://kubernetes.io/docs/setup/pick-right-solution/) to set up a Kubernetes cluster. Just make sure the Kubernetes cluster version is equal or greater than v1.10. If you want to use AWS, GKE or local machine, there are quick start tutorials:

* [Local DinD tutorial](./local-dind-tutorial.md)
* [Google GKE tutorial](./google-kubernetes-tutorial.md)
* [AWS EKS tutorial](./aws-eks-tutorial.md)

If you want to use a different envirnoment, a proper DNS addon must be installed in the Kubernetes cluster. You can follow the [official documentation](https://kubernetes.io/docs/tasks/access-application-cluster/configure-dns-cluster/) to set up a DNS addon.

TiDB Operator uses [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) to persist TiDB cluster data (including the database, monitor data, backup data), so the Kubernetes must provide at least one kind of persistent volume. To achieve better performance, local SSD disk persistent volume is recommended. You can follow [this step](#local-persistent-volume) to auto provisioning local persistent volumes.

The Kubernetes cluster is suggested to enable [RBAC](https://kubernetes.io/docs/admin/authorization/rbac). Otherwise you may want to set `rbac.create` to `false` in the values.yaml of both tidb-operator and tidb-cluster charts.

Because TiDB by default will use at most 40960 file descriptors, the [worker node](https://access.redhat.com/solutions/61334) and its [Docker daemon's](https://docs.docker.com/engine/reference/commandline/dockerd/#default-ulimit-settings) ulimit must be configured to greater than 40960. Otherwise you have to change TiKV's `max-open-files` to match your work node `ulimit -n` in the configuration file `charts/tidb-cluster/templates/config/_tikv-config.tpl`, but this will impact TiDB performance.

## Helm

You can follow Helm official [documentation](https://helm.sh) to install Helm in your Kubernetes cluster. The following instructions are listed here for quick reference:

1. Install helm client

    ```
    $ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash
    ```

    Or if you use macOS, you can use homebrew to install Helm by `brew install kubernetes-helm`

2. Install helm server

    ```shell
    $ kubectl apply -f https://raw.githubusercontent.com/pingcap/tidb-operator/master/manifests/tiller-rbac.yaml
    $ helm init --service-account=tiller --upgrade
    $ kubectl get po -n kube-system -l name=tiller # make sure tiller pod is running
    ```

    If `RBAC` is not enabled for the Kubernetes cluster, then `helm init --upgrade` should be enough.

## Local Persistent Volume

Local disks are recommended to be formatted as ext4 filesystem.

Mount local ssd disks of your Kubernetes nodes at subdirectory of /mnt/disks. For example if your data disk is `/dev/nvme0n1`, you can format and mount with the following commands:

```shell
$ sudo mkdir -p /mnt/disks/disk0
$ sudo mkfs.ext4 /dev/nvme0n1
$ sudo mount -t ext4 -o nodelalloc /dev/nvme0n1 /mnt/disks/disk0
```

To auto-mount disks when your operating system is booted, you should edit `/etc/fstab` to include these mounting info.

After mounting all data disks on Kubernetes nodes, you can deploy [local-volume-provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/local-volume) to automatically provision the mounted disks as Local PersistentVolumes.

```shell
$ kubectl apply -f https://raw.githubusercontent.com/pingcap/tidb-operator/master/manifests/local-dind/local-volume-provisioner.yaml
$ kubectl get po -n kube-system -l app=local-volume-provisioner
$ kubectl get pv | grep local-storage
```

## Install TiDB Operator

```shell
$ kubectl apply -f https://raw.githubusercontent.com/pingcap/tidb-operator/master/manifests/crd.yaml
$ helm install charts/tidb-operator --name=tidb-operator --namespace=tidb-admin
$ kubectl get po -n tidb-admin -l app.kubernetes.io/name=tidb-operator
```
