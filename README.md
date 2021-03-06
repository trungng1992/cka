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
  
  Node selector sử dụng để assign pod đến một một tập các nodes. Có rất nhiều cách để làm thực hiện,  trong đó sử dụng `labels selectors` để action.

  Tạo 1 pod `pod1` chạy với trên node đang có GPU. 

  Giả sử node01 đang có GPU nhưng chưa được set labels để dễ detect.
  
  ``` shell
  # Set labels GPU=true for node 01
  kubectl labels nodes node01 gpu=true
  ```

  Tạo 1 pod với thuộc tính nodeSelector:

  ``` yaml
  // pod1.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: podd1
  spec:
    nodeSelector:
      gpu: true 
    containers:
    - name: pod1
      image: nginx
  ```

  Nodeselector trên pod.Specs có format là một kiểu map[string]string. Do đó không thể sử dụng cùng 1 loại key cho nodeSelector

  Ví dụ hạ tầng node có 3 loại disk được phân theo labels: disk=sata, disk=ssd và disk=sas. Thì nodeSelector không thể giải quyết bài toán shedule cho pod nếu yêu cầu pod có thể chạy trên disk là sata or disk là sas.

  Các điều kiện trong nodeSelector được thực hiện bằng `AND` operators.

- Affinity và AntiAffinity

  Tính năng affinity và antiAffitiny  giải quyết hạn chế của nodeSelector. 

  1. Cung cấp nhiều quy tắc so sánh hơn so với nodeSelector (nodeSelector chỉ hỗ trợ `AND` operator)
  2. Cung cấp thêm một cơ chế matching là `soft`. Nghĩa là nếu pod có những thuộc tính mà `kube-scheduler` không đáp ứng được, thì pod này vẫn được schedule. Như nodeSelector là `hard` matching. Nếu không có node nào đáp ứng đúng những labels của `POD` cần select thì `pod` sẽ ở trạng thái pending đến khi nào có 1 node thỏa điều kiện
  3. Ngoài ra với antiAffinity: `pod` sẽ được hạn chế schedule trên các labels của các `pods` đang chạy. Ví dụ như khi bạn triển khai deployment một application nào đó với replicas là 6. Tuy nhiên hiện tại chỉ có 5 node available. Nếu như không có thuộc tính antiAffinity thì sẽ có 1 node sẽ chạy ít nhất là 2 pod (phụ thuộc workload CPU)

  Example:
  ``` yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-1
  spec:
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: disk
              operator: In
              values:
              - sata
              - sas
    containers:
    - name: with-node-affinity
      image: k8s.gcr.io/pause:2.0
  // Với config nodeAffinity này, pod chỉ được schedule trên những node có labels disk là sata hoặc là sas
  ```
  
  Operator support: In, NotIn, Exists, DoseNotExists, Gt, Lt

  Nếu cùng sử dụng cả hai `nodeSelector` và `nodeAffinity`, thì node ứng cử viên phải đáp ứng cả hai điều kiện.
  
  Nếu có nhiều `nodeSelectorTerms` trong `nodeAffinity` thì pod sẽ được scheduled trên lên node thỏa 1 trong những điều kiện của `nodeSelectTerms`

  Nếu có nhiều `matchExpressions` trong `nodeSelectorTerms` thì pod sẽ được schedule lên node thỏa tất cả điều kiện của `matchExpressions`

  **podAffinity/podAntiAffinty**

  Hai thuộc tính này cho phép bạn  giới hạn các node mà các `pod` của bạn bạn đủ điều kiện để schedule dựa trên labels của `pod` đang chạy trên node.

  Để dễ hiểu ta xem ví dụ dưới đây
  ``` yaml
  apiVersion: apps
  kind: Deployment
  metadata:
    name: deploy-v1
    labels:
      id: deploy-v1
  spec:
    replicas: 2
    selector:
      matchLabels:
        id: deploy-v1
    template:
      metadata:

        labels:
          id: deploy-v1
      spec:
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                - key: id
                  operator: In
                  values:
                  - deploy-v1
              topologyKey: "kubernetes.io/hostname"
        containers:
        - name: nginx
          image: nginx
  ```

  Với manifest như thế này thì pod sẽ không được schedule trên chính các node mà đã chưa pod deploy-v1-xxxx-xxxx đang chạy.

  Ví dụ với replicas là 2, định nghĩa thứ tự mỗi pod là pod-1 và pod-2. Và hệ thống hiện tại chỉ có 1 worker  A đang available. Đầu tiên pod-1 sẽ được schedule trên worker A. Lúc đo pod đang có labels `id=deploy-v1`.  Sau đó pod-2 sẽ tiếp tục schedule, tuy nhiên khi check điều kiện `affinity` thì worker A không thỏa điều kiện dó đã có pod-1 đang chạy trên đó có labels match với điều kiện của podAntiAffinity

  Tham khảo thêm [link](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) để biết thêm về namespaceSelector cũng như hiểu được 4 cơ chế scheduler:

    - requiredDuringScheduling**Ignored**DuringExecution
    - preferredDuringScheduling**Ignored**DuringExecution
    - requiredDuringScheduling**Required**DuringExecution
    - preferredDuringScheduling**Ignored**DuringExecution

  Phần này sẽ không có trong đề thi CKA tuy nhiên cũng là một kiến thức bạn nên cần nắm vũng.

