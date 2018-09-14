


## 问题记录

- privileged forbidden
```text
The DaemonSet "virt-handler" is invalid: spec.template.spec.containers[0].securityContext.privileged: Forbidden: disallowed by cluster policy
```

解决方法`exprot ALLOW_PRIVILEGED=true` 重启`hack/local-up-cluster.sh`

```
# cluster/kubectl.sh get pods -n kube-system
NAME                              READY     STATUS    RESTARTS   AGE
virt-api-747ff7bdc6-fbll8         1/1       Running   0          5m
virt-api-747ff7bdc6-ss86g         1/1       Running   1          5m
virt-controller-767587575-m6224   0/1       Running   4          5m
virt-controller-767587575-r5nr6   0/1       Running   3          5m
virt-handler-wq7lr                1/1       Running   0          5m
```

virt-launcher pod

```commandline
Command:
      /usr/share/kubevirt/virt-launcher/entrypoint.sh
      --qemu-timeout
      5m
      --name
      vmi-flavor-small
      --namespace
      default
      --kubevirt-share-dir
      /var/run/kubevirt
      --readiness-file
      /tmp/healthy
      --grace-period-seconds
      15
      --hook-sidecars
      0
```



- virt-api x509认证失败

原因 KUBE_ENABLE_CLUSTER_DNS=false

- allinone 模式 kubernetes_provider=local virt-controller failed

坑的原因，都是因为代理惹得祸，因为设置了代理给kube cluster导致的 

livenessprobe实际上是kubelet直接检测pod的状态，因为加了代理，导致检测失败


问题记录： k8s namespace ???


evel=error timestamp=2018-07-30T03:52:14.951323Z pos=server.go:68 component=virt-launcher namespace=default name=vmi-flavor-small-pvc kind= uid=ba6ae222-93ab-11e8-b1d2-246e96275bc0 reason="virError(Code=38, Domain=18, Message='Cannot access storage file '/var/run/kubevirt-private/vmi-disks/rbdvolume/disk.img': No such file or directory')" msg="Failed to sync vmi"



## kubernetes rbd volume

workflow： rbd map 将image映射到device，然后mount到指定的mount point，attach 给pod



## virtlet 
### installation
```commandline
go get -u -d github.com/Mirantis/criproxy github.com/Mirantis/virtlet

bash -x build/cmd.sh build

kubectl label node kubdev --overwrite extraRuntime=virtlet
kubectl create configmap -n kube-system virtlet-image-translations --from-file  `pwd`/deploy/images.yaml


docker exec virtlet-build "${remote_project_dir}/_output/virtletctl" gen "--dev" >virtlet.yaml

```