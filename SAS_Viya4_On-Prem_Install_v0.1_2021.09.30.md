## On-Prem 환경에서 Viya 4 설치를 위한 인프라 준비



[toc]

## 1. 시스템 환경 구성 및 사전 준비 사항

- 총 서버 수는 5대
- OS: CentOS 8.4
- Account: root / sasX@XXX , sas / sasX@XXX

### 1.1 서버의 사양

- 172.27.64.70 (korviyaonprem1.apac.sas.com) CPU:16, RAM:64GB, HDD:500GB
- 172.27.64.71 (korviayonprem2.apac.sas.com) CPU:16, RAM:64GB, HDD:500GB
- 172.27.64.72 (korviyaonprem3.apac.sas.com) CPU:16, RAM:64GB, HDD:500GB
- 172.27.64.73 (korviyaonprem4.apac.sas.com) CPU:16, RAM:64GB, HDD:500GB
- 172.27.64.74 (korviyaonprem5.apac.sas.com) CPU:16, RAM:100GB, HDD:500GB

### 1.2 사전 준비 사항

- 5 VMs: RHEL 8, 8 CPUs, 100GB RAM, 500GB Disk Space
- Docker (example configuration)
- Kubernetes Cluster: 1 Master, 4 Worker (CONNECT placed in other node pool and SMP CAS)
- nginx: Ingress Controller
- cert-manager: SAS Viya 4 doesn’t have its own Secret Manager
- default storage class: for persistent volumes
- calico : for k8s network management
- JFrog Artifactory for Private Docker Registry
- Mirror manager for Viya 4 to create local mirror repository aka dark site.
- Tools: ansible, kustomize, helm, yum, kubectl, lens



## 2.  모든 서버에 Docker 설치

### 2.1 마스터 노드(서버) 및 워커 노드 간에 SSH 연결 설정

```bash
# For Master Node (172.27.64.70)
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

# Master to Master
ssh-copy-id 172.27.64.70
ssh 172.27.64.70

# Master to Worker 1
ssh-copy-id 172.27.64.71
ssh 172.27.64.71

# Master to Worker 2
ssh-copy-id 172.27.64.72
ssh 172.27.64.72

# Master to Worker 3
ssh-copy-id 172.27.64.73
ssh 172.27.64.73

# Master to Worker 4
ssh-copy-id 172.27.64.74
ssh 172.27.64.74
```



### 2.2 Centos Repository 생성

```bash
cat << EOF >> /etc/yum.repos.d/centos.repo
[base]
name=CentOS-$releasever - Base
baseurl=http://ftp.daum.net/centos/7/os/$basearch/
gpgcheck=1
gpgkey=http://ftp.daum.net/centos/RPM-GPG-KEY-CentOS-7

[updates]
name=CentOS-$releasever - Updates 
baseurl=http://ftp.daum.net/centos/7/updates/$basearch/
gpgcheck=1
gpgkey=http://ftp.daum.net/centos/RPM-GPG-KEY-CentOS-7

[extras]
name=CentOS-$releasever - Extras 
baseurl=http://ftp.daum.net/centos/7/extras/$basearch/
gpgcheck=1
gpgkey=http://ftp.daum.net/centos/RPM-GPG-KEY-CentOS-7

[centosplus]
name=CentOS-$releasever - Plus 
baseurl=http://ftp.daum.net/centos/7/centosplus/$basearch/
gpgcheck=1
gpgkey=http://ftp.daum.net/centos/RPM-GPG-KEY-CentOS-7
EOF
```



### 2.3 EPEL Repository 생성

```bash
yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
```

```bash
yum repolist
```



### 2.4 Docker 관련 사용자 계정 생성 및 기본 OS 패키지 설치

```bash
groupadd docker
usermod -aG docker root
yum install -y screen firefox lsof telnet libpng12 libXp libXmu net-tools numactl xterm libcgroup libcgroup-tools sshpass yum-utils

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum makecache

mkdir -p /opt/kube-cluster
cd /opt/kube-cluster
```

### 2.5 Docker Dependencies 패키지 설치

