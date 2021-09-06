# How to pass CKA in 70 minutes
[![CKA](https://img.shields.io/badge/cka-1.21.0-green.svg?style=plastic)](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/) [![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)

## Cấu trúc

### Cấu trúc đề thi 

* Số lượng câu hỏi: 17
* Thời gian: 2 giờ

#### 25% - Cluster Architecture, Installation & Configuration

##### Architecture
Phần lớn trong phần này chủ yếu liên quan đến kiến trúc của cluster cũng như các thức install cluster.

Chú trọng các thành phần chính như sau:
- kubelet
- kube-apiserver
- kube-scheduler
- etcd
- controller-manager

Ngoài ra cũng phân cần phân biệt giữa cách hoạt động của static pod, pod và process.

##### Upgrade/Installation

Các bài thi sẽ không bao gồm việc setup 1 cluster mới mà thay vào đó là sẽ là các task upgrade version cho 1 hoặc nhiều nodes.

Lưu ý trình tự upgrade các node từ master cho đến các node. Cần đọc kĩ yêu cầu của task để làm đúng chính xác node nào cần upgrade và các thành phần upgrade là gì

**Master Node**

Nếu đề yêu cầu là upgrade master node. Ta sẽ action như sau:

* ***Step 1***: Check xem version `kubeadm` hiện tại của master node
* ***Step 2***: Tiến hành unschedule node và allocate pod scheduled on master node.
``` shell
basenode$ kubect drain master01 --ignore-daemonsets
```
* ***Step 3***: Tiến hành update package kubeadm

``` shell
basenode$ ssh master01
master01$ sudo apt update  # update package repo
master01$ sudo apt install kubeadm=<version> # Version mà task yêu cầu upgrade
# eg task yeu cau upgrade version 1.21.1
master01$ sudo apt install kubeadm=1.21.1-00
```
* ***Step 4***: Planing cho việc upgrade và tiến hành upgrade cho node
``` shell
master01$ kubeadm upgrade plan

# When the above command is done, run the following command to upgrade the components of k8s on master node.
master01$ kubeadm upgrade v1.21.1
```
* ***Step 5***: Update kubelet và các package liên quan (nếu có)
```shell
master01$ sudo apt install kubelet=1.21.1-00 kubectl=1.21.1-00
```
* ***Step 6***: Restart kubelet
``` shell
master01$ systemctl restart kubelet
```
* ***Step 7***: Back to basenode and uncordon node
``` shell
basenode$ kubectl uncordon master01
```

**Worker**

On the worker node, we easy to upgrade k8s

* ***Step 1***: Drain node. Example the node name is worker01
```shell
basenode$ kubectl drain node01 --ignore-daemonsets
```
* ***Step 2***: Upgrade kubeadm
```shell
basenode$ ssh node01
worker01$ sudo apt update && sudo apt install kubeadm=1.21.0-00 
```
* ***Step 3***: Run kubeadm to upgrade node
```shell
worker01$ kubeadm upgrade node
```
* ***Step 4***: Upgrade kubelet (and related package ) and restart kubelet
```shell
worker01$ apt install kubelet=1.21.0-00 
worker01$ systemctl restart kubelet
# Some task also require upgrade kubectl. Please careful.
worker01$ apt install kubectl=1.21.0-00
```
* ***Step 5***: Uncordon node
```
# Back to basenode
basenode$ kubectl uncordon worker01
```

**Some Tips**

```shell
# Generate and print join command for worker
kubeadm token create --print-join-command
```

**ETCD Backup**

Etcd DB khá quan trọng trong hệ thống k8s. Do đó nên cẩn trọng trong việc backup và restore etcd. Nếu làm không cẩn thận, nó sẽ làm cho các công việc bạn làm trước đó bị ảnh hưởng. Ví dụ như sai etcd của cluster k8s chính có thể làm mất hết các result mà bạn đã làm trước đó.

Phần lớn tỉ trọng etcd là 7-10% do đó bạn có thể cân nhắc by-pass qua vấn đề này

Tuy nhiên, tôi không khuyến khích bạn bỏ qua chỉ để pass CKA mà nên hiểu rõ để áp dụng thực tế. 

Đầu tiên cần kiểm tra xem, `etcd` nào mà đề yêu cầu bạn cần backup và restore. 

Thông thường để tránh ảnh hưởng đến các kết quả của bạn, đề thi sẽ bắt bạn thực hiện backup và restore trên một host độc lập. Tạm gọi là basenode.

Kế đến, kiểm tra `etcd` đang run dưới dạng `process` hay một `static pod` bằng cách.

    * Kiểm tra basenode có đang chạy kubelet ko. Và manifest nó đang là đường dẫn gì? Và trong đường dẫn manifest có static pod `etcd` hay không?
    * Sử dụng `ps -ef | grep etcd` để check nếu etcd sử dụng process.

***Lời khuyên***: backup lại file static pod hoặc lưu lại command xuất hiện trong ps để thuận tiện trong việc restore.

```shell
export ECTDCTL_API=3
etcdctl --cacert=<file_ca_crt> --key=<file_server_key> --cert=<file_server_crt> --endpoints https://127.0.0.1:2379 snapshot save /opt/backup-etcd

# Lưu ý endpoint phải đúng với etcd cần backup.
# /opt/backup-etcd: thay bằng path mà đề yêu cầu backup
```

***Restore***
``` shell
export ECTDCTL_API=3
etcdctl snapshot restore <path_file> --data-dir=<dir_extract_file_backup>

# Path file: file mà đề yêu cầu cần restore
# data-dir: data-dir thực ra là optional của của command restore. Tuy nhiên nên sử dụng nó để dễ dàng cho việc restore

# Sau đó chỉ cần sửa lại thu mục data dir trong etcd là đc
```

* Nếu etcd run là static-pod
``` yaml
api: v1
kind: Pod
metadata:
...
...
spec:
....
  volumes:
  - hostPath:
      path: <dir_extract_file_backup> # change here
      type: DirectoryOrCreate
    name: etcd-data
```
* Nếu etcd run là process
Chỉ biến môi trường ETCD_DATA_DIR trong systemd đúng với path vừa extract
* Nếu etcd run là docker
``` docker run -v <path_extract>:/var/lib/etcd:rw quay.io/coreos/etcd:v3.4.13 --env-file=/etc/etcd.env /usr/local/bin/etcd```

#### 15% - Workloads & Scheduling

**Sheduling**

Việc schedule cho `pod` được thực hiện bởi `kube-scheduler`. Tuy nhiên sẽ các trường hợp đề sẽ yêu cầu bạn manual scheduling khi kube-scheduler không hoạt động.

- Đề yêu cầu bạn tạm thời disable static pod kube-schedule rồi tạo 1 pod và schedule nó lên một node được đặt tên là node01.
``` yaml
// $ mv /etc/kubenetes/manifests/kube-schedulers.yaml /opt
// $ kubectl -n kube-system get pod  # chắc rằng pod đã tạm thời bị disable
// $ kubectl run pod --image=nginx --dry-run=client -o yaml > pod.yaml
// # edit file pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: pod
spec:
  nodeName: node01  // Name of node where the pod is located at
  containers:
  - image: nginx
    name: pod
```

- Taints và tolerations: 
  Mặc định khi khởi tạo cluster bằng kubeadm thì master node sẽ được thiết lập thuộc tính taints. 
``` shell
$ kubectl describe node master | grep -i taints # Use option "-i" is search pattern without case sensitive
Taints: node-role.kubernetes.io/master:NoSchedule
```
  Khi masster node có thuộc tính taints nghĩa là kube-scheduler sẽ không lập lịch các pod bình thường vào node này. Để lập lịch pod có thể được locate lên master ta action như sau:
  ``` yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod
  spec:
    tolerations:
    - key:  node-role.kubernetes.io/master
      effect: NoSchedule
    containers:
    - name: pod
      image: nginx
  ```
- NodeSelector
**Workloads**

#### 20% - Services & Networking

• Understand host networking configuration on the cluster nodes
• Understand connectivity between Pods
• Understand ClusterIP, NodePort, LoadBalancer service types and endpoints
• Know how to use Ingress controllers and Ingress resources
• Know how to configure and use CoreDNS
• Choose an appropriate container network interface plugin

#### 10% - Storage

Phần này khá dễ, chủ yếu đọc kĩ đề và lựa chọn đúng storage classes, đúng accessMode, đặt đúng tên PV hoặc PVC lúc config

Chỉ một lưu ý đối với đề có sidecar container thì thường khái niệm đang chạy chỉ đến là multi container on pod. Lúc này ta sử dụng emptyDir để tạo ra volume sharing giữa các container nhưng không persistent

``` yaml
// kubectl run multi-container --image=busybox --dry-run=client -o yaml > multi-container.yaml
// Edit multi-container.yaml be like
---
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
  labels:
    run: multi-container
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  container:
  - image: busybox
    name: container1
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo $date > /vol/date.log; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /vol/
  - image: busybox
    name: container1
    command: ["/bin/sh"]
    args: ["-c", "tail -f /vol/date.log"]
    volumeMounts:
    - name: shared-data
      mountPath: /vol/

// run: kubectl -f multi-container.yaml create
```

Nếu đề có yêu cầu expands capacity của PVC mà có record thì ta thực hiện như sau:
``` shell
# Expand capacity for pvc example-pvc to 100Gi
$ kubectl edit pvc example-pvc --record=true

# Find line capacity and edit number to 100Gi.
```

#### 30% - Troubleshooting



## Cluster for CKA

|Cluster|Members|CNI|Description|
|:------|:--|:--|:--|
|k8s|1 master, 2 worker|flanel|K8s cluster|
|hk8s|1master, 2 worker|calico|k8s cluster|
|bk8s|1master, 1 worker|flannel|k8s cluster|
|wk8s|1master, 2 worker|flanel|k8s cluster|
|ek8s|1master, 2 worker|flanel|k8s cluster|
|ik8s|1master, 1 base node|loopback|k8s cluster - missing worker node|

## Cheat sheet

Trong quá trình làm exam, các cheat sheet sau sẽ giúp cho việc thực hiện sẽ được nhanh hơn

``` shell
# 1. Use kubectl alias, which means you always run kubectl just with k
alias k=kubectl

# 2. This way you can just run k run pod1 --image=nginx $do
export do="--dry-run=client -o yaml"

# 3. Sometime you need to wait few seconds to kill pods, force killing them is better, create a shortcut to do it faster `k delete pod <podname> $fk`
export fk="--force --grace-period=0"

# 4. Dont delete and recreate pods, do replace of the yaml
k replace -f <yaml-file> --force

# 5.  Adapt ~/.vimrc to be able to apply tab on multiple selected lines(I did not do it myself)
set shiftwidth=2

# 6. Enable autocompletion(After you use the alias, autocompletion will be disabled, use this to reenable it)
source <(kubectl completion bash)
and then
complete -F __start_kubectl k

```