**Workloads**

  - ***Scale***:
  
  ```kubectl scale <resource> <name> --replicas=<number_replicas> [--record=true]```

  Resouce available:
  * Deployment
  * Statefulset
  * ReplicaSet

  Options: record=true. It allows record the command in `Annotations`

  - ***tip***: 
  
  Trong bài thi, nếu đề yêu cầu bạn scale một applications trong một namespace nào đó. Không chỉ rõ resource source nào thì có thể check bằng command như sau:

  ```kubectl -n <namespace> get all | grep -i "<pattern_you_wanna_search">```

  - ***Create manifest for daemonset app-1 quickly***:

  ```kubectl create deployment app-1 -n namespace1 --image=nginx --dry-run=client -o yaml > pod-1.yaml```
  
  Edit the manifest

  ```yaml
  // pod-1.yaml
  apiVersion: apps
  kind: DaemonSet # change Deployment to DaemonSet
  metadata:
    labels:
      app: app-1
    name: app-1
    namespace: namespace-1
  spec:
    # replicas: 1    remove line
    selector:
      matchLabels:
        app: app-1
    # strategy: {}   remove line
    template:
      metadata:
        labels:
          app: app-1
      spec:
        containers:
        - image: nginx
          name: nginx
          resources: {}
  ```

  - ***Update image for deployment app-1 and record***

  ```kubectl set image deployment app-1 <name_container>=<version> --record=true```

  Eg: ```kubectl set image deployment app-1 nginx=1.21 --record=true```

  - ***Undo revision***

  ```kubectl rollout undo <resource> <name> --to-revision=<revision>```

  Eg: ```kubectl rollout undo daemonset ingress-nginx --to-version=1```

  - ***Set limit/request resouce***

  ```
  apiVersion: v1
  kind: Pod
  metadata:
    name: pod-1
  spec:
    containers:
    - name: pod-1
      image: busybox
      command: ["sleep", "4800"]
      resources:
        requests:
          memory: 1M
          cpu: 100m
        limits:
          memory: 10M
          cpu: 1
  ```

  Quick command create `pod` with request/limit

  ```kubectl run pod-1 --image=busybox --limits=cpu=1,memory=10M --requests=cpu=100m,memory=1M --command -- sleep 4800```

  Set a deployments nginx container cpu limits to "100m" and memory to "256Mi":

  ```kubectl set resources deployment nginx -c=nginx --limits=cpu=100m,memory=256Mi```

  Set the resource request and limits for all containers in nginx:

  ```kubectl set resources deployment nginx --limits=cpu=200m,memory=512Mi --requests=cpu=100m,memory=256Mi```

  Ngoài ra, trong phần này bạn cần nắm rõ nguyên tắc hoạt động của các extensions như Deployment, StatefulSet cũng như DaemonSets. Hiểu và biết vận dụng vào các trường hợp nhất định.
#### 20% - Services & Networking

- Understand host networking configuration on the cluster nodes
- Understand connectivity between Pods
- Understand ClusterIP, NodePort, LoadBalancer service types and endpoints
- Know how to use Ingress controllers and Ingress resources
- Know how to configure and use CoreDNS
- Choose an appropriate container network interface plugin

- ***Create services with nodePort for deployment nginx***

  ```kubectl expose deployment nginx --name=<service-name> --port=80 --target-port=80 --type=NodePort```

- ***Create services with clusterIP for pod nginx***

  ```kubectl expose pod nginx --name=<service-name> --port=80 --target-port=80```

- ***Create networkpolicy***

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: <name_network_policy>
  # namespace: namespace | if task required
spec:
  podSelector: {} # must have this spec. If network policy apply for all pods in a namespace. Fill {}. If apply for invidiual pod, fill matchLabels: and labels of pod.
  # podSelector:
  #   matchLabels:
  #     app: db
  policyTypes:
  - Ingress
  ingress:
  - from:
      podSelector:
        matchLabels:
          app: front-end
    ports:
    - port: 80
      protocol: TCP