```bash
wget http://mirror.centos.org/centos/7/extras/x86_64/Packages/fuse-overlayfs-0.7.2-6.el7_8.x86_64.rpm
yum localinstall -y fuse-overlayfs-0.7.2-6.el7_8.x86_64.rpm

wget http://mirror.centos.org/centos/7/extras/x86_64/Packages/slirp4netns-0.4.3-4.el7_8.x86_64.rpm
yum localinstall -y slirp4netns-0.4.3-4.el7_8.x86_64.rpm

wget http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/libcgroup-0.41-19.el8.x86_64.rpm
yum localinstall -y libcgroup-0.41-19.el8.x86_64.rpm

wget http://mirror.centos.org/centos/8/AppStream/x86_64/os/Packages/socat-1.7.3.3-2.el8.x86_64.rpm
yum localinstall -y socat-1.7.3.3-2.el8.x86_64.rpm

wget http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/glibc-2.28-151.el8.x86_64.rpm
yum localinstall -y glibc-2.28-151.el8.x86_64.rpm

wget http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/libmnl-1.0.4-6.el8.x86_64.rpm
yum localinstall -y libmnl-1.0.4-6.el8.x86_64.rpm

wget http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/libnfnetlink-1.0.1-13.el8.x86_64.rpm
yum localinstall -y libnfnetlink-1.0.1-13.el8.x86_64.rpm

wget http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/libnetfilter_cthelper-1.0.0-15.el8.x86_64.rpm
yum localinstall -y libnetfilter_cthelper-1.0.0-15.el8.x86_64.rpm

wget http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/libnetfilter_cttimeout-1.0.0-11.el8.x86_64.rpm
yum localinstall -y libnetfilter_cttimeout-1.0.0-11.el8.x86_64.rpm

wget http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/libnetfilter_queue-1.0.4-3.el8.x86_64.rpm
yum localinstall -y libnetfilter_queue-1.0.4-3.el8.x86_64.rpm

wget http://mirror.centos.org/centos/8/BaseOS/x86_64/os/Packages/conntrack-tools-1.4.4-10.el8.x86_64.rpm
yum localinstall -y conntrack-tools-1.4.4-10.el8.x86_64.rpm
```



### 2.6  Docker 설치

```bash
wget https://download.docker.com/linux/centos/8/x86_64/test/Packages/containerd.io-1.4.3-3.1.el8.x86_64.rpm
yum localinstall -y containerd.io-1.4.3-3.1.el8.x86_64.rpm --allowerasing 

wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-cli-18.09.9-3.el7.x86_64.rpm
yum localinstall -y docker-ce-cli-18.09.9-3.el7.x86_64.rpm 

wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-18.09.9-3.el7.x86_64.rpm
yum localinstall -y docker-ce-18.09.9-3.el7.x86_64.rpm 

wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-rootless-extras-20.10.4-3.el7.x86_64.rpm
yum localinstall -y docker-ce-rootless-extras-20.10.4-3.el7.x86_64.rpm 

systemctl start docker 
docker version 
systemctl enable docker.service 

systemctl restart docker 
systemctl status -l docker 

systemctl stop firewalld 
systemctl disable firewalld 
swapoff -a
```



### 2.7 Private Docker Registry 위한 Daemon 설정 

Update daemon.json file on all VMs

```bash
cd /etc/docker
vi daemon.json
```

```bash
{
"exec-opts": ["native.cgroupdriver=systemd"],
 "insecure-registries":["172.27.64.70:5000"],
"experimental" : false,
"disable-legacy-registry" : true
}
```



### 2.8 Ansible 설치

```bash
yum install -y python3 python3-setuptools python3-devel openssl-devel
# yum install -y python3 python3-devel openssl-devel

yum install -y python3-pip gcc wget automake libffi-devel python3-six
pip3 install --upgrade pip setuptools
pip3 install ansible==2.9.15
 
ansible --version
ansible localhost -m ping
ansible all -m shell -a "hostname -f"
```



## 3. Ansible을 통해 Kubernetes 마스터 노드 설치  

**As root on master node**

```(bash)
cd /opt/kube-cluster
vi hosts
[masters]
master ansible_host=172.27.64.70 ansible_user=root
[workers]
worker1 ansible_host=172.27.64.71 ansible_user=root
worker2 ansible_host=172.27.64.72 ansible_user=root
worker3 ansible_host=172.27.64.73 ansible_user=root
worker4 ansible_host=172.27.64.74 ansible_user=root
```



### 3.1 Kubernetes 의 utilities and dependencies 설치

Create kube-dependencies.yml to install Kubernetes utilities and dependencies.

```bash
cat << EOF >> /opt/kube-cluster/kube-dependencies.yml
- hosts: all
  become: yes
  tasks:
  - name: disable SELinux
    command: setenforce 0

  - name: disable SELinux on reboot
    selinux:
      state: disabled

  - name: ensure net.bridge.bridge-nf-call-ip6tables is set to 1
    sysctl:
      name: net.bridge.bridge-nf-call-ip6tables
      value: 1
      state: present

  - name: ensure net.bridge.bridge-nf-call-iptables is set to 1
    sysctl:
      name: net.bridge.bridge-nf-call-iptables
      value: 1
      state: present

  - name: add Kubernetes' YUM repository
    yum_repository:
      name: Kubernetes
      description: Kubernetes YUM repository
      baseurl: https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
      gpgkey: https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
      gpgcheck: yes

  - name: install kubelet
    yum:
      name: kubelet-1.18.16
      state: present
      update_cache: true

  - name: install kubeadm
    yum:
      name: kubeadm-1.18.16
      state: present

  - name: start kubelet
    service:
      name: kubelet
      enabled: yes
      state: started

- hosts: master
  become: yes
  tasks:
  - name: install kubectl
    yum:
      name: kubectl-1.18.16
      state: present
      allow_downgrade: yes
EOF
```

