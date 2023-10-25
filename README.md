# doks-s3
How to use a Spaces Object Storage bucket as storage for a Kubernetes Pod with DOKS (DigitalOcean Kubernetes)

## About this guide

In this guide we will deploy a DigitalOcean Managed Kubernetes cluster. We'll use [k8s-csi-s3](https://github.com/yandex-cloud/k8s-csi-s3) which is a [GeeseFS-based](https://github.com/yandex-cloud/geesefs) CSI for mounting S3 buckets as PersistentVolumes and give some working examples of consuming RMX storage.

### About DigitalOcean DOKS

> DigitalOcean Kubernetes (DOKS) is a managed Kubernetes service that lets you deploy Kubernetes clusters without the complexities of handling the control plane and containerized infrastructure. Clusters are compatible with standard Kubernetes toolchains and integrate natively with DigitalOcean Load Balancers and volumes.

### About DigitalOcean Spaces Object Storage

> Spaces Object Storage is an S3-compatible object storage service that lets you store and serve large amounts of data. Each Space is a bucket for you to store and serve files. The built-in Spaces CDN minimizes page load times and improves performance.

### About CSI for S3

> This is a Container Storage Interface (CSI) for S3 (or S3 compatible) storage. This can dynamically allocate buckets and mount them via a fuse mount into any container.

## Deploy a DigitalOcean DOKS Cluster

### How to create a Kubernetes cluster using the DigitalOcean CLI

To create a Kubernetes cluster via the command-line, follow these steps:

1.  [Install `doctl`](https://docs.digitalocean.com/reference/doctl/how-to/install/), the DigitalOcean command-line tool.

2.  [Create a personal access token](https://docs.digitalocean.com/reference/api/create-personal-access-token/), and save it for use with `doctl`.

3.  Use the token to grant `doctl` access to your DigitalOcean account.

    ```
    doctl auth init
    ```

4.  Finally, create a Kubernetes cluster with `doctl kubernetes cluster create`.

    ```
    doctl kubernetes cluster create doks-shark-1 --auto-upgrade=true --ha=true --node-pool="name=pool-apps;size=s-4vcpu-8gb-amd;count=3" --region=lon1 --surge-upgrade=true
    ```

* `doks-shark-1` is our cluster name
* We are creating one node pools, 1x for our regular workloads and in the next step we will create 1x dedicated to our storage nodes.
* The region is London, surge-upgrades are enabled, HA is enabled, auto-upgrade is enabled.
* You'll want to [read the usage docs for more details](https://docs.digitalocean.com/reference/doctl/reference/kubernetes/cluster/create/)

## Setup k8s-csi-s3

#### 1. Create a secret with your S3 credentials

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-s3-secret
  namespace: kube-system
stringData:
  accessKeyID: <YOUR_ACCESS_KEY_ID>
  secretAccessKey: <YOUR_SECRET_ACCESS_KEY>
  endpoint: https://ams3.digitaloceanspaces.com

#### 2. Deploy the driver

```bash
cd deploy/kubernetes
kubectl create -f provisioner.yaml
kubectl create -f driver.yaml
kubectl create -f csi-s3.yaml
```

If you're upgrading from a previous version which had `attacher.yaml` you
can safely delete all resources created from that file:

```
wget https://raw.githubusercontent.com/yandex-cloud/k8s-csi-s3/v0.35.5/deploy/kubernetes/attacher.yaml
kubectl delete -f attacher.yaml
```

#### 3. Create the storage class

```bash
kubectl create -f examples/storageclass.yaml
```

#### 4. Test the S3 driver

1. Create a pvc using the new storage class:

    ```bash
    kubectl create -f examples/pvc.yaml
    ```

1. Check if the PVC has been bound:

    ```bash
    $ kubectl get pvc csi-s3-pvc
    NAME         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    csi-s3-pvc   Bound     pvc-c5d4634f-8507-11e8-9f33-0e243832354b   5Gi        RWO            csi-s3         9s
    ```

1. Create a test pod which mounts your volume:

    ```bash
    kubectl create -f examples/pod.yaml
    ```

    If the pod can start, everything should be working.

1. Test the mount

    ```bash
    $ kubectl exec -ti csi-s3-test-nginx bash
    $ mount | grep fuse
    pvc-035763df-0488-4941-9a34-f637292eb95c: on /usr/share/nginx/html/s3 type fuse.geesefs (rw,nosuid,nodev,relatime,user_id=65534,group_id=0,default_permissions,allow_other)
    $ touch /usr/share/nginx/html/s3/hello_world
    ```

If something does not work as expected, check the troubleshooting section below.

## Additional configuration

### Bucket

By default, csi-s3 will create a new bucket per volume. The bucket name will match that of the volume ID. If you want your volumes to live in a precreated bucket, you can simply specify the bucket in the storage class parameters:

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: csi-s3-existing-bucket
provisioner: ru.yandex.s3.csi
parameters:
  mounter: geesefs
  options: "--memory-limit 1000 --dir-mode 0777 --file-mode 0666"
  bucket: some-existing-bucket-name
```

If the bucket is specified, it will still be created if it does not exist on the backend. Every volume will get its own prefix within the bucket which matches the volume ID. When deleting a volume, also just the prefix will be deleted.

### Static Provisioning

If you want to mount a pre-existing bucket or prefix within a pre-existing bucket and don't want csi-s3 to delete it when PV is deleted, you can use static provisioning.

To do that you should omit `storageClassName` in the `PersistentVolumeClaim` and manually create a `PersistentVolume` with a matching `claimRef`, like in the following example: [deploy/kubernetes/examples/pvc-manual.yaml](deploy/kubernetes/examples/pvc-manual.yaml).

You can check POSIX compatibility matrix here: https://github.com/yandex-cloud/geesefs#posix-compatibility-matrix.

#### GeeseFS

* Almost full POSIX compatibility
* Good performance for both small and big files
* Does not store file permissions and custom modification times
* By default runs **outside** of the csi-s3 container using systemd, to not crash
  mountpoints with "Transport endpoint is not connected" when csi-s3 is upgraded
  or restarted. Add `--no-systemd` to `parameters.options` of the `StorageClass`
  to disable this behaviour.

## Troubleshooting

### Issues while creating PVC

Check the logs of the provisioner:

```bash
kubectl logs -l app=csi-provisioner-s3 -c csi-s3 -n kube-system
```

### Issues creating containers

1. Ensure feature gate `MountPropagation` is not set to `false`
2. Check the logs of the s3-driver:

```bash
kubectl logs -l app=csi-s3 -c csi-s3 -n kube-system
```


## Benchmarks

Benchmarks ran using [dbench](https://github.com/jkpedo/dbench/tree/doks)

DigitalOcean Volume limits are [detailed here](https://docs.digitalocean.com/products/volumes/details/limits/)

### Native volume testing

Below are the results of a `s-2vcpu-4gb-amd` worker node with a 1TB Volume attached using the `do-block-storage` storageClass

```
==================
= Dbench Summary =
==================
Random Read/Write IOPS: 9986/9987. BW: 384MiB/s / 387MiB/s
Average Latency (usec) Read/Write: 750.36/399.11
Sequential Read/Write: 384MiB/s / 395MiB/s
Mixed Random Read/Write IOPS: 7515/2471
```

## Conclusion