```

- ***Ingress Controller***

  Create ingress services ingress-app-1 for service service-1 in `namespace-1` namespace with prefix path /hello
  ``` yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: ingress-app-1
    namepsace: namespace-1
  spec:
    rules:
    - http:
        paths:
        - path: /hello
          pathType: Prefix
          backend:
            service:
              name: service-1
              port:
                number: 80
  ```


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

Troubleshoot thì có khá rộng. Tuy nhiên trong phạm vi cert thì phần lớn chú trọng vào các điểm như sau:

- kubelet đã start hay chưa? Có startup on-boot chưa
- kubectl scheduler đang work properly?
- kubectl-apiserver đang work properly?

Theo tôi tìm hiểu phần lớn các lỗi của các component trên đều có những cái sai như sau:
- Binary path  của kubelet, kube-scheduler hay kube-apiserver bị sai.
- Path certificate bị sai
- Image name của container  kubec-schuduler hay apiserver trong static pod bị sai.

Cố gắng đọc kỹ yêu cầu của đề. Maybe chỉ đơn giản là lệnh start services của kubelet và enable services kubelet là bạn đã hoàn thành câu trả lời.


#### RBAC

- ***Role&ClusterRole**
  
  Chứa tập hợp các rules đại diện cho một số quyền nào đó. Rules này chỉ có tính chất thêm vào chứ ko chặn quyền.

  - Role hoạt động trong 1 name namespace cụ thể
  - Cluster role: hoạt động trong non-namespaced.
   
  *Role*
  
  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    namespace: default
    name: pod-reader
  rules:
  - apiGroups: [""] # "" indicates the core API group
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
  // fast command with kubectl
  $ kubectl create role pod-reader -n default --verb=get,watch,list --resource=pod
  ```
  
  *ClusterRole*

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRole
  metadata:
    name: create-deployment
  rules:
  - apiGroups: ["apps","extensions"]
    resources: ["deployments"]
    verbs: ["create"]

  // fast command
  $ kubectl create clusterrole create-deployment --verb=create --resource=deployments
  ```

  *Tip*
  - Resorce: lower case and end with `s`. Example: pod`s`, deployement`s`, secret`s`, v.v...

  *RoleBinding/Cluster role binding
  
  ``` yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: pod-reader
    namespace: default
  subjects:
  - kind: User
    name: dave # Name is case sensitive
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: Role
    name: pod-reader
    apiGroup: rbac.authorization.k8s.io
  ```
  RoleBinding có thể tham chiếu đến ClusterRole để thiết lập quyền cho namespace nhất định.

  ``` yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: create-deployment
    namespace: staging
  subjects:
  - kind: User
    name: dave # Name is case sensitive
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: ClusterRole
    name: create-deployment
    apiGroup: rbac.authorization.k8s.io
  ```

  ```yaml
  apiVersion: rbac.authorization.k8s.io/v1
  # This cluster role binding allows anyone in the "manager" group to read secrets in any namespace.
  kind: ClusterRoleBinding
  metadata:
    name: read-secrets-global
  subjects:
  - kind: Group
    name: manager # Name is case sensitive
    apiGroup: rbac.authorization.k8s.io
  roleRef:
    kind: ClusterRole
    name: secret-reader
    apiGroup: rbac.authorization.k8s.io
  ```

  *Command*

  ``` shell
  # With user
  $ kubectl rolebinding pod-reader --role=pod-reader -n default --user=dave

  # With services account
  $ kubectl rolebinding pod-reader --role=pod-reader -n default --serviceaccount=default:admin

  $ kubectl rolebinding create-deployment --clusterrole=create-deployment -n staging --user=dave

  $ kubectl clusterrolebinding read-secrets-global --clusterrole=secret-reader --group=manager

  # Check
  $ kubectl auth can-i <verb> <resource> [-n namespace ] --as <user>
  $ kubectl auth can-i <verb> <resource> --as system:serviceaccount:<namespace>:<service_account_name>
  ```

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

## Notes

- Nếu bạn có dự định thi chứng chỉ CKA thì nên đăng ký liền. Vì khi đăng ký, CNCF sẽ cho bạn 2 session simulator trên killer.sh. Mỗi session sẽ được 36 tiếng khi bạn active nó. Bạn hãy tận dụng triệt đẽ 2 session này để tập làm quen cũng như nhuần nhuyễn các câu lệnh.

- Theo kinh nghiệm thì bạn nên active session đầu tiên khi bạn có vài kiến thức cơ bản. Cố gắng làm các task đó mà không cần nhìn vào đáp án. Session thứ 2 bạn nên active trước ngày thi 1->2 ngày để cũng cố lại kiến thức.

- CKA sẽ cho bạn 1 lần retake nếu first attempt của bạn bị failed. Nên đừng quá lo lắng. Nhưng theo những gì tôi tìm hiểu về CKA này thì bạn nên đăng ký tiếp lần 2 vì có khả năng bạn sẽ làm lại bộ đề giống như first attempt. Tuy nhiên đây chỉ là những trick giúp bạn pass CKA chứ ko giúp bạn có nhiều kiến thức về k8s.