```(bash)
ansible-playbook -i hosts kube-dependencies.yml

# (Must see all ok at the end of this script)
# PLAY RECAP ********************************************************************************************************
# master                     : ok=11   changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
# worker1                    : ok=9    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
# worker2                    : ok=9    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
# worker3                    : ok=9    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
# worker4                    : ok=9    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```



### 3.2 Create 'kube-flannel.yml' File

- https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml

- 'extensions/v1beta1' change to 'apps/v1'

```yaml
cat << EOF >> /opt/kube-cluster/kube-flannel.yml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
rules:
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes/status
    verbs:
      - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-amd64
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      hostNetwork: true
      nodeSelector:
        beta.kubernetes.io/arch: amd64
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.io/coreos/flannel:v0.11.0-amd64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: true
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-arm64
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      hostNetwork: true
      nodeSelector:
        beta.kubernetes.io/arch: arm64
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.io/coreos/flannel:v0.11.0-arm64
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-arm64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: true
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-arm
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      hostNetwork: true
      nodeSelector:
        beta.kubernetes.io/arch: arm
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.io/coreos/flannel:v0.11.0-arm
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-arm
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: true
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-ppc64le
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      hostNetwork: true
      nodeSelector:
        beta.kubernetes.io/arch: ppc64le
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.io/coreos/flannel:v0.11.0-ppc64le
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-ppc64le
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: true
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds-s390x
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      hostNetwork: true
      nodeSelector:
        beta.kubernetes.io/arch: s390x
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni
        image: quay.io/coreos/flannel:v0.11.0-s390x
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-s390x
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: true
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        volumeMounts:
        - name: run
          mountPath: /run
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      volumes:
        - name: run
          hostPath:
            path: /run
        - name: cni
          hostPath:
            path: /etc/cni/net.d
        - name: flannel-cfg
          configMap:
            name: kube-flannel-cfg
EOF
```



### 3.3 Create master.yml file to deploy and initialize Kubernetes master server

vi master.yml

```yaml
cat << EOF >> /opt/kube-cluster/master.yml
- hosts: master
  become: yes
  tasks:
  - name: initialize the cluster
    shell: kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version=v1.18.8 --ignore-preflight-errors=all >> cluster_initialized.txt
    args:
      chdir: /home/sas
      creates: cluster_initialized.txt

  - name: create .kube directory
    become: yes
    become_user: sas
    file:
      path: /home/sas/.kube
      state: directory
      mode: 0755

  - name: copy admin.conf to user's kube config
    copy:
      src: /etc/kubernetes/admin.conf
      dest: /home/sas/.kube/config
      remote_src: yes
      owner: sas

  - name: install Pod network
    become: yes
    become_user: sas
    shell: kubectl apply -f kube-flannel.yml >> pod_network_setup.txt
    args:
      chdir: /home/sas
      creates: pod_network_setup.txt
EOF
```

```bash
ansible-playbook -i hosts master.yml

# (Must see all ok at the end of this script)
# PLAY RECAP 
# *******************************************************************************************************************
# master                     : ok=5    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### 3.4 sudo 권한 설정 

```bash
vi /etc/sudoers
sas  ALL=(ALL:ALL) ALL
```

As normal user like sas

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
```



Now as our Kube master is ready, we are ready to join our nodes to the cluster

### 3.5 Create worker.yml file to join nodes

vi workers.yml

```yaml
cat << EOF >> /opt/kube-cluster/worker.yml
- hosts: master
  become: yes
  gather_facts: false
  tasks:
  - name: get join command
    shell: kubeadm token create --print-join-command
    register: join_command_raw

  - name: set join command
    set_fact:
      join_command: "{{ join_command_raw.stdout_lines[0] }}"

- hosts: workers
  become: yes
  tasks:
  - name: join cluster
    shell: "{{ hostvars['master'].join_command }} >> node_joined.txt"
    args:
      chdir: $HOME
      creates: node_joined.txt
EOF
```

```bash
ansible-playbook -i hosts workers.yml

# PLAY RECAP 
# ******************************************************************************************************************
# master                     : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
# worker1                    : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
# worker2                    : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
# worker3                    : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
# worker4                    : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```



### 3.6 Kubernetes 노드의 status 확인

- Run command on master to get status as sas user

```bash
kubectl get nodes

# NAME                          STATUS   ROLES    AGE     VERSION
# korviayonprem2.apac.sas.com   Ready    <none>   4m8s    v1.18.16
# korviyaonprem1.apac.sas.com   Ready    master   87m     v1.18.16
# korviyaonprem3.apac.sas.com   Ready    <none>   4m4s    v1.18.16
# korviyaonprem4.apac.sas.com   Ready    <none>   4m10s   v1.18.16
# korviyaonprem5.apac.sas.com   Ready    <none>   4m11s   v1.18.16
```



