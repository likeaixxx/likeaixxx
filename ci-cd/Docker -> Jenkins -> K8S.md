本文使用的环境是MacOs-M1Pro+Parallels-Desktop+Linux-Stream9

### 宿主机MacOS: 
OrbStack: https://orbstack.dev/
```shell
brew install orbstack
```

### 在宿主机安装Jenkins
由于我们要在Jenkins的容器内访问宿主机的docker.sock来使用宿主机的docker做某些事情,所以我们使用Dockerfile自己打包一个Jenkins镜像
```Dockerfile
FROM jenkins/jenkins:2.460
USER root
RUN apt-get update && \
    apt-get install -y ca-certificates curl gnupg lsb-release && \
    curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg && \
    echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
    $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    apt-get update && \
    apt-get install -y docker-ce-cli && \
    rm -rf /var/lib/apt/lists/*
USER jenkins
```
启动这个镜像
```shell
docker run -d -p 8703:8080 -p 50000:50000 -v ~/repo/jenkins:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock --privileged --name jenkins jenkins-with-docker
```
此处指定并挂载Jenkins的工作目录,并且把宿主机的docker.sock挂载到Jenkins容器内部,但是某些时候docker.sock会出现权限异常的问题,所以我们还需要手动修改一下权限,此处我直接给了最高权限
```shell
docker exec -it -u root jenkins bash
chmod -R 777 /var/run/docker.sock
```
然后直接访问 localhost:8073 端口就能看到Jenkins服务正常运行了.
可以简单测试一下,新建一个流水线项目如图
![Arc screenshot 0625000635.png](https://raw.githubusercontent.com/like-aiquan/like-aiquan/main/attachments/Arc%20screenshot%200625000635.png)
![Arc 0625000636 screenshot.png](https://raw.githubusercontent.com/like-aiquan/like-aiquan/main/attachments/Arc%200625000636%20screenshot.png)

```shell
pipeline {
    agent any
    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Branch name to build')
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: "${params.BRANCH_NAME}", url: 'https://github.com/like-aiquan/lyrics.git'
            }
        }
        stage('Build Docker Image') {
            steps {
                dir('server') {
                    script {
                        def sanitizedBranchName = "${params.BRANCH_NAME}".replaceAll("/", "-")
                        def imageName = "lyrics-server:${sanitizedBranchName}"
                        sh "docker build -t ${imageName} ."
                    }
                }
            }
        }
        
        stage('Deploy') {
            steps {
                script {
                    def sanitizedBranchName = "${params.BRANCH_NAME}".replaceAll("/", "-")
                    def imageName = "lyrics-server:${sanitizedBranchName}"
                    
                    // Stop and remove existing container (if exists)
                    sh "docker stop lyrics-server || true"
                    sh "docker rm lyrics-server || true"
                    
                    // Run new container
                    sh "docker run -d --name lyrics-server -p 8331:8331 ${imageName}"
                }
            }
        }
    }
}
```

运行这个项目你应该就能看到宿主机的docker运行了一个golang的web项目.这个项目是我一个简单的后台应用项目.可以看看接口自己测试一下是否可以访问

### Linux安装K8S环境
设置域名->ip解析,这是我三个虚拟机的IP, 想要在宿主机访问还需要设置宿主机的hosts文件,这里就不写了.
```shell
cat >> /etc/hosts << EOF
10.211.55.16 kubernetes-node2.likeai kubernetes-node2
10.211.55.12 kubernetes-node1.likeai kubernetes-node1
10.211.55.14 kubernetes-master.likeai kubernetes-master
EOF
```
安装虚拟机可以选择最小安装然后再安装必要的软件即可
```shell
yum install -y wget tree bash-completion lrzsz psmisc net-tools vim
```
设置防火墙和SELINUX(这个是官方推荐关闭)
```shell
systemctl stop firewalld
systemctl disable firewalld
sed -i  '/^SELINUX=/ c  SELINUX=disabled' /etc/selinux/config
setenforce 0
```
安装时间插件,然后三台机器需要做同一时间处理
```shell
yum install chrony -y
vim /etc/chrony.conf
server ntp1.aliyun.com iburst
systemctl enable --now chronyd
chronyc sources
```
禁用swap
```shell
 swapoff -a
 sed -i 's/.*swap.*/#&/' /etc/fstab
```
设置桥接流量
```shell
cat >> /etc/sysctl.d/k8s.conf << EOF
#内核参数调整
vm.swappiness=0
#配置iptables参数，使得流经网桥的流量也经过iptables/netfilter防火墙
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```
br_netfilter, containerd
```shell
modprobe br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
sysctl -p /etc/sysctl.d/k8s.conf

sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf update
sudo dnf install -y containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
#替换配置开启SystemdCgroup
sed -e 's/SystemdCgroup = false/SystemdCgroup = true/g' -i /etc/containerd/config.toml
#镜像源..大家懂的
sed -e 's/k8s.gcr.io/pause:3.6/registry.aliyuncs.com/google_containers/pause:3.6' -i /etc/containerd/config.toml
systemctl restart containerd
sudo systemctl enable containerd
```
配置crictl(监控容器的)
```shell
cat <<EOF > /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock 
image-endpoint: unix:///run/containerd/containerd.sock 
timeout: 10
pull-image-on-create: true
EOF
```
安装K8S组件
```shell
sudo tee /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-aarch64
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubelet-1.28.2 kubeadm-1.28.2 kubectl-1.28.2
sudo systemctl enable kubelet
```

#### Node
初始化
```shell
kubeadm init --kubernetes-version=v1.28.2 --apiserver-advertise-address=10.211.55.14 --image-repository registry.aliyuncs.com/google_containers --pod-network-cidr=10.244.0.0/16 --control-plane-endpoint=kubernetes-master.likeai --cri-socket=unix:///run/containerd/containerd.sock 
```

`--apiserver-advertise-address`
```
设定 Kubernetes API server 的公共 IP 地址。这是集群内节点和 Kubernetes API server 通信的 IP 地址。
这个 IP 地址需要是集群内所有节点都可以访问的。
如果你没有明确设置该地址，Kubernetes 将默认使用 master 节点的第一个非环回网络接口的 IP 地址。
```

`--image-repository`
```
切换成阿里云的,毕竟...
```

`--pod-network-cidr` 
```
用于设置 Pod 网络的 IP 地址范围, 后续会选用flannel作为网络插件,所以此处直接写死为flannel的默认配置最好
```

`--control-plane-endpoint`
```
用来指定 Kubernetes 控制平面（control plane）的访问地址。控制平面通常包含 API Server、scheduler、controller-manager 等核心组件，它们共同负责管理和调度整个 K8s 集群。
```

初始化完毕后, 直接使用控制台打印的命令在两个node执行即可
```shell
kubeadm join kubernetes-master.likeai:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:67ea537411822fe684d1ddb984802da62a4f22aa1c32fefe7c3404bb8f3f52e0
```

### Jenkins -> K8S
这个文档写的太详细了, 我就不重复了. [Jenkins如何链接云山K8S](https://github.com/sunweisheng/Jenkins/blob/master/Jenkins-Kubernetes.md)

K8S链接镜像私服, 我用的阿里云个人版免费的如下:
```shell
kubectl create secret docker-registry private-reg-cred \
  --docker-server=registry.cn-beijing.aliyuncs.com \
  --docker-username=...... \
  --docker-password=...... \
  --docker-email=.......
```
按名称配置参数即可

测试:
如最开始Docker创建Jenkins容器一样的测试方法, 重新开一个项目Git项目地址填写
https://github.com/like-aiquan/k8s-test.git/
这是我一个测试仓库,然后pipeline脚本填入如下:
```pipeline
pipeline {
    agent any
    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Branch name to build')
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: "${params.BRANCH_NAME}", url: 'https://github.com/like-aiquan/k8s-test.git'
            }
        }
        
        stage('Build Project') {
            steps {
                script {
                    sh "./gradlew build"
                }
            }    
        }
        
        stage('Build Image') {
            steps {
                script {
                    def sanitizedBranchName = "${params.BRANCH_NAME}".replaceAll("/", "-")
                    # 别用我的!!!
                    def imageName = "registry.cn-beijing.aliyuncs.com/like_ai/k8s-test:${sanitizedBranchName}"
                    sh "docker build -t ${imageName} ."
                }
            }
        }
        
        stage('Publish Image') {
            steps {
                script {
                    def sanitizedBranchName = "${params.BRANCH_NAME}".replaceAll("/", "-")
                    # TODO 账户、密码
                    sh(script: 'docker login --username=...... --password=...... registry.cn-beijing.aliyuncs.com', mask: true)
                    # 别用我的!!!
                    sh "docker push registry.cn-beijing.aliyuncs.com/like_ai/k8s-test:${sanitizedBranchName}"
                }
            }
        }
        
        stage('Deploy') {
            steps {
                script {
                    def tag = "${params.BRANCH_NAME}".replaceAll("/", "-")
                    configFileProvider([configFile(fileId: '58c7fc70-e103-486a-84bc-ab68e5a3a3c7', targetLocation: 'kubernetes-admin')]) {
                        sh 'cp /var/jenkins_home/k8s/k8s-test/k8s-test.yaml /var/jenkins_home/k8s/k8s-test/k8s-test-tmp.yaml'
                        sh "sed -i 's/TAG/${tag}/g' /var/jenkins_home/k8s/k8s-test/k8s-test-tmp.yaml"
                        sh "kubectl apply -f /var/jenkins_home/k8s/k8s-test/k8s-test-tmp.yaml --kubeconfig=kubernetes-admin"
                    }
                }
            }
        }
    }
}
```

[configFile插件使用](https://blog.csdn.net/catoop/article/details/119358736)

可以看到我复制了一份k8s-test.yaml,这是k8s的模版
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s
  labels:
    app: k8s-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-test
  template:
    metadata:
      labels:
        app: k8s-test
    spec:
      containers:
      - name: k8s-test
        image: registry.cn-beijing.aliyuncs.com/like_ai/k8s-test:TAG
        ports:
        - containerPort: 8080
      imagePullSecrets:
      - name: private-reg-cred
```

使用sed命令将TAG占位符替换成了真正的tag, K8S在已经登陆阿里云的情况下直接发布即可
最后,注意下模版文件的地址, 我放到了Jenkins的工作目录下,创建了一个名为k8s的文件夹,然后按照项目去化分模版存放位置