### 3.7 노드에 worker 레이블 적용

```bash
kubectl label node korviayonprem2.apac.sas.com node-role.kubernetes.io/worker=worker
kubectl label node korviyaonprem3.apac.sas.com node-role.kubernetes.io/worker=worker
kubectl label node korviyaonprem4.apac.sas.com node-role.kubernetes.io/worker=worker
kubectl label node korviyaonprem5.apac.sas.com node-role.kubernetes.io/worker=worker

kubectl get nodes

# NAME                          STATUS   ROLES    AGE   VERSION
# korviayonprem2.apac.sas.com   Ready    worker   15m   v1.18.16
# korviyaonprem1.apac.sas.com   Ready    master   98m   v1.18.16
# korviyaonprem3.apac.sas.com   Ready    worker   15m   v1.18.16
# korviyaonprem4.apac.sas.com   Ready    worker   15m   v1.18.16
# korviyaonprem5.apac.sas.com   Ready    worker   15m   v1.18.16
```



## 4. kube cluster에서 기타 components 설치

### 4.1 Install Kustomize (as root)

```bash
curl -o install_kustomize.sh https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh
bash install_kustomize.sh 3.7.0 /usr/local/bin
# kustomize installed to /usr/local/bin/kustomize

kustomize version
# v3.7.0
```

### 4.2 Install helm

```bash
curl -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
bash get_helm.sh
# helm installed into /usr/local/bin/helm
```

### 4.3 Install Calico (as sas)

```bash
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml
watch kubectl get pods -n kube-system
```

### 4.4 Install cert-manager (as sas)

```bash
kubectl create namespace cert-manager
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.4.0/cert-manager.crds.yaml
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.yaml
watch kubectl get pods -n cert-manager
kubectl -n cert-manager describe deployments/cert-manager | grep Image
```

### 4.5 Validation of cert-manager (as root)

```bash
cat <<EOF >  test-resources.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
  - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
```

```bash
kubectl apply -f test-resources.yaml
# namespace/cert-manager-test created
# issuer.cert-manager.io/test-selfsigned created
# certificate.cert-manager.io/selfsigned-cert created

kubectl describe certificate -n cert-manager-test
# .....
# Events:
#  Type    Reason     Age    From          Message
#  ----    ------     ----   ----          -------
#  Normal  Issuing    2m14s  cert-manager  Issuing certificate as Secret does not exist
#  Normal  Generated  2m14s  cert-manager  Stored new private key in temporary Secret resource "selfsigned-cert-hqgzk"
#  Normal  Requested  2m14s  cert-manager  Created new CertificateRequest resource "selfsigned-cert-8nx45"
#  Normal  Issuing    2m14s  cert-manager  The certificate has been successfully issued

kubectl delete -f test-resources.yaml
# namespace "cert-manager-test" deleted
# issuer.cert-manager.io "test-selfsigned" deleted
# certificate.cert-manager.io "selfsigned-cert" deleted
```



### 4.6 Remove cert-manager (as sas)

```bash
kubectl get clusterrole | grep cert-manager

kubectl delete clusterrole \ 
cert-manager-cainjector \ 
cert-manager-controller-certificates \
cert-manager-controller-challenges \
cert-manager-controller-clusterissuers \
cert-manager-controller-ingress-shim \
cert-manager-controller-issuers \
cert-manager-controller-orders \
cert-manager-edit cert-manager-view

kubectl delete ns cert-manager
```



### 4.7 Install nginx

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
kubectl create ns nginx
helm install my-nginx --namespace nginx \
--set controller.service.type=NodePort \
--set controller.service.nodePorts.http=30001 \
--set controller.service.nodePorts.https=30002 \
--set controller.extraArgs.enable-ssl-passthrough="" \
--set controller.autoscaling.enabled=true \
--set controller.autoscaling.minReplicas=2 \
--set controller.autoscaling.maxReplicas=5 \
--set controller.resources.requests.cpu=100m \
--set controller.resources.requests.memory=500Mi \
--set controller.autoscaling.targetCPUUtilizationPercentage=90 \
--set controller.autoscaling.targetMemoryUtilizationPercentage=90 \
--set controller.admissionWebhooks.patch.image.repository=jettech/kube-webhook-certgen \
--set controller.admissionWebhooks.patch.image.tag=v1.5.0 \
--version 3.20.1 \
ingress-nginx/ingress-nginx

# NAME: my-nginx
# LAST DEPLOYED: Fri Sep 24 14:26:10 2021
# NAMESPACE: nginx
# STATUS: deployed
# REVISION: 1
# TEST SUITE: None
# NOTES:
# The ingress-nginx controller has been installed.
# Get the application URL by running these commands:
#  export HTTP_NODE_PORT=30001
#  export HTTPS_NODE_PORT=30002
#  export NODE_IP=$(kubectl --namespace nginx get nodes -o jsonpath="{.items[0].status.addresses[1].address}")

#  echo "Visit http://$NODE_IP:$HTTP_NODE_PORT to access your application via HTTP."
#  echo "Visit https://$NODE_IP:$HTTPS_NODE_PORT to access your application via HTTPS."

```



### 4.8 Remove nginx

```bash
helm uninstall my-nginx --namespace nginx
kubectl delete namespace nginx
```



## 5. Install nfs-client storage class

In order to install storage class, you need to have a common mount point between all the servers. You either you can set it up on your own using the commands below on one of the servers and share it with all other servers, or you can use a NAS mount, mounted on all servers.

### 5.1 Setting up nfs server

```bash
# nfs host = nfshost.sas.com
# nfs host = korviyaonprem1.apac.sas.com
nfs host = 172.27.64.70
nfs path = /viya4nfs
As root

vi /etc/exports
/viya4nfs *(rw,sync,no_root_squash,insecure)
systemctl restart nfs-server.service
```

### 5.2 Setting up nfs clients As root

```bash
mkdir -p /viya4nfs
chown -R sas:sas /viya4nfs
vi /etc/fstab
# nfshost.sas.com:/viya4nfs /viya4nfs nfs defaults 0 0
172.27.64.70:/viya4nfs /viya4nfs nfs defaults 0 0
# showmount -e nfshost.sas.com
showmount -e 172.27.64.70
mount -a
```



### 5.3 Now as shared mount is configured, lets configure storage class (as sas)

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo update

helm search repo stable

kubectl create ns nfs-client
helm install my-nfs-client stable/nfs-client-provisioner --namespace nfs-client --set nfs.server=172.27.64.70 --set nfs.path=/viya4nfs

kubectl get storageclass

# NAME         PROVISIONER                                          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
# nfs-client   cluster.local/my-nfs-client-nfs-client-provisioner   Delete          Immediate           true                   18s
```

Make Storage Class Default: Just installing storage class doesn’t work, you need to make it default storage class in your kube cluster otherwise your many sas pods like cachelocator, cosul, rabbitmq will stuck in pending state forever.

```bash
kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get sc

# NAME                   PROVISIONER                                          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
# nfs-client (default)   cluster.local/my-nfs-client-nfs-client-provisioner   Delete          Immediate           true                   2m7s
```



### 5.4 Remove nfs-client storage class

```bash
helm uninstall my-nfs-client -n nfs-client

kubectl delete ns nfs-client
```



## 6. Validate Kubernetes Cluster

```bash
kubectl cluster-info
kubectl version --short
kubelet --version
kubeadm version
kustomize version --short
kubectl get nodes

#kubectl get pods -n kube-system -o wide <to list all pods running on all nodes under kube-system namespace>
kubectl get pods -n kube-system -o wide

#kubectl get pods -A -o wide | grep <node name> <to list node wise pods>
kubectl get pods -A -o wide | grep korviyaonprem1.apac.sas.com

kubectl get namespace <to list all namespaces>

kubectl config view --minify | grep namespace: <to know current namespace>
kubectl config set-context --current --namespace=default <to set default namespace>

kubectl get svc <to list services running>

#kubectl run nginx --image=nginx --namespace=<insert-namespace-name-here>
kubectl run nginx --image=nginx --namespace=kube-system
kubectl get pods -A
kubectl delete pod nginx -n kube-system

# kubectl get pods --namespace=<insert-namespace-name-here>
kubectl get pods -n kube-system

kubectl get storageclass

kubectl get clusterrole
kubectl get clusterrolebindings.rbac.authorization.k8s.io
kubectl get clusterrolebindings

# kubectl --namespace <your namespace> logs -f <name of the pod>
kubectl -n kube-system logs -f kube-proxy-57jpc

kubectl get pvc <PersistentVolumeClaims>
kubectl describe pvc <pvc name>


kubectl get psp <list Pod Security Policy>

## Otsuda ne rabotaet mozet udalit
kubectl get psp restricted -o custom-columns=NAME:.metadata.name,"SECCOMP":".metadata.annotations.seccomp\.security\.alpha\.kubernetes\.io/allowedProfileNames"
kubectl auth can-i use podsecuritypolicy/restricted | podsecuritypolicy/psp.restricted
kubectl auth can-i use podsecuritypolicy/restricted --as=system:serviceaccount:default:default
kubectl delete psp <psp name>
kubectl delete clusterrole <psp cluster role>
kubectl delete clusterrolebinding <psp cluster role binding>
```



## 7. Removing Kubernetes

Suppose you terribly messed up with something and don’t have time to troubleshoot, you can remove what you have installed and start fresh. But same time it’s a caution, if you run reset command by mistake, you will lose everything and have to redo everything again, from initializing the cluster to rejoin the nodes.

### 7.1 On Kube Master

```
# kubeadm reset -f
# rm -rf /etc/cni /etc/kubernetes /var/lib/dockershim /var/lib/etcd /var/lib/kubelet /var/run/kubernetes /home/sas/.kube/*
# iptables -F &amp;&amp; iptables -X
# iptables -t nat -F &amp;&amp; iptables -t nat -X
# iptables -t raw -F &amp;&amp; iptables -t raw -X
# iptables -t mangle -F &amp;&amp; iptables -t mangle -X
# systemctl restart docker
# systemctl status -l docker
```

### 7.2 On all worker nodes

```
# systemctl daemon-reload
# systemctl stop kubelet
# rm -f /etc/kubernetes/kubelet.conf
# rm -f /etc/kubernetes/pki/ca.crt
# rm -f /etc/kubernetes/bootstrap-kubelet.conf
# cd /var/lib/kubelet/pki
# rm -f kubelet-client*
```



## 8. Create a Mirror Registry at a Dark Site

To deploy SAS Viya to a dark site, download the contents to a local directory first, from where you can copy the content to any isolated network. In my case it was the same server but at a client site, you might have to download it somewhere due to lack of internet connectivity, bring it inside on local server through some external storage device. From there, you can upload SAS Viya content to an internal/private docker registry.

I am not providing you the steps to download the mirror manager. Please follow the steps in official documentation [Download SAS Mirror Manager](https://go.documentation.sas.com/doc/en/sasadmincdc/v_016/dplyml0phy0dkr/n1h0rgtr10fpnfn1mg0s8fgfuof8.htm#p1t0kfeu3dm5dbn1j4h0yewzb4nr) to obtain the SAS Viya software and entitlements and to download SAS Mirror Manager. Once done, please follow the steps as below:

**Let’ deploy a Private Container Registry first**

Here, in my example, I am using container registry from JFrog. There are many options available like harbor and there are no such reason to choose this one. I have worked with this before and it is easy to setup. I am not deploying it on Kubernetes cluster to avoid any extra load on the cluster. It is called standalone installation using Linux archive.

### 8.1 Download the JFrog Container Registry

```bash
mkdir /opt/jfrog
cd /opt/jfrog

cat << EOF >> /etc/profile
export JFROG_HOME=/opt/jfrog
EOF
source /etc/profile
echo $JFROG_HOME

wget https://releases.jfrog.io/artifactory/bintray-artifactory/org/artifactory/jcr/jfrog-artifactory-jcr/[RELEASE]/jfrog-artifactory-jcr-[RELEASE]-linux.tar.gz

tar -xvf 'jfrog-artifactory-jcr-[RELEASE]-linux.tar.gz'
rm -f 'jfrog-artifactory-jcr-[RELEASE]-linux.tar.gz'
mv artifactory-jcr-7.25.7/ artifactory
```



### 8.2 Customize the product configuration – system.yaml file

 You can configure all your system settings using the system.yaml file located in the

```bash
cd $JFROG_HOME/artifactory/var/etc
cp system.full-template.yaml system.yaml
```



There are many parameters here which you can configure as per your requirements. I changed the following lines:

```bash

## ARTIFACTORY TEMPLATE
artifactory:
  port: 8081

    ## Artifactory connector settings
    connector:
      maxThreads: 1024

    httpsConnector:
      ## Enable connector with SSL/TLS
      enabled: false

      ## Port to use for the HTTPS connector
      port: 8443

      ## Certificate file to use
      # certificateFile: "$JFROG_HOME/artifactory/var/etc/artifactory/security/ssl/server.crt"
      certificateFile: "$JFROG_HOME/artifactory/var/etc/artifactory/security/ssl/ca.crt"
      ## Certificate key file to use.
      # certificateKeyFile: "$JFROG_HOME/artifactory/var/etc/artifactory/security/ssl/server.key"
      certificateKeyFile: "$JFROG_HOME/artifactory/var/etc/artifactory/security/ssl/ca.key"      
```

Note: I created self-signed certificates for this.



### 8.3 Install the service.

```bash
$JFROG_HOME/artifactory/app/bin/installService.sh
```



### 8.4 Start|Stop service

```bash
# systemctl &lt; start|stop|status&gt; artifactory.service
systemctl start artifactory.service
systemctl status artifactory.service
```

URL to login

```bash
# http://registry-host.in.sas.com:8082/ui/
http://172.27.64.70:8082/ui/
```

Default userid and password to login first time
admin/password (SasX@XXX)



### 8.5 Check the logs

```bash
tail -f $JFROG_HOME/artifactory/var/log/console.log
```

You also need to setup a reverse proxy server to access this registry through SSL:
You can install or use the default apache. You can directly download this file from http settings section from the UI.

```bash
yum install httpd
systemctl start httpd
systemctl status httpd
```

```ba
yum info mod_ssl
yum install mod_ssl -y
```

```bash
yum info openssl

```

[Setting up an SSL secured Webserver with CentOS](https://wiki.centos.org/HowTos/Https)

```bash
# Generate private key 
openssl genrsa -out ca.key 2048 

# Generate CSR 
openssl req -new -key ca.key -out ca.csr

# Generate Self Signed Key
openssl x509 -req -days 365 -in ca.csr -signkey ca.key -out ca.crt

# Copy the files to the correct locations
# cp ca.crt /etc/pki/tls/certs
# cp ca.key /etc/pki/tls/private/ca.key
# cp ca.csr /etc/pki/tls/private/ca.csr

mkdir /opt/jfrog/artifactory/var/etc/artifactory/ssl
chown artifactory:artifactory ssl/

cp ca.key /opt/jfrog/artifactory/var/etc/artifactory/ssl
cp ca.crt /opt/jfrog/artifactory/var/etc/artifactory/ssl

```



```bash
cd /etc/httpd/conf.d
vi artifactory.conf
```

```bash
###########################################################
## this configuration was generated by JFrog Artifactory ##
###########################################################

Listen 8443 https

<VirtualHost *:8082>

ProxyPreserveHost On
ProxyAddHeaders off

ServerName 172.27.64.70
ServerAlias *.172.27.64.70
ServerAdmin server@admin

## Application specific logs
## ErrorLog /etc/httpd/logs/registry-host.in.sas.com-error.log
## CustomLog /etc/httpd/logs/registry-host.in.sas.com-access.log combined

AllowEncodedSlashes On
RewriteEngine on

RewriteCond %{SERVER_PORT} (.*)
RewriteRule (.*) - [E=my_server_port:%1]
## NOTE: The 'REQUEST_SCHEME' Header is supported only from apache version 2.4 and above
RewriteCond %{REQUEST_SCHEME} (.*)
RewriteRule (.*) - [E=my_scheme:%1]

RewriteCond %{HTTP_HOST} (.*)
RewriteRule (.*) - [E=my_custom_host:%1]
RewriteCond "%{REQUEST_URI}" "^/(v1|v2)/"

RewriteRule ^(/)?$ /ui/ [R,L]
RewriteRule ^/ui$ /ui/ [R,L]

RequestHeader set Host %{my_custom_host}e
RequestHeader set X-Forwarded-Port %{my_server_port}e
## NOTE: {my_scheme} requires a module which is supported only from apache version 2.4 and above
RequestHeader set X-Forwarded-Proto %{my_scheme}e
RequestHeader set X-JFrog-Override-Base-Url %{my_scheme}e://172.27.64.70:%{my_server_port}e
ProxyPassReverseCookiePath / /

ProxyRequests off
ProxyPreserveHost on
ProxyPass "/artifactory/" http://172.27.64.70:8081/artifactory/
ProxyPassReverse "/artifactory/" http://172.27.64.70:8081/artifactory/
ProxyPass "/" http://172.27.64.70:8082/
ProxyPassReverse "/" http://172.27.64.70:8082/

</VirtualHost>
<VirtualHost *:8443>

ProxyPreserveHost On
ProxyAddHeaders off

ServerName 172.27.64.70
ServerAlias *.172.27.64.70
ServerAdmin server@admin

# By KORAND
SSLEngine on
SSLCertificateFile /etc/pki/tls/certs/localhost.crt
SSLCertificateKeyFile /etc/pki/tls/private/localhost.key
#SSLCertificateChainFile /etc/pki/tls/certs/vault-ca.crt
SSLProxyEngine on

## Application specific logs
ErrorLog /etc/httpd/logs/172.27.64.70-error.log
CustomLog /etc/httpd/logs/172.27.64.70-access.log combined

AllowEncodedSlashes On
RewriteEngine on

RewriteCond %{SERVER_PORT} (.*)
RewriteRule (.*) - [E=my_server_port:%1]
## NOTE: The 'REQUEST_SCHEME' Header is supported only from apache version 2.4 and above
RewriteCond %{REQUEST_SCHEME} (.*)
RewriteRule (.*) - [E=my_scheme:%1]

RewriteCond %{HTTP_HOST} (.*)
RewriteRule (.*) - [E=my_custom_host:%1]
RewriteCond "%{REQUEST_URI}" "^/(v1|v2)/"

RewriteRule ^(/)?$ /ui/ [R,L]
RewriteRule ^/ui$ /ui/ [R,L]

RequestHeader set Host %{my_custom_host}e
RequestHeader set X-Forwarded-Port %{my_server_port}e
## NOTE: {my_scheme} requires a module which is supported only from apache version 2.4 and above
RequestHeader set X-Forwarded-Proto %{my_scheme}e
RequestHeader set X-JFrog-Override-Base-Url %{my_scheme}e://172.27.64.70:%{my_server_port}e
Header always set Strict-Transport-Security ""
ProxyPassReverseCookiePath / /

ProxyRequests off
ProxyPreserveHost on
ProxyPass "/artifactory/" http://172.27.64.70:8081/artifactory/
ProxyPassReverse "/artifactory/" http://172.27.64.70:8081/artifactory/
ProxyPass "/" http://172.27.64.70:8082/
ProxyPassReverse "/" http://172.27.64.70:8082/
</VirtualHost>
```

```ba
systemctl restart httpd
```

Now you can access the private container registry on the following URL:

```
https://172.27.64.70:8443/ui
```

You have to login to registry url and create a local repository,

Administration >> Repositories >> click on add repository.

Provide Repository Key Name, select apri version V2, click on save and finish button

Now, we will download the local copy of SAS Viya 4 contents through mirror manager and load them to JFrog registry.

```bash
mkdir -p ~/project/deploy
cd ~/project/deploy
```

```bash
wget https://support.sas.com/installation/viya/4/sas-mirror-manager/lax/mirrormgr-linux.tgz
tar -xzvf mirrormgr-linux.tgz
ls -l
# drwxr-xr-x. 4 sas sas       32 Aug  5 05:13 esp-edge-extension
# drwxr-xr-x. 2 sas sas     4096 Mar 30 02:14 licenses
# -rwxr-xr-x. 1 sas sas 14484373 Aug  5 05:14 mirrormgr
# -rw-rw-r--. 1 sas sas  7421398 Aug  5 05:28 mirrormgr-linux.tgz
# -rw-r--r--. 1 sas sas      779 Aug  5 05:14 README.md
# -rwx------. 1 root root     4215 Sep 27 12:28 SASViyaV4_09YR1Z_certs.zip
# -rwx------. 1 root root   805711 Sep 27 12:28 SASViyaV4_09YR1Z_lts_2021.1_20210926.1632672493450_deploymentAssets_2021-09-27T023523.tgz
# -rwx------. 1 root root    34355 Sep 27 12:28 SASViyaV4_09YR1Z_lts_2021.1_license_2021-09-27T023504.jwt
```

### 8.6 Run mirror manager to download the software and upload to Repository

```bash
mkdir -p /data/sas_repo

Check what all releases are available
./mirrormgr --deployment-data SASViyaV4_09YR1Z_certs.zip list remote cadences
# ...
# lts-2021.1
# ...

Check the list of remote cadences available
./mirrormgr --deployment-data SASViyaV4_09YR1Z_certs.zip list remote cadence releases
# ...
# Releases for Cadence lts-2021.1
# ...
# 20210926.1632672493450
# ...

Check the size of remote repository
./mirrormgr --deployment-data SASViyaV4_09YR1Z_certs.zip list remote repos size --latest
```

Finally when you have decided on which release and cadence to use, you can start the download of the repository. I decided to go with the latest one.

```bash
./mirrormgr mirror registry --deployment-data SASViyaV4_09YR1Z_certs.zip --path /home/sas/sas_repo --cadence lts-2021.1 --release 20210926.1632672493450 --latest --log-file SASViyaV4_09YR1Z_Image_Download_210928.log --debug --workers 100
```

Once this is downloaded completely, we can push the contents to the JFrog registry:

```bash
./mirrormgr mirror registry --deployment-data SASViyaV4_09YR1Z_certs.zip --path /home/sas/sas_repo --destination 172.27.64.70:8443/viya --username admin --password Sask@123 --push-only --insecure
```

Finally when you are done, you can again visit the JFrog registry dashboard and check if packages re uploaded to registry successfully:



## 9. 참조

- Shatrughan Saxena의 Blog: [SAS Viya 4 Deployment on Open Source K8s cluster running on Bare Metal](http://sww.sas.com/blogs/wp/technical-insights/3458/sas-viya-4-deployment-on-open-source-k8s-cluster-running-on-bare-metal-part-1-preparation-of-bare-metal-infrastructure-for-sas-viya-4-deployment/sinsxs/2021/08/17)

- Shatrughan Saxena의 GitLab:  [SAS Viya 4 Deployment Pre-requisites and Deployment Process](https://gitlab.sas.com/Shatrughan.Saxena/sasviya4)

- GEL Course: [SAS Viya 4 Deployment Workshop](https://gitlab.sas.com/GEL/workshops/PSGEL255-deploying-viya-4.0.1-on-kubernetes)

- SAS Docs: [SAS Viya 4 Deployment Guide](https://go.documentation.sas.com/doc/en/sasadmincdc/v_016/dplyml0phy0dkr/n0phs7qjof8s4dn17m7d4ft2bkqk.htm)

