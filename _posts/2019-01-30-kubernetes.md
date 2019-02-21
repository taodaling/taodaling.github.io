---
categories: tool
layout: post
---
- Table
{:toc}
# 注意

本文基本是对于《Kubernetes in Action》的翻译和个人整理。

# 概论

## 微服务带来的挑战

大型的应用包含一些紧密耦合的组件，这些组件必须作为一个整体被一同开发，部署和管理。由于应用是以进程的形式启动的，对任意组件的修改都需要重启整个应用才能生效。并且组件缺乏明确边界会导致开发复杂度的不断增加，同时作为结果导致整个系统质量的不断恶化。

运行一个巨大的应用往往对硬件的要求也会更高，这类应用一般会要求少数的几台强力服务器来为应用提供足够的资源。要处理系统不断上升的负载，你要么通过继续增加服务器的性能来提供垂直扩展，或者通过增加机器和服务数来进行水平扩展。垂直扩展不需要变更应用代码，实现起来比较容器，但是服务器的配置具有上限，并且服务器的费用的增长速度回比负载增长的速度更快。而水平扩展虽然可以通过使用廉价服务器降低开支，但是要求应用中的每一个组件都必需支持水平扩展，这个条件是很难实现的，比如关系型数据库就很难支持水平扩展。

这一系列的问题都强迫我们将大型应用切分为更小的独立组件，每个组件都可以独立部署，这些组件称为微服务。每个微服务都作为独立进程运行，并通过网络与其他组件进行通讯。

由于微服务更小，所以开发复杂度更低，并且它们是独立部署的，因此我们仅需要修改必要的组件，并进行重启，而不需要变动整个应用。

而由于不同的组件运行在不同的服务上，我们可以为每个组件分配不同的资源，定义各自的副本数。这样能更充分地利用计算资源。

但是微服务也有一些缺点，随着服务数量的上升，部署成了一个难题，因为我们部署时不仅仅需要启动它们，还需要维护它们之间的联系。微服务就像一个团队一样工作，它们之间必然存在交流，因此它们也必须能相互定位。因此在启动它们之前，必须有人将它们作为一个完整的系统一同配置好。同时故障处理也成为运维团队的肩上的重担，他们必须第一时间发现问题，并迁移服务。

微服务架构还会使得调试和追踪调用链变得困难，因为一次请求会跨越多个机器上的进程。值得庆幸的是，这些问题现在被像Zipkin的分布式追踪系统所重视。

由于微服务不仅会被独立部署，实际上还会交付给不同的团队进行独立开发。由于失去了来自其他组件的约束，服务的开发者会随时随地引入自己的依赖。而实际上生产环境上一台服务器上会执行多个服务，这自然要求这些服务拥有不冲突的依赖。但这对于运维团队来说，又是一次挑战。

## 持续集成带来的改变

过去，开发人员负责开发应用，之后将应用提供给运维人员，运维人员需要负责部署这些应用。但是现在，个个组织开始发现，由相同的团队来开发、部署并关注应用的整个生命周期是更好的解决方案。这意味着开发人员、QA和运维团队现在需要在整个流程中通力合作，这种方案称为DevOps。

但是事实上，开发人员和运维人员拥有着不同的关注点。开发人员关注的是创建新的特性以及提升用户体验，但是却对数据中心的硬件组织架构不感兴趣。而同样的运维人员关注数据中心的硬件组织架构、系统的安全性，而不希望处理应用的依赖问题以及组件之间的潜在依赖关系。

理想情况下，你希望开发人员在不知道硬件架构的情况下，无需运维团队的协助，自己部署应用。这中方案被称为NoOps。当然，你依旧需要人员管理硬件架构，但是理想情况下，他们不需要处理应用的各种依赖。

如你所见，Kubernetes帮助我们实现了上面的一切。通过抽象实际硬件并以单独的平台暴露出来，提供部署和运行应用的能力。它允许开发人员在没有系统管理员的帮助下独立配置和部署他们的应用，并允许系统管理员关注硬件系统的状态，而不需要了解硬件上面实际运行了哪些应用。

## 了解容器技术

当一个大的应用仅由少量组件组成时，我们完全可以为每个组件启动一个虚拟机，这样就能完全满足每个组件的环境需求。

但是随着微服务化，越来越多的小服务出现，如果我们选择使用更多的虚拟机启动服务，那么将会耗费大量的计算资源，并且启动和配置虚拟机也需要许多的人力成本。

人们开始转投Linux的容器技术。容器拥有虚拟机的隔离能力，且开销更小。运行在容器中的进程，就像一般的Linux进程一样被驱动（虚拟机中的进程时运行在分离的操作系统中），但是对于进程本身来说，它会认为在自己独占了整个内核和机器。

相较于虚拟机，容器要更加轻量，也因此使用容器允许你在相同的机器上运行更多的组件。虚拟机除了运行你的服务进程外，还会运行一系列的系统进程，这也导致了组件进程外额外的开销。而一个容器，仅仅是运行在宿主机中的一个隔离进程，仅消费应用需要的资源而已，没有额外的开销。

由于虚拟机带来的开销，你很难为一个组件分配一个完整虚拟机，但是使用容器，你可以且应该为一个组件分配一个独立的容器。

![](https://raw.githubusercontent.com/taodaling/assets/master/images/2019-01-30-kubernetes/vm%20and%20container.PNG)

到了这个点，你可能会好奇容器如何能做到进程隔离，如果他们运行在相同的操作系统上。两个机制使之变得可能。第一个，Linux命名空间保证了每个进程仅能看到它们视野中的系统（文件、进程、网络接口、主机名等）。第二个，Linux控制组，限制每个进程可以使用的资源量（CPU、内存、网络带宽）。

容器化技术也有一些局限。由于容器使用的是宿主机的Linux内核，如果组件要求一个特定版本的容器内核，那么容器将无法在所有主机上运行。同样你不能将使用x86架构的应用容器化后在ARM机器上运行，容器是不能跨硬件架构的。

## 为什么需要Kubernetes

原来我们以手动的方式配置启动服务。这些工作交由运维团队，部署好服务后，运维团队还需要监控这些服务，一旦其中一些服务出现故障，他们还需要将服务迁移到其它健康的机器上。

如今，庞大的服务在逐渐被切分为更小的，独立运行的组件，这些组件被称为微服务。由于微服务彼此解耦，因此他们可以被独立的开发，部署，更新。这能帮助你跟上快速变化的业务需求的脚步。

但是越来越多的服务，已经很难再如往日一般手动配置，部署和管理了。我们很难得知如何在机器和服务之间进行关联可以获得更好的资源利用率，因此也无从削减硬件开销。我们需要自动化，需要自动化地在服务和机器之间进行关联，自动化的配置，监控和故障处理。

## Kubernates提供了什么

Kubernetes抽象了基础硬件，并将整个数据中心抽象为一个单独的巨大的计算资源。这样不论Kubernetes管理了多少节点，用户都是通过相同的方式部署和运行应用。更多的节点对于用户来说仅仅是更多的计算资源。

Kubernetes非常适用于已存的数据中心，但是让其真正发挥作用的还要数云服务提供商。Kubernetes允许他们向开发者提供一个简单的平台，用于部署和允许任意类型的应用，而不需要提供商提供自己的管理功能，来知晓运行在他们硬件上数以万记的服务的任何信息。

## Kubernetes做了什么

系统由一个主节点和若干工作节点组成，提供者向Kubernetes提交应用列表，Kubernetes会将这些应用部署到工作节点上，而具体哪个应用部署到哪台机器上对于开发者和系统管理员都是不重要的（也不应该重要）。

![](https://raw.githubusercontent.com/taodaling/assets/master/images/2019-01-30-kubernetes/submit%20app%20to%20kubernetes.PNG)

开发者也可以指定哪些应用必须在一起运行，这样Kubernetes就会将它们部署到同一个工作节点上，而其它的应用则会落在集群的各处。但是它们依旧可以相互交流，无论它们部署在哪里。

可以认为Kubernetes为我们做了服务发现、副本扩展、负载均衡、健康监控、主从选举的通用工作。而开发者只需要开发应用的实际特性。

Kubernetes会运行你的应用，并向它的组件提供如何找到其余组件的路由信息，并保证它们都处于运行状态。由于应用不关心在哪个节点上运行，Kubernetes可以在任何时候迁移你的应用，从而实现比手动调度更好的资源利用率。

## 理解Kubernetes架构

硬件层面上，Kubernetes集群是由许多节点组成的，节点可以分成两类：

- 主节点，上面运行了Kubernetes的舵面，用来控制和管理整个Kubernetes系统。
- 工作节点，运行你部署的应用。

![](https://raw.githubusercontent.com/taodaling/assets/master/images/2019-01-30-kubernetes/basic%20architecture.PNG)

舵面是控制集群的单元。它有多个组件组成，这些组件可以在一个主节点运行，或者可以划分到多个节点上实现多副本来确保高可用性。这些组件有：

- API Server，你和其他舵面组件的对象。
- Scheduler，分配组件到工作节点
- Controller Manager，负责集群级功能，比如为组件创建副本，追踪工作节点，处理节点失败，以及其他内容。
- etcd，一个可靠的分布式数据存储，负责持久化集群配置。

舵面的组件持有并控制集群的状态，但是它们不会运行你的应用，这些由工作节点负责。

工作节点是负责运行你的容器化应用的机器，它们负责运行、监控和为你的应用提供服务，它们主要包含下面的组件：

- Docker，rkt或则其他实时容器服务，用于运行你的容器。
- Kubelet，与API Server交流并管理它上面的容器。
- kube-proxy，负责应用组件的网络负载均衡。

## 了解Kubernetes工作流程

首先你要将你的组件打包到镜像中，并推送到镜像注册中心，之后向API server发送一份描述。

描述中包含如下信息。

- 包含组件的镜像信息
- 哪些组件要部署在相同节点上，哪些不需要
- 每个组件的副本数目
- 组件对内或对外通过IP地址暴露服务，并对其他服务是可以发现的

API server处理你的描述信息时，Scheduler将指定的容器分组，基于每个分组所需的计算资源以及工作节点当时的可用资源，调度到可用的工作节点上。之后工作节点上的Kubelet命令Docker拉去所需的镜像并启动容器。

一旦应用被运行后，Kubernetes会持续地确保应用的状态匹配你提供的描述。

在应用启动后，你还可以实时增加或减少应用的副本数目，而Kubernates会对应地部署额外的副本或停止超出的部分。你甚至可以让Kubernetes替你做决定，Kubernetes会根据统计数据自动改变副本数。

# 上手

## 在Docker上运行app

首先你需要按照docker官网的指示安装docker应用。

之后我们创建第一个Node.js应用，它会启动一个服务器，并监听8080端口，向客户端返回自己所在主机的hostname。

创建一个目录app并切换工作目录为app，并在其中创建app.js文件。

```js
const http = require('http');
const os = require('os');
console.log("Kubia server starting...");
var handler = function(request, response) {
 console.log("Received request from " + request.connection.remoteAddress);
 response.writeHead(200);
 response.end("You've hit " + os.hostname() + "\n");
};
var www = http.createServer(handler);
www.listen(8080);
```

之后在app目录下创建Dockerfile文件。

```dockerfile
FROM node:7
ADD app.js /app.js
WORKDIR /
EXPOSE 8080
ENTRYPOINT ["node", "app.js"]
CMD []
```

之后我们以app作为上下文构建Docker镜像。

```sh
$ docker build -t app .
```

之后启动我们的服务器镜像。

```sh
$ docker run -d -p 8080:8080 app
```

之后访问我们的服务器。

```sh
$ curl -i -G localhost:8080

HTTP/1.1 200 OK
Date: Fri, 01 Feb 2019 02:43:39 GMT
Connection: keep-alive
Transfer-Encoding: chunked

You've hit 5a1111ad2a1e
```

## 使用Kubernetes运行app

由于app现在仅本地能访问，我们需要先推送到Docker Hub上。首为app镜像创建别名。

```sh
$ docker tag app taodaling/app
```

之后推送镜像。

```sh
$ docker push taodaling/app
```

之后我们需要安装Kubernetes。由于使用Kubernetes配置集群比较复杂，因此选用可以快捷支持单机模式的minikube，你可以在它的github主页上找到安装说明[https://github.com/kubernetes/kops](https://github.com/kubernetes/kops)。

之后启动minikube。

```sh
$ minikube start
Starting local Kubernetes v1.13.2 cluster...
Starting VM...
Downloading Minikube ISO
 181.48 MB / 181.48 MB [============================================] 100.00% 0s
Getting VM IP address...
Moving files into cluster...
Downloading kubeadm v1.13.2
Downloading kubelet v1.13.2
Finished Downloading kubeadm v1.13.2
Finished Downloading kubelet v1.13.2
Setting up certs...
Connecting to cluster...
Setting up kubeconfig...
Stopping extra container runtimes...
Starting cluster components...
Verifying kubelet health ...
Verifying apiserver health ...
Kubectl is now configured to use the cluster.
Loading cached images from config file.


Everything looks great. Please enjoy minikube!
```

在国内由于google被墙了，所以可能需要指定梯子的信息。

```sh
$ minikube start --docker-env HTTP_PROXY=<your proxy> --docker-env HTTPS_PROXY=<your proxy> --registry-mirror=https://registry.docker-cn.com --iso-url https://storage.googleapis.com/minikube/iso/minikube-v0.33.1.iso
```

有了服务器后，我们还需要客户端工具与kubernetes进行交流。在这里你可以找到kubectl的安装说明[https://kubernetes.io/docs/tasks/tools/install-kubectl/](https://kubernetes.io/docs/tasks/tools/install-kubectl/)。

查看k8s集群状态。

```sh
$ kubectl cluster-info
Kubernetes master is running at https://192.168.99.101:8443
KubeDNS is running at https://192.168.99.101:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

打开k8s的控制面板。

```sh
$ minikube dashboard
Enabling dashboard ...
Verifying dashboard health ...
Launching proxy ...
Verifying proxy health ...
Opening http://127.0.0.1:10990/api/v1/namespaces/kube-system/services/http:kubernetes-dashboard:/proxy/ in your default browser...
```

之后我们利用k8s启动容器。

```sh
$ kubectl run kubia --image=taodaling/kubia --port=8080 --generator=run/v1
pod/kubia created
```

上面命令首先发送REST风格HTTP请求到Kubernetes的API服务器，再集群中创建一个新的ReplicationController对象，之后新建的ReplicationController对象创建了一个新的豆荚，这个豆荚之后被Scheduler调度到某个工作节点上，工作节点接到调度命令后，利用docker基于镜像启动容器。

这样你实际上就基于镜像taodaling/kubia启动了自己的容器。

你可能希望列举出所有运行的容器，比如`kubectl get containers`，但是k8s并不会直接暴露单个容器，它使用了多个同地址容器的概念，这样一组容器称为豆荚（pod）。豆荚是一组紧密关联的容器，它们总是会在同一个工作节点运行，并处于相同的Linux命名空间下。每一个pod都像一个分离的逻辑机器一样，拥有自己的IP，域名，进程等。只有在同一个pod下才会有这样的待遇，属于不同pod的两个容器，即使运行在同一个工作节点上，也会像运行在不同的机器上一样无法直接交互。

![](https://raw.githubusercontent.com/taodaling/assets/master/images/2019-01-30-kubernetes/pod.PNG)

你可以拉取pod列表。

```sh
$ kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
kubia-2lhvt   1/1     Running   1          21h
```

类似的，你可以用`kubectl describe pod`来查看一个pod的详细信息。

每个pod都有自己的IP地址，但是这地址是集群内部分配的地址，无法从外部直接通过该地址访问。你需要通过一个服务对象才能在集群外部访问pod。首先你需要创建一个特殊的LoadBalancer类型服务，并通过负载均衡服务连接到集群内部的pod中。

```sh
$ kubectl expose rc kubia --type=LoadBalancer --name kubia-http
service/kubia-http exposed
```

这里rc是ReplicationController的缩写。

可以通过`kubectl get services`查看我们创建的服务对象。

```sh
$ kubectl get services
NAME         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP      10.96.0.1        <none>        443/TCP          30h
kubia-http   LoadBalancer   10.109.153.192   <pending>     8080:32111/TCP   2m2s
```

由于我们使用的是minikube，所以是无法得到EXTERNAL-IP的，我们需要通过下面的命令获得IP和端口。

```sh
$ minikube service kubia-http
```

重新整理一下流程。我们首先利用kubectl run命令创建了一个ReplicationController，而ReplicationController则为我们创建了实际的Pod对象。要让Pod变得可以从外部访问，你需要告诉Kubernetes将这个ReplicationController管理的所有pods作为单独的服务暴露。

- Pod：pod是一组紧密关联的容器，它们运行在相同的工作节点上，拥有独立的IP地址和域名。
- ReplicationController：ReplicationController负责维持pod的副本数目达到预期值。
- Service：由于Pod死亡重启后会得到不同的ip地址，所以通过拥有不变IP地址的Service暴露服务，Service会将连接交付给某个提供服务的Pod。

下面我们来观察我们已有的ReplicationController

```sh
$ kubectl get replicationcontrollers
NAME    DESIRED   CURRENT   READY   AGE
kubia   1         1         1       28h
```

上面desired属性字段表示副本的期望数目。如果在使用kubelet run命令时没有指定则取默认值1。

你可以在ReplicationController创建后调整副本期望数。

```sh
$ kubectl scale rc kubia --replicas=3
replicationcontroller/kubia scaled
```

如果你非常在意每个pod运行在哪个工作节点上的话，可以使用下面命令。

```sh
$ kubectl get pods -o wide
NAME          READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
kubia-2lhvt   1/1     Running   1          28h   172.17.0.4   minikube   <none>           <none>
kubia-5psld   1/1     Running   0          16m   172.17.0.6   minikube   <none>           <none>
kubia-cmhd6   1/1     Running   0          16m   172.17.0.7   minikube   <none>           <none>
```

或者

```sh
$ kubectl describe pod kubia-2lhvt
Name:               kubia-2lhvt
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               minikube/10.0.2.15
Start Time:         Mon, 11 Feb 2019 12:48:32 +0800
Labels:             run=kubia
Annotations:        <none>
Status:             Running
IP:                 172.17.0.4
Controlled By:      ReplicationController/kubia
```

## 理解Pod

Pod是Kubernetes的基石，就像容器之于docker。你可能会好奇为什么Kubernetes要使用pod这个概念而非直接使用容器。我们可以将一组容器中运行的进程集中在一个容器中运行，但是这与容器的哲学相悖，每个容器中都应该仅运行一个用户进程，否则你就必须自己实现进程的重启机制，而每个容器仅运行一个用户进程，就可以将容器的存活交由Kubernetes负责。

存在于同一个Pod中的容器，它们并不像大多数容器一样完全隔离，它们会选择共享网络、UTS、IPC等命名空间。但是由于容器本身的设计使得容器之间无法共享文件系统，但是可以使用卷的概念来共享一部分的目录。也由于Pod中的容器共享了网络命名空间，所以容器之间可能会存在端口冲突。同样Pod中的容器共享相同的网络接口回路（loopback network interface），因此你可以在Pod中通过localhost与其它容器交流。

你应该将pod视作一个单独的机器，但是每个机器上都仅运行一个应用。每个pod仅包含紧密关联的组件。

你不应该将许多不必运行在相同机器上的组件放在一个pod中，比如前端服务器和后端服务器，原因很简单，同一个pod中意味着pod中的组件无法部署在不同的机器上，这样就无法充分利用计算能力，并且一个pod中包含大量的组件会导致很难找到能提供充足计算能力的工作节点。并且pod是kubernetes中的最小伸缩单位，这意味着pod中的不同组件会拥有相同的副本数。但是一般前端服务器和后端服务器的吞吐能力往往是不同的，因此，不应该拥有相同的副本数。

## 通过配置文件创建POD

Kubernetes支持使用配置文件来创建pod，使用配置文件的好处在于可以使用所有的属性和kubernetes的特性。

之前我们已经启动了一些pod了，现在让我们看看它们对应的yaml格式的配置文件是怎样和的。

```sh
$ kubectl get pod kubia-f9fgt -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2019-02-12T11:03:28Z"
  generateName: kubia-
  labels:
    run: kubia
  name: kubia-f9fgt
  namespace: default
  ownerReferences:
  - apiVersion: v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicationController
    name: kubia
    uid: d3827062-2eb5-11e9-9154-080027a39588
  resourceVersion: "6216"
  selfLink: /api/v1/namespaces/default/pods/kubia-f9fgt
  uid: d38383aa-2eb5-11e9-9154-080027a39588
spec:
  containers:
  - image: taodaling/kubia
    imagePullPolicy: Always
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-b22sj
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: minikube
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: default-token-b22sj
    secret:
      defaultMode: 420
      secretName: default-token-b22sj
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2019-02-12T11:03:28Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2019-02-12T11:03:34Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2019-02-12T11:03:34Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2019-02-12T11:03:28Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://3ae7c740a654cec94b2862af7a67fc036484f674464d8ff8f0b8549e4ed48e03
    image: taodaling/kubia:latest
    imageID: docker-pullable://taodaling/kubia@sha256:6442839842ddc7998c90becf14e0d456ae3d6e892363a52fb9c45386dc95aa01
    lastState: {}
    name: kubia
    ready: true
    restartCount: 0
    state:
      running:
        startedAt: "2019-02-12T11:03:33Z"
  hostIP: 10.0.2.15
  phase: Running
  podIP: 172.17.0.4
  qosClass: BestEffort
  startTime: "2019-02-12T11:03:28Z"
```

别被上面冗长的配置文件所吓倒，仅保留必要部分后还是非常简短的。我们先创建一个新文件kubia-manual.yml。

```yaml
apiVersion: v1 #描述符满足V1版本的Kubernetes API
kind: Pod #配置文件用于描述一个Pod
metadata:
  name: kubia-manual #这个Pod的名字
spec:
  containers:
  - image: taodaling/kubia #用来创建容器的镜像
    name: kubia #容器的名字
    ports:
    - containerPort: 8080 #应用监听的端口
      protocol: TCP #端口协议
```

是不是简单多了。

在配置文件中所写的端口信息仅仅是起提示作用而已，忽略它们不会带来任何影响。只要一个容器监听0.0.0.0地址的某个端口，其它的pod就能连接上它的，即使这个端口没有出现在配置文件中。但是在配置文件中显式声明端口并非毫无意义，它可以让使用者容易地找到需要容器提供服务的端口，并且显示指定端口信息允许你为端口分配一个别名。

要查看一个属性的含义，以及包含的子属性，可以使用下面命令。

```sh
$ kubectl explain pods
KIND:     Pod
VERSION:  v1

DESCRIPTION:
     Pod is a collection of containers that can run on a host. This resource is
     created by clients and scheduled onto hosts.

FIELDS:
   apiVersion   <string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#resources

   kind <string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#types-kinds

......
```

要查看子属性

```sh
$ kubectl explain pods.spec
KIND:     Pod
VERSION:  v1

RESOURCE: spec <Object>

DESCRIPTION:
     Specification of the desired behavior of the pod. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status

     PodSpec is a description of a pod.

FIELDS:
   activeDeadlineSeconds        <integer>
     Optional duration in seconds the pod may be active on the node relative to
     StartTime before the system will actively try to mark it failed and kill
     associated containers. Value must be a positive integer.
......
```

接下来从配置文件中启动pod

```sh
$ kubectl create -f kubia-manual.yml
pod/kubia-manual created
```

`kubectl create -f`命令用于创建创建配置文件中指定的所有资源（不仅仅是pods）。

## 查看日志

容器化应用一般会将日志输出到标准输出流（stdout）和标准错误流（stderr）而非日志文件，也因此允许用户直接查看日志。

在kubernetes中你可以通过ssh直接登录允许容器的工作节点，并用docker logs命令查看容器输出。

```sh
$ docker logs <container id>
```

但是kubernetes提供了一种更加简单的方式来直接查看一个pod的日志。

```sh
$ kubectl logs kubia-manual
Kubia server starting...
```

容器日志会每天在日志达到10M时自动滚动，而kubectl logs命令仅显示最后一次滚动后保留的日志。

如果pod中包含多个容器时，那你在获取日志时需要通过`-c <容器名>`显式指定从哪个容器中获取日志。

```sh
$ kubectl logs kubia-manual -c kubia
```

## 端口转发

kubernetes支持端口转发，允许在本地机器和pod之间建立转发关系。

```sh
$ kubectl port-forward kubia-manual 8888:8080
Forwarding from 127.0.0.1:8888 -> 8080
Forwarding from [::1]:8888 -> 8080
```

之后当你访问localhost:8888就可以访问kubia-manual提供的服务了。

端口转发可以帮助你直接测试你的服务。

## 通过标签组织pods

现在你的集群中仅运行了两个pods，但是当pods越来越多时，就越来越需要对pods进行细分。我们需要一种基于某种规则进行分组的方式，分组后，我们可以更清晰地得知每个pod的用途。有了分组后，我们也可以对同组所有pods进行某种操作，而非一一执行。

标签是一种简单但是非常强力的一种手段，它不仅可以对pods进行分组，还可以对kubernetes中的所有资源进行分组。一个标签是加在资源上的任意键值对。之后就可以通过标签集合对资源进行过滤和筛选。一个资源可以有多个不同标签，这些标签的关键字互异。一般你在创建资源的时候加上标签，但是也可以在资源创建后再加标签。

每个pod都有两个标签。

- app，指定pod属于哪个组件、微服务、应用。
- rel，指定pod的发现版本，比如stable、beta、canary。

接下来我们修改配置文件来增加额外的标签。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual-v2
  labels:
    creation_method: manual
    env: prod
spec:
  containers:
  - image: taodaling/kubia
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
```

之后可以查看标签信息。

```sh
$ kubectl get pods --show-labels
NAME              READY   STATUS    RESTARTS   AGE     LABELS
kubia-f9fgt       1/1     Running   0          3h23m   run=kubia
kubia-manual      1/1     Running   0          27s     <none>
kubia-manual-v2   1/1     Running   0          2m17s   creation_method=manual,env=prod
```

除了列出所有标签，你也可以仅查看指定的标签。

```sh
$ kubectl get pods -L creation_method,env
NAME              READY   STATUS    RESTARTS   AGE     CREATION_METHOD   ENV
kubia-f9fgt       1/1     Running   0          3h27m
kubia-manual      1/1     Running   0          4m22s
kubia-manual-v2   1/1     Running   0          6m12s   manual            prod
```

你也可以直接追加标签。

```sh
$ kubectl label pods kubia-manual creation_method=manual
pod/kubia-manual labeled
```

要修改已有标签的值，需要加上`--overwrite`选项。

```sh
$ kubectl label pods kubia-manual creation_method=manual-v2 --overwrite
```

## 刻画应用需求

虽然再k8s中，我们要尽量不关心应用和工作节点之间关联关系从而避免偶尔，并由k8s为我们做出最优的决策，但是还是存在一些特殊情况。比如你希望你的应用能使用SSD，或者你的应用可以使用GPU加速，这些情况都对工作节点提出了需求。

前面已经提过，我们可以为k8s中所有的资源添加标签，我们可以利用标签来描述一个节点。

```sh
$ kubectl label node minikube gpu=true
node/minikube labeled
```

之后查看节点标签。

```sh
$ kubectl get nodes -L gpu
NAME       STATUS   ROLES    AGE    VERSION   GPU
minikube   Ready    master   2d1h   v1.13.2   true
```

利用标签选择器可以仅查看所有拥有指定标签的节点。

```sh
$ kubectl get nodes -l gpu=true
NAME       STATUS   ROLES    AGE    VERSION
minikube   Ready    master   2d1h   v1.13.2
```

为节点增加标签刻画了节点特性后，我们接下来刻画应用的需求。创建kubia-gpu.yml文件。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  nodeSelector:
    gpu: "true"
  containers:
  - image: taodaling/kubia
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
```

由于每个节点都有自己唯一的`kubernetes.io/hostname`标签，因此你可以利用这个标签将应用调度到一个工作节点上去，但是这样也就放弃了kubernetes的优势。

## 使用注解

k8s除了提供标签外，还提供了注解。注解与标签类似，也是以键值对的形式存在，但是区别在于，注解没有选择器，但是注解支持更长的长度（256KB）。注解一般用于内部字段的演化，一开始官方会以注解形式提供新的字段，如果这个字段被广泛接受，则官方会加入字段，并deprecate原来的注解。

增加注解非常简单。

```sh
$ kubectl annotate pod kubia-gpu mycompany.com/someannotation="foo bar"
pod/kubia-gpu annotated
```

要查看pod的annotations，有两种方法。

```sh
$ kubectl get pod kubia-gpu -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    mycompany.com/someannotation: foo bar
  creationTimestamp: "2019-02-13T03:36:21Z"
  name: kubia-gpu
  namespace: default
  resourceVersion: "77813"
  selfLink: /api/v1/namespaces/default/pods/kubia-gpu
  uid: 87a2599f-2f40-11e9-b0c4-0800279f3491
......
```

还有一种方法是。

```sh
$ kubectl describe pod kubia-gpu
Name:               kubia-gpu
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               minikube/10.0.2.15
Start Time:         Wed, 13 Feb 2019 11:36:21 +0800
Labels:             <none>
Annotations:        mycompany.com/someannotation: foo bar
......
```

## 使用命名空间

K8s提供了命名空间，用于对资源进行分组。不同命名空间的资源允许重名。你可以按照租户划分命名空间，也可以根据生产、开发、QA环境来划分命名空间，这些都由你决定。资源名字只需要保证再命名空间内唯一即可，不同命名空间下可以存在重名资源。虽然大部分资源都可以存放于命名空间下，但是还是有少数不支持命名空间，比如节点，节点始终是全局的，不属于任何命名空间。

查看已有的命名空间。

```sh
$ kubectl get namespaces
NAME          STATUS   AGE
default       Active   2d3h
kube-public   Active   2d3h
kube-system   Active   2d3h
```

至今为止，你只操作过default命名空间下的资源。除了default外，列表中还列出了另外两个命名空间。下面我们来看看kube-system下的pod。

```sh
$ kubectl get pods --namespace kube-system
NAME                                   READY   STATUS    RESTARTS   AGE
coredns-86c58d9df4-8kv5b               1/1     Running   2          2d3h
coredns-86c58d9df4-wn7vl               1/1     Running   2          2d3h
etcd-minikube                          1/1     Running   0          3h48m
kube-addon-manager-minikube            1/1     Running   2          2d3h
kube-apiserver-minikube                1/1     Running   0          3h48m
kube-controller-manager-minikube       1/1     Running   2          2d3h
kube-proxy-m2v2b                       1/1     Running   0          3h47m
kube-scheduler-minikube                1/1     Running   2          2d3h
kubernetes-dashboard-ccc79bfc9-bc7qm   1/1     Running   4          2d2h
storage-provisioner                    1/1     Running   4          2d3h
```

上面的pod都是属于k8s本身。k8s通过命名空间将系统的pod与用户的pod分离开来，这样不仅可以保持default命名空间的简洁，还能避免用户误删系统pod。

命名空间还能在多个用户操作k8s时避免命名冲突，每个用户仅在自己的命名空间下执行操作。并且命名空间可以控制用户的访问权限，甚至于可以为每个用户分配各自的计算资源。

接下来我们通过配置文件创建命名空间，首先建立文件custom-namespace.yml。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: custom-namespace
```

之后创建命名空间。

```sh
$ kubectl create -f custom-namespace.yml
namespace/custom-namespace created
```

当然要为每个命名空间都建立一个文件显得很繁琐，我们可以直接使用命令行就可以创建命名空间。

```sh
$ kubectl create namespace custom-namespace2
namespace/custom-namespace2 created
```

至于在命名空间下创建pod，有两种方式。一种是直接在配置文件的metadata.namespace中写死，还有一种就是增加`-n <命名空间>`选项。

```sh
$ kubectl create -f kubia-gpu.yml -n custom-namespace
pod/kubia-gpu created
```

如果你创建pod时没有显式指定命名空间，则会使用default命名空间。

注意命名空间并没有提供任何实际上的隔离，比如两个不同命名空间下的pod可以通过彼此的内部IP地址自由交流而不受命名空间的影响。

## 停止并移除pods

现在我们已经在default命名空间和custom-namespace下创建了不少的pods，但是它们已经不再被需要了，我们接下来要移除它们。

```sh
$ kubectl delete pod kubia-gpu
pod "kubia-gpu" deleted
```

要删除一个pod，k8s首先终止属于该pod的所有容器。k8s会向这些容器进程发送一个SIGTERM信号，并等待30秒时间，如果超时还没有关闭，那么会发送SIGKILL信号。

除了手动指定名字删除pod外，你还可以使用标签选择器来选择被删除的pod。

```sh
$ kubectl delete pods -l creation_method=manual
pod "kubia-manual" deleted
pod "kubia-manual-v2" deleted
```

你也可以选择直接删除命名空间，删除命名空间的同时会删除命名空间下所有的pod。

```sh
$ kubectl delete namespace custom-namespace
namespace "custom-namespace" deleted
```

现在你应该就仅剩下通过kubectl run启动的pod了。

```sh
$ kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
kubia-5psld   1/1     Running   1          22h
```

你可以通过传递--all而非pod名来删除当前命名空间下所有的pods。

```sh
$ kubectl delete pods --all
pod "kubia-5psld" deleted
```

接下来再看看pod的状态。

```sh
$ kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
kubia-87w58   1/1     Running   0          97s
```

之前的kubia-5psld被删除了，但是又一个新的pod被启动了。还记得一开始我们通过kubectl run创建pod时，并不是直接创建pod，而是通过创建一个副本控制器之后由副本控制器负责创建pod。要删除这个pod，我们还需要删除管理它的副本控制器。

```sh
$ kubectl delete all --all
pod "kubia-87w58" deleted
replicationcontroller "kubia" deleted
service "kubernetes" deleted
service "kubia-http" deleted
```

上面命令中第一个all表示删除所有类型的资源，第二个all表示删除全部而不指定名字。

## Kubelet与存活探测

至今为止，我们学会了如何手动创建、监控、管理pod，但是再真实世界中，你可能只是希望你的pod能保持运行，并自动保证健康的副本数，而无需任何人工介入。要实现这个目标，你不能再直接创建pods，取而代之的是你要创建pod的管理者，比如副本控制器或Deployments，而它们将负责pod的生命周期。

当你创建了不被管理的pod，这个pod会分配到某个工作节点上。kubernetes会监控这些容器并在容器失败后自动重启它们，但是一旦整个节点损害，那么节点上运行的所有不受管理的pod都将会丢失并不再被重启。

在pod调度到一个节点上后，该节点上的kubelet服务会启动pod，并保证在pod存在的期间pod中的容器始终存活。一旦容器的主进程挂了，kubelet会重启该容器。

但是上面的方案还是不保险，因为存在容器不能提供服务但是却依旧存活的情况，比如抛出内存溢出错误的JVM会依旧存活，但是不能再保证功能。这提示我们需要应用在自己失去能力的时候杀死自己。但是如果错误是程序逻辑，比如死循环或死锁，我们无法正确的自我制裁，而k8s也不能察觉到问题发生。

k8s提供了存活探测的方式来检查一个容器是否能正常工作。k8s会定期执行探测并在探测失败的情况下重启容器。k8s可以以下面的一种机制来探测一个容器：

- HTTP GET探测：向容器的地址以及你指定好的端口和路径发送HTTP GET请求，如果收到响应并且响应码不意味着错误（2xx或3xx），那就认为容器正常工作，否则认为容器已经失败。
- TCP套接字探测：尝试向容器的某个端口发起TCP连接。如果连接成功建立就认为容器正常工作，否则认为容器失败。
- Exec探测：在容器中执行任意命令并校验命令的返回值，如果返回值为0，则容器正常工作，否则认为容器失败。

由于我们的kubia太过简单，因此有生之年可能都很难看到它出什么问题。我们修改代码，使得它只能正常服务5次，5次后就只会返回500内部服务器异常。 先创建文件kubia-liveness.yml。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-liveness
spec:
  containers:
  - image: luksa/kubia-unhealthy
    name: kubia
    livenessProbe:
      httpGet:
        path: /
        port: 8080
```

之后创建应用并不断访问应用即可。

如果想要知道一个容器是因为什么原因宕机，用`kubectl logs`命令只能拉去到重启后新容器的日志，而无法看到之前宕机容器的日志。你可以使用`--previous`选项查看该容器重启之前的所有日志。

利用describe命令可以查看到许多探测相关的信息。

```sh
$ kubectl describe pods kubia-liveness
......
Containers:
  kubia:
......
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    137
      Started:      Wed, 13 Feb 2019 17:32:42 +0800
      Finished:     Wed, 13 Feb 2019 17:34:29 +0800
    Ready:          False
    Restart Count:  9
    Liveness:       http-get http://:8080/ delay=0s timeout=1s period=10s #success=1 #failure=3
......
Events:
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  29m                   default-scheduler  Successfully assigned default/kubia-liveness to minikube
  Normal   Created    22m (x3 over 26m)     kubelet, minikube  Created container
  Normal   Started    22m (x3 over 26m)     kubelet, minikube  Started container
  Normal   Pulling    20m (x4 over 29m)     kubelet, minikube  pulling image "luksa/kubia-unhealthy"
  Warning  Unhealthy  19m (x11 over 25m)    kubelet, minikube  Liveness probe failed: HTTP probe failed with statuscode: 500
  Normal   Pulled     14m (x7 over 26m)     kubelet, minikube  Successfully pulled image "luksa/kubia-unhealthy"
  Warning  BackOff    9m36s (x19 over 16m)  kubelet, minikube  Back-off restarting failed container
  Normal   Killing    4m38s (x9 over 24m)   kubelet, minikube  Killing container with id docker://kubia:Container failed liveness probe.. Container will be killed and recreated.
```

可以看到liveness刻画了探测的属性，Exit Code表示上一次退出发送的信号+128，Events中显示了与pod相关的各种事件。liveness的delay属性比较重要，需要设置得足够大（保证app正常启动），在delay后将开始执行第一次探测。failure=3表示连续失败三次就认为容器不可用。

在生产环境中，你始终应该为容器定义一个存活探测，只有这样k8s才能确认容器的状态并采取相应的措施。之前我们使用的探测地址是`/`，但是一般来说你需要提供一个特殊的URL地址，比如`/health`，而当容器内服务器接受到这样的请求时，应该检查所有容器内关键的模块以确保它们都能正常工作。并且还要确保这个地址不需要额外的认证，否则探测会失败。也要记住仅检查服务器自身内部的模块，而不能被外部依赖所影响，比如服务器不该检查数据库是否能正常访问，因为重启服务器对于修复数据库是没有任何意义的。并且要保证检查要足够轻量，因为k8s默认仅提供一秒的超时时间，并且由于检查会频繁发生，因此可能会带来可观的CPU资源浪费。同样由于k8s仅会在连续重试多次还是失败的情况下才会认为容器需要重启，并且即使你将failure设为1，k8s依旧会使用连续多次探测的方式来确保容器状态，因此不需要在提供的探测接口中使用重试机制。

## 副本控制器

工作节点上的kubelet服务会负责容器的探测和重启，这个过程中我们的k8s控制舵面没有参与，因此一旦节点宕机，舵面必须为该节点上所有的pod创建替代品。但是仅由kubelet管理的pod是不会有人记得的，它们会随着kubelet一同停止。要让pod能在其它节点上复活，你需要为这些pod指派一个管理者，比如副本控制器。

副本控制器是一类k8s资源，它们可以保证你的pod始终处于运行状态，如果一个pod由于任何原因从节点上消失，副本控制器会注意到这一事件，并在某个节点上复活这个pod。

当副本控制器管理的副本数目少于预期，则会创建新的副本，如果副本数目多于预期，则会移除多余的副本。

一个副本管理器由以下三部分组成。

- 一个标签选择器，用于决定副本管理器需要管理的pods。
- 一个副本预期数，指定每个pod应该有的实例数。
- 一个pod模板，用于在副本数不足时创建新的pod副本。

副本管理器的这三个部分可以在任何时候修改，并不需要在创建时指定。修改标签选择器和pod模板不会影响现存的pods，修改标签选择器只会让副本控制器放弃对一部分pods的管理，并且开始管理一部分其它pods，而修改pod模板仅对后面由该副本控制器创建的新的副本生效。

接下来我们创建一个副本控制器的配置文件，命名为kubia-rc.yml。

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    app: kubia
  template:
   metadata:
     labels:
       app: kubia
   spec:
     containers:
     - name: kubia
       image: taodaling/kubia
       ports:
       - containerPort: 8080
```

之后我们用create命令创建副本控制器。k8s会为我们创建一个新的副本控制器，命名为kubia。之后kubia会发现拥有标签app=kubia的pod数目少于3，因此会用模板创建3个pod。

需要注意的是模板中创建的pod必须能被副本管理器的标签选择器所选择，否则就会根据模板创建无数的副本。为了避免这种情况，API服务器会验证你提交的副本控制器的定义文件。

你也可以不为副本控制器提供标签选择器，这时候会自动地利用模板中的标签创建标签选择器。这也是推荐的方式。

之后我们创建副本控制器后，重新检查pods。

```sh
$ kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
kubia-2n5lh   1/1     Running   0          4m23s
kubia-5srg2   1/1     Running   0          4m23s
kubia-fddzx   1/1     Running   0          4m23s
```

当我们删除一个pod的时候，副本控制器会收到pod下线的通知，从而做出响应。如果k8s长时间无法收到pod的心跳，那么k8s会将pod状态标记为unknown，同样副本控制器也会被通知到，从而创建的新的副本。

副本管理器与pod没有直接关联，pod的控制权可以在不同的副本控制器之间流转。修改一个pod的标签可以将pod移除出所在副本控制器的控制，这样的pod就和手动创建的pod拥有一样的表现。

之前提到过了可以在任意时候修改副本控制器的三个部分，我们不会修改一个控制器的标签过滤器，却会经常修改控制器的pod模板。

```sh
$ kubectl edit replicationcontroller kubia
```

上面的命令会打开你的默认编辑器编辑副本控制器对应的yaml定义文件。关闭编辑器后你的修改就会即使上传并生效。

要修改副本控制器的副本预期数，同样非常简单，你可以通过编辑控制器的定义文件，也可以借助scale命令。

```sh
$ kubectl scale rc kubia --replicas=3
```

当你用kubectl delete删除副本控制器时会连带将它管理的所有pod也一同删除。但是考虑到pod与副本控制器并非一个整体，因此k8s提供了仅删除副本控制器而不影响pod的方式。

```sh
$ kubectl delete rc kubia --cascade=false
replicationcontroller "kubia" deleted
```

## 副本集合

最初的时候，副本控制器是k8s中仅有的可以创建副本和重新调度pod的模块，但是后来引入了相似的一种资源，称为副本集合。可以认为副本集合是新一代的副本控制器并且最终将取代副本控制器。

副本集合与副本控制器基本一致，但是副本集合提供了比副本控制器的标签选择器更灵活的pod选择器。标签选择器仅过滤出包含所有指定标签的pod，但是副本集合的选择器允许我们过滤缺少某个标签的pods，或包含某个特定标签关键字的pods。

让我们创建一个副本集合来收养之前因为删除副本控制器而带来的孤儿pods。先建立一个名为kubia-replicaset.yml的文件。

```yaml
apiVersion: apps/v1beta2
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: taodaling/kubia
```

在这里我们使用的apiVersion是apps/v1beta2 ，而非v1。这是因为v1接口不支持副本集合，但是apps/v1beta2 支持副本集合。一般apiVersion的格式为`group/api`，group表示api的分组，但是大部分的接口都处于k8s的核心API组下，而如果组不填，则缺省为核心API组。

用create命令创建副本集合后，拉去pod状态。

```sh
$ kubectl get pods
NAME          READY   STATUS        RESTARTS   AGE
kubia-5fnfc   1/1     Terminating   0          34m
kubia-7qs7d   1/1     Terminating   0          34m
kubia-8zd4h   1/1     Terminating   0          34m
kubia-9x4cw   1/1     Running       0          36m
kubia-b9g2g   1/1     Terminating   0          34m
kubia-dtdbn   1/1     Running       0          36m
kubia-fvzsp   1/1     Running       0          36m
kubia-l9pbr   1/1     Terminating   0          34m
kubia-mnt6s   1/1     Terminating   0          34m
kubia-v2wfp   1/1     Terminating   0          34m
```

可以看到副本集合中额外的7个副本都在被停止。查看我们的副本集合。

```sh
$ kubectl get replicasets
NAME    DESIRED   CURRENT   READY   AGE
kubia   3         3         3       94s
```

可以看到我们的副本集合和副本控制器并没有太大区别，除了选择器一块。副本集合当前使用的matchLabels的表达能力依旧欠缺，下面我们用另外一种方式表达相同的含义。

```yaml
selector:
  matchExpressions:
  - key: app
    operator: In
    values:
    - kubia
```

如上所示，每个表达式一定包含一个key，operator，以及可选的values属性。接下来你会看到总共四种operator。

- In，标签值出现在values中
- NotIn，标签值不出现在values中
- Exists，标签关键字出现
- DoesNotExist，标签关键字不出现

如果在一个matchExpressions中出现多个表达式，那么必须所有表达式全真才认为是真。如果你还指定了matchLabels，那么必须matchExpressions和matchLabel全真才认为是真。

要记住你始终应该使用副本集合而非副本控制器，接下来我们删除这个副本集合以及它麾下的pods。

```sh
$ kubectl delete rs kubia
replicaset.extensions "kubia" deleted
```

## 守护集合

副本控制器和副本集合都是用于在集群中部署特定数目的副本实例。但是我们还会遇到另外一种需求，在每个节点上运行一个pod。比如你的pod是节点的资源监控器或日志收集器。

在不使用k8s的情况下，一般是注入到系统的启动脚本中，并于系统一同启动。而k8s提供了守护集合的概念。守护集合确保每一个工作节点上都运行pod的一份副本。当一个节点从集群移除，守护集合会终止上面自己管理的pod，当一个节点加入集群，守护集合会立即在节点上按照模板创建一个新的副本。

默认情况下，守护集合会在所有节点上运行pod，但是你依旧可以通过指定节点选择器来保证pod仅运行在满足条件的节点上。

一些节点可以标记为不可调度，这样调度器就不会把pod分配给它们。但是由于这是通过调度器实现节点不可调度的，而守护集合是直接跳过调度器，因此守护集合依旧会把pod部署在不可调度的节点上。通常这样是对的，因为守护集合一般运行的都是系统服务。

我们先建立一个ssh-monitor.yml文件。

```yaml
apiVersion: apps/v1beta2
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector:
        dist: ssd
      containers:
      - name: main
        image: luksa/ssd-monitor
```

之后用create命令创建守护集合。

```sh
$ kubectl create -f ssh-monitor.yml
daemonset.apps/ssd-monitor created
```

查看守护集合的状态

```sh
$ kubectl get ds
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
ssd-monitor   0         0         0       0            0           dist=ssd        67s
```

很显然我们没有一个标有dist=ssd的节点。为我们的minikube加上标签后就可以看到守护集合将ssd-monitor部署在了我们的minikube节点上。

```sh
$ kubectl label nodes minikube dist=ssd 
node/minikube labeled
$ kubectl get ds
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
ssd-monitor   1         1         1       1            1           dist=ssd        3m37s
```

当然我们之后发现minikube并没有使用ssd，所以我们修改标签，再查看守护集合的状态。

```sh
$ kubectl label nodes minikube dist=hdd --overwrite
node/minikube labeled
$ kubectl get ds
NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
ssd-monitor   0         0         0       0            0           dist=ssd        6m6s
```

## 一次性任务

你可能会遇到这样一种场景，你希望能运行一个任务，并在任务结束后终止。至今提到的副本控制器、kubelet、副本集合、守护集合都自带自动重启的功用。

k8s提供了Job类型的资源，它们与副本集合类似，会为你自动创建pod。但是如果一个pod以结果码0退出，那么这个pod不会被重启。而如果以任何异常码退出，Job会将pod重启（但是你也可以指定不重启）。

接下来我们创建一个文件exporter.yml：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure # default value is Always， another choice is Never
      containers:
      - name: main
        image: luksa/batch-job
```

之后创建Job。

```sh
$ kubectl create -f exporter.yml
job.batch/batch-job created
```

luksa/batch-job这个镜像会在等待120秒后退出，等待120秒后。

```sh
$ kubectl get jobs
NAME        COMPLETIONS   DURATION   AGE
batch-job   1/1           2m4s       6m9s
$ kubectl get pods
NAME              READY   STATUS      RESTARTS   AGE
batch-job-884sp   0/1     Completed   0          4m43s
```

之所以在Job完成后没有删除pod是为了留给用户机会访问pod的日志。

```sh
$ kubectl logs batch-job-884sp
Fri Feb 15 09:40:18 UTC 2019 Batch job starting
Fri Feb 15 09:42:18 UTC 2019 Finished succesfully
```

一个Job中可以运行多个pod实例。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job-more
spec:
  completions: 5 # 运行共5次
  parallelism: 3 # 最大并行3
  template:
    metadata:
      labels:
        app: batch-job-more
    spec:
      restartPolicy: OnFailure
      containers:
      - name: main
        image: luksa/batch-job
```

创建Job后。

```sh
$ kubectl get job
NAME             COMPLETIONS   DURATION   AGE
batch-job        1/1           2m4s       16m
batch-job-more   0/5           20s        20s
$ kubectl get pods
NAME                   READY   STATUS      RESTARTS   AGE
batch-job-884sp        0/1     Completed   0          16m
batch-job-more-llp4g   1/1     Running     0          24s
batch-job-more-m9s2x   1/1     Running     0          24s
batch-job-more-tnqhp   1/1     Running     0          24s
```

你能在任意时候修改并行数目。

```sh
$ kubectl scale job batch-job-more --replicas 1
kubectl scale job is DEPRECATED and will be removed in a future version.
job.batch/batch-job-more scaled
```

最后我们还要考虑一个问题，如果程序存在bug，导致死锁，那么Job应该等待多久。你可以设置spec.activeDeadlineSeconds属性来定义等待时间。如果pod超时则会被标记为失败。你也可以配置Job最多可以重试几次，设置spec.backoffLimit字段，默认为6。

## 定时任务

定时任务是指在某个特定时间点启动或定期启动的任务。在Linux和Unix操作系统中，这类任务被称为cron任务。k8s对它们也提供了支持。

首先你要创建一个CronJob类型的资源。在匹配时间点，k8s会按照CronJob定义中配置的Job模板创建一个Job对象，而Job对象会负责创建pods。

假设你要运行之前例子中的批量任务每十五分钟一次。先创建一个文件cronjob.yml：

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: batch-job-every-fifteen-minutes
spec:
  schedule: "0,15,30,45 * * * *" #cron格式
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: periodic-batch-job
        spec:
          restartPolicy: OnFailure
          containers:
          - name: main
            image: luksa/batch-job
```

创建CronJob后，查看作业列表。

```sh
$ kubectl get cj
NAME                              SCHEDULE             SUSPEND   ACTIVE   LAST SCHEDULE   AGE
batch-job-every-fifteen-minutes   0,15,30,45 * * * *   False     0        <none>          15s
```

在指定时间，k8s会按照jobTemplate中指定的数据创建一个Job。由于k8s不会精确地时间点启动Job，它会在一个近似的时间点启动Job。如果你不希望Job的启动时间过迟，你可以为Job指定一个死线。

```yaml
apiVersiopn: batch/v1beta1
kind: CronJob
spec:
  schedule: "0,15,30,45 * * * *"
  startingDeadlineSeconds: 15 #必须在时间点15秒内启动Job
```

在上面的例子中，超过死线后，则不会启动Job，并且Job会被标记为失败。一个CronJob可能会重复创建任务或丢失任务，所以Job需要实现幂等来保证重复运行不会带来问题，并且之后执行Job应该完成之前未完成的工作。

## 内部服务

你已经了解过了如何部署pods。虽然每个pod可以忽略外部刺激独立工作，但是现今的应用往往需要对外部请求做出响应。比如微服务场景，pod需要处理来自集群内的其它pods的请求，以及来自集群外部的pods的请求。

Pod需要一种发现其它pods的手段。在k8s世界外部，系统管理员需要在应用的配置文件中配置依赖组件的IP地址。但是在k8s中却不能这么做，原因有三：

- Pods是临时的，随时都可能因为各种原因上线下线
- K8s在pod调度到节点后，pod启动前为pod分配一个ip地址。
- 水平扩展可能会提供更多的相同服务。

为了解决这些问题，k8s提供了另一种类型的资源—服务。服务是用来为一组提供相同服务的pods暴露单一不变入口的资源。在服务存在期间中，每个服务都会有一个不变IP地址和端口。而服务接受的连接会路由到服务后面的一个pod去。这样不同组件之间就通过服务解耦了开来。

一个服务可以被多个pod支持，而与服务建立的连接会在这些pods之间进行负载均衡。我们先创建一个ReplicationController。

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    app: kubia
  template:
   metadata:
     labels:
       app: kubia
   spec:
     containers:
     - name: kubia
       image: taodaling/kubia
       ports:
       - containerPort: 8080
```

之后创建kubia副本控制器。再创建一个文件kubia-svc.yml。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  ports:
  - port: 80 # 服务暴露端口
    targetPort: 8080 # 容器转发端口
  selector:
    app: kubia
```

创建服务后，查看已有服务。

```sh
$ kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1     <none>        443/TCP   34m
kubia        ClusterIP   10.102.82.0   <none>        80/TCP    26m
```

可以看到kubia被分配了集群IP，这个IP只能在集群内部使用。要在集群中的pod中执行命令，你需要使用exec命令。exec命令用于在容器中执行一个命令。

```sh
$ kubectl exec kubia-7p95h -c kubia -- curl -s 10.102.82.0:80
You've hit kubia-rk257
```

如果你多次执行这个命令，你会发现处理请求的服务器会发生变更。这是因为服务会随机选取一个背后的pod进行请求转发，即使请求来自同一个客户端。如果你希望同一个客户端的请求都由一个pod进行处理，你可以设置服务的sessionAffinity为ClientIP，默认值是None。

```yaml
apiVersion: v1
kind: Service
spec:
  sessionAffinity: ClientIP
......
```

如果你的Pod暴露了多个端口，你也可以通过服务进行暴露。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  selector:
    app: kubia
```

上面的例子中我们为端口分配了名字。如果你使用了不常见的端口，那么你可以为端口分配别名，在使用时可以通过端口名称引用。比如：

```yaml
kind: Pod
sepc:
  containers:
  - name: kubia
    ports:
    - name: http
      containerPort: 8080
    - name: https
      containerPort: 8443
```

```yaml
apiVersion: v1
kind: Service
spec:
  ports:
  - name: http
    port: 80
    targetPort: http
  - name: https
    port: 443
    targetPort: https
```

这样做最大的好处是允许你任意变动pod的端口，而不需要修改服务的定义，即pod和服务端口解耦。

那么pod是如何发现服务的呢。当一个pod启动，k8s会为它设置一系列环境变量，其中就包含了当时存在的服务的信息。由于我们是先创建了pod后再创建了服务，因此pod中不包含kubia服务的信息。我们先删除所有的pod，并让副本控制器为我们重新创建。之后我们选择一个重启后的pod并使用exec命令查看环境变量。

```sh
$ kubectl exec kubia-9vqtg -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=kubia-9vqtg
KUBIA_SERVICE_PORT=80
KUBIA_PORT=tcp://10.102.82.0:80
KUBIA_PORT_80_TCP_PROTO=tcp
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBIA_SERVICE_HOST=10.102.82.0
KUBIA_PORT_80_TCP=tcp://10.102.82.0:80
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
KUBIA_PORT_80_TCP_ADDR=10.102.82.0
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBIA_PORT_80_TCP_PORT=80
KUBERNETES_PORT=tcp://10.96.0.1:443
NPM_CONFIG_LOGLEVEL=info
NODE_VERSION=7.10.1
YARN_VERSION=0.24.4
HOME=/root
```

可以看到里面包含了kubia相关的地址和端口。环境变量通常被用于查找服务的地址和端口，这也正是DNS的工作。再kube-system命名空间下存在一个名为kube-dns的pod。所有集群中的pod的信息都会自动配置在其中。所有pod中运行的进程发起的DNS请求都会被k8s自带的DNS服务器所处理。

每个服务都对应内部DNS服务器的一个入口，而所有知道服务名字的客户端也可以使用全限定域名（FQDN）来访问它，而不需要解析环境变量。比如前后端问题中，前端访问后端数据库可以通过地址`backend-database.default.svc.cluster.local`。其中backend-database是服务名称，default是命名空间，svc表示服务，cluster表示集群，local表示本地。但是端口还是需要预先知道或解析环境变量。

由于默认的后缀就是svc.cluster.local，因此可以省略为`backend-database.default`，而如果服务与客户端处于同一命名空间，则可以省略命名空间，即`backend-database`。

为了实验，我们先使用exec命令登陆到pods上。

```sh
$ kubectl exec -i -t kubia-dh2fk -- /bin/bash
root@kubia-dh2fk:/#
```

接着尝试所有的FQDN。

```sh
root@kubia-dh2fk:/# curl http://kubia.default.svc.cluster.local
You've hit kubia-9vqtg
root@kubia-dh2fk:/# curl http://kubia.default.svc
You've hit kubia-9vqtg
root@kubia-dh2fk:/# curl http://kubia.default
You've hit kubia-n7dqz
root@kubia-dh2fk:/# curl http://kubia
You've hit kubia-9vqtg
```

查看/cat/resolv.conf文件，这个文件中配置了你的自定义DNS服务器以及默认域名。

```sh
root@kubia-dh2fk:/# cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

但是如果你希望ping服务时，你会发现无法ping通。

```sh
root@kubia-dh2fk:/# ping kubia
PING kubia.default.svc.cluster.local (10.102.82.0): 56 data bytes
^C--- kubia.default.svc.cluster.local ping statistics ---
170 packets transmitted, 0 packets received, 100% packet loss
```

这是因为服务的集群IP是虚拟IP，只有结合服务端口时才有意义。

如果你希望服务能为你将请求转发到外部IP和端口，而非集群内IP和端口，这样就可以对客户端屏蔽依赖是否处于集群中这个信息。要解决这个问题，我们需要揭露一些关于服务的实现方式。服务并不是与pod直接关联，二者之间存在端点（Endpoints）类的资源。

```sh
$ kubectl describe svc kubia
......
Endpoints:         172.17.0.10:8080,172.17.0.8:8080,172.17.0.9:8080
......
```

一个端点资源是一组IP地址和端口，它们对外暴露服务。就像其它k8s中的资源一样你可以查看它们。

```sh
$ kubectl get endpoints kubia
NAME    ENDPOINTS                                          AGE
kubia   172.17.0.10:8080,172.17.0.8:8080,172.17.0.9:8080   3h13m
```

服务的标签选择器用于为端点构建一系列IP和端口地址，当客户端连接到服务，服务会选择一个IP和端口进行转发。如果你创建服务时没有指定选择器，那么k8s甚至不会为你创建一个端点资源。你可以自行创建端点资源并设置一组IP和端口。

我们先创建一个名为external-service.yml的文件。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
```

接下来我们创建端点资源，external-service-endpoints.yml。

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service
subsets:
  - addresses:
    - ip: 11.11.11.11
    - ip: 22.22.22.22
    ports:
    - port: 80
```

注意这里端点和服务应该同名。之后创建资源，再查看服务详细信息。

```sh
$ kubectl create -f external-service-endpoints.yml
endpoints/external-service created

$ kubectl describe svc external-service
Name:              external-service
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          <none>
Type:              ClusterIP
IP:                10.97.118.223
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         11.11.11.11:80,22.22.22.22:80
Session Affinity:  None
Events:            <none>
```

如果你之后把外部应用迁移到k8s中，你可以向服务添加一个选择器，这样端点就会被自动更新。当然如果你将内部应用移动到外部，你可以移除选择器，这样端点就不会被自动更新。

除了通过手动配置端点来暴露外部服务，你也可以使用FQDN的方式来引用外部服务。要为外部应用创建一个服务为其提供别名，我们可以创建一个资源类型为ExternalName的资源。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: someapi.somecompany.com
  ports:
  - port: 80
```

创建了资源后，你就可以通过external-service.default来在内部引用它了。ExternalService在DNS层次单独实现，在DNS服务器中会为别名服务专门创建一条CNAME类型的DNS记录。因此客户端会跳过别名服务，直接通过DNS服务器拿到外部应用地址并连接。因此，别名服务甚至不会被分配集群IP。

## 外部服务

至今为止我们讨论的都是仅供内部访问的service。要让一个服务可以被外部访问，有三种方法：

- 设置service类型为NodePort。对于NodePort类型的服务，集群中每个节点都会打开一个端口，并将这个端口收到的报文转发给潜在的服务。因此你可以通过集群的任意一个节点对服务进行访问。
- 设置服务类型为LoadBalancer。这是对NodePort类型的一个扩展，这允许服务能够通过一个负载均衡设备进行访问，这个负载均衡组件由k8s运行的云架构所提供。这个负载均衡设备将接收到的请求重定向到节点端口，客户端通过负载均衡设别的IP连接服务。
- 创建一个Ingress资源。

### NodePort

先创建一个NodePort服务，新建一个文件kubia-svc-nodeport.yml。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30123
  selector:
    app: kubia
```

创建服务后，你就可以通过任意节点的IP地址加上30123端口访问服务了。

对于minikube要查看服务地址。

```sh
$ minikube service list
|-------------|----------------------|-----------------------------|
|  NAMESPACE  |         NAME         |             URL             |
|-------------|----------------------|-----------------------------|
| default     | external-service     | No node port                |
| default     | kubernetes           | No node port                |
| default     | kubia                | No node port                |
| default     | kubia-nodeport       | http://192.168.99.107:30123 |
| kube-system | kube-dns             | No node port                |
| kube-system | kubernetes-dashboard | No node port                |
|-------------|----------------------|-----------------------------|
```

### LoadBalancer

我们先创建一个kubia-svc-loadbalancer.yml文件。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
```

然后创建服务。如果不支持LoadBalancer，那效果会和NodePort相同，因为LoadBalancer是NodePort的一个扩展。

### Ingress

每个LoadBalancer都需要自己的负载均衡设备以及一个IP地址。而Ingress只需要一个，即使是为一打服务提供访问。当客户端向Ingress发送一个HTTP请求，HTTP请求中的主机和路径用于决定请求应该转发给哪个服务。Ingress在HTTP层执行操作，并且可以提供像基于Cookie的Session关联这类特性。

![](https://raw.githubusercontent.com/taodaling/assets/master/images/2019-01-30-kubernetes/ingress.PNG)

要让ingress资源工作，需要创建一个Ingress控制器，不同的k8s环境使用不同的控制器实现。

由于我们在minikube上实验，minikube创建启用控制器需要执行下面步骤。

```sh
$ minikube addons list
- addon-manager: enabled
- dashboard: enabled
- default-storageclass: enabled
- efk: disabled
- freshpod: disabled
- gvisor: disabled
- heapster: disabled
- ingress: disabled
- kube-dns: disabled
- metrics-server: disabled
- nvidia-driver-installer: disabled
- nvidia-gpu-device-plugin: disabled
- registry: disabled
- registry-creds: disabled
- storage-provisioner: enabled
- storage-provisioner-gluster: disabled
```

可以看到ingress是disabled的。接下来启动ingress。

```sh
$ minikube addons enable ingress
ingress was successfully enabled
```

之后我们查看kube-system命名空间下的pods。

```sh
$ kubectl get pods --namespace=kube-system
NAME                                       READY   STATUS              RESTARTS   AGE
coredns-86c58d9df4-8kv5b                   1/1     Running             8          7d6h
coredns-86c58d9df4-wn7vl                   1/1     Running             8          7d6h
default-http-backend-5ff9d456ff-smzwv      1/1     Running             0          4m42s
etcd-minikube                              1/1     Running             0          6h38m
kube-addon-manager-minikube                1/1     Running             8          7d6h
kube-apiserver-minikube                    1/1     Running             0          6h38m
kube-controller-manager-minikube           1/1     Running             0          6h40m
kube-proxy-2zvkk                           1/1     Running             0          6h39m
kube-scheduler-minikube                    1/1     Running             7          7d6h
kubernetes-dashboard-ccc79bfc9-bc7qm       1/1     Running             17         7d5h
nginx-ingress-controller-7c66d668b-tcwfs   0/1     ContainerCreating   0          4m41s
storage-provisioner                        1/1     Running             17         7d6h
```

可以看到出现了一个名为nginx-ingress-controller-7c66d668b-tcwfs的pod。

接下来创建一个Ingress资源。新建文件kubia-ingress.yaml。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
  - host: kubia.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubia-nodeport
          servicePort: 80
```

创建ingress之后查看列表。

```sh
$ kubectl get ingresses
NAME    HOSTS               ADDRESS     PORTS   AGE
kubia   kubia.example.com   10.0.2.15   80      4m42s
```

之后配置host文件，追加一行。

```sh
192.168.99.107 kubia.example.com
```

之后访问地址http://kubia.example.com即可。

```sh
$ curl http://kubia.example.com
You've hit kubia-9vqtgs
```

我们可以为同一个host配置多个路径。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
  - host: kubia.example.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: foo
          servicePort: 80
      - path: /bar
        backend:
          serviceName: bar
          servicePort: 80
```

同样我们也可以配置多个host。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  rules:
  - host: foo.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: foo
          servicePort: 80
  - host: bar.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: bar
          servicePort: 80
```

如果客户端发起https连接，那么现在ingress还无法处理。我们可以为ingress增加https的支持，这样的好处不仅仅提高了安全性，而且只要通过ingress进行访问，我们所有的集群中的pod都不需要实现https，只需要支持http即可。

先生成密钥和证书。

```sh
$ openssl genrsa -out tls.key 2048
Generating RSA private key, 2048 bit long modulus
...............................................................................................+++
...+++
$ openssl req -new -x509 -key tls.key -out tls.cert -days 360 -subj /CN=kubia.example.com
```

之后基于证书和密钥为ingress创建证书和密钥。

```sh
$ kubectl create secret tls tls-secret --cert=tls.cert --key=tls.key
secret/tls-secret created
```

现在私钥和证书保存在名为tls-secret的文件中。现在我们就可以支持https协议了。创建一个新文件kubia-ingress-tls.yml。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
spec:
  tls:
  - hosts:
    - kubia.example.com # kubia.example.com域名支持tls连接
    secretName: tls-secret
  rules:
  - host: kubia.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kubia-nodeport
          servicePort: 80
```

你可以选择使用apply命令更新已有资源。

```sh
$ kubectl apply -f kubia-ingree-tls.yml
```

之后访问https地址。

```sh
$ curl -k -s https://kubia.example.com
You've hit kubia-n7dqz
```

### Pod预热

当一个服务创建时，由于被标签选择器选中会加入到端点中，而之后服务会直接将请求转发给刚启动的Pod。但是一个刚启动的Pod往往需要预热，比如加载必要的框架，初始化缓存，加载配置文件等等。

之前我们已经学习过了存活探针，类似于存活探针，k8s为我们提供了可读探针。可读探针会周期性运行，来判断pod是否可以接受请求。如果容器的可读探针返回成功，则表示容器已经准备好接受请求了。

存在三类可读探针：

- Exec探针，在pod中执行一个进程，并按照进程退出码来判断是否可读。
- HTTP GET探针，发送一个HTTP GET请求，并根据HTTP响应的状态码来判断是否可读。
- TCP Socket探针，与容器的指定端口建立TCP连接，如果建立成功则表示可读。

如果一次可读探测失败，那么会将该pod移除出端点，直到之后某次可读探测成功，才将pod加回到端点。

之后我们编辑名为kubia的副本控制器，并追加可读探针。

```yaml
......
  spec:
      containers:
      - image: taodaling/kubia
        readinessProbe:
          exec:
            command:
            - ls
            - /var/ready
          failureThreshold: 3
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
......
```

可读探针会每10s执行一次容器中的ls /var/ready命令，ls在文件存在的情况下退出码为0，否则非0。编辑完服务后删除所有pods，让kubia副本控制器为你重启它们。

```sh
$ kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
kubia-bb9dq   0/1     Running   0          64s
kubia-q725m   0/1     Running   0          64s
kubia-qdkrr   0/1     Running   0          64s
$ kubectl get rc
NAME    DESIRED   CURRENT   READY   AGE
kubia   3         3         0       6m2s
```

之后我们在一个容器中创建`/var/ready`文件并查看状态。

```sh
$ kubectl exec kubia-bb9dq -- touch /var/ready
$ kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
kubia-bb9dq   1/1     Running   0          5m15s
kubia-q725m   0/1     Running   0          5m15s
kubia-qdkrr   0/1     Running   0          5m15s
```

上面这种方式是作为演示使用，真实场景下应该通过判断服务是否可用来返回成功失败。

作为建议，你始终要提供可读探针，否则可能会出现连接拒绝的异常。并且在pod收到关闭信号时，k8s会自动将其从所有服务中移除，因此你不必在处理关闭的期间保证可读探测失败。

### 去中心服务

服务可以为我们提供一个静态的访问地址和端口。但是如果我们需要请求所有服务背后的pods呢，如果这些pods需要相互访问呢。一个选择是客户端通过k8s的API来拉取其它服务的地址和端口。但是由于你始终应该争取自己的应用对k8s无感知，因此这不是一种理想的解决方案。

将一个服务的ClusterIP字段设置为None可以使服务去中心。

我们创建一个kubia-svc-headless.yml文件。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-headless
spec:
  clusterIP: None
  ports:
  - name: http
    port: 80
    targetPort: http
  selector:
    app: kubia
```

创建服务后，查看kubia-headless的详情。

```sh
$ kubectl describe svc kubia-headless
Name:              kubia-headless
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=kubia
Type:              ClusterIP
IP:                None
Port:              http  80/TCP
TargetPort:        http/TCP
Endpoints:         172.17.0.10:8080,172.17.0.8:8080,172.17.0.9:8080
Session Affinity:  None
Events:            <none>
```

接下来我们可以启动DNS查询来查询Pods的IP地址。但是很不幸，你的kubia容器镜像并不包含nslookup（或dig）二进制文件，因此你无法使用DNS查询。要实现DNS相关动作，你可以使用tutum/dnsutils镜像，其位于Docker Hub上，同时包含nslookup和dig二进制文件。如果想要用于开发，你需要重建自己的镜像。但是由于是为了试验，因此我们简单点。

```sh
$ kubectl run dnsutils --image=tutum/dnsutils --generator=run-pod/v1 --command -- sleep infinity
```

这里使用--generator=run-pod/v1选项，表示直接创建pod，不需要任何副本控制器。

等待容器创建好后，执行命令。

```sh
$ kubectl exec dnsutils -- nslookup kubia-headless
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   kubia-headless.default.svc.cluster.local
Address: 172.17.0.10
Name:   kubia-headless.default.svc.cluster.local
Address: 172.17.0.9
Name:   kubia-headless.default.svc.cluster.local
Address: 172.17.0.8
$ kubectl get pods -o wide
NAME          READY   STATUS    RESTARTS   AGE     IP            NODE       NOMINATED NODE   READINESS GATES
dnsutils      1/1     Running   0          3m36s   172.17.0.6    minikube   <none>           <none>
kubia-bwfpc   1/1     Running   0          23m     172.17.0.10   minikube   <none>           <none>
kubia-mvxx7   1/1     Running   0          23m     172.17.0.9    minikube   <none>           <none>
kubia-vs84r   1/1     Running   0          23m     172.17.0.8    minikube   <none>           <none>
```

可以看到列出了所有支持kubia-headless的pods信息。

## 卷

由于容器的文件系统是镜像提供的，因此你在容器中对文件系统的写入不会被持久化。容器一旦重启，所有的写入都会丢失。

k8s提供了卷（volume）的定义，它们是pod的一部分，并与pod共享生命周期，即当pod启动时创建，关闭时销毁。一个pod中所有的容器都可以看到卷中的内容，但在这之前需要挂载到容器的文件系统中

除了需要在pod中定义volume，你还需要为容器定义VolumeMount。

如果你仅需要一个空白卷，那么可以使用emptyDir类型的卷。当然k8s还支持其它带内容的卷，内容的装填发生正在容器启动前。卷的类型有如下：

- emptyDir，一个用于存储临时数据的空白卷。
- hostPath，用于挂载来自工作节点的文件系统
- gitRepo，一开始装填git仓库检出内容
- nfs，一个NFS共享
- persistentVolumeClaim，使用一个预先或动态提供的持久化存储。
- configMap，secret，downwardAPI，用于向pod暴露k8s资源和集群信息。

### emptyDir

新建一个文件fortune-pod.yml。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
  - image: luksa/fortune
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
    - name: html
      emptyDir: {}
```

luksa/fortune这个镜像会每10s更新/var/htdocs/index.html文件，这些变动会体现在web-server容器中。

emptyDir是在工作节点的实际硬盘上创建的，因此性能受限于硬盘的读写速度。但是你可以要求k8s在tmpfs文件系统（在内存中而非硬盘上）创建emptyDir。

```yaml
volumes:
- name: html
  emptyDir:
    medium: Memory
```

### gitRepo

一个gitRepo卷是在emptyDir上克隆git仓库得到的。创建文件gitrepo-volume-pod.yml。

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: gitrepo-volume-pod
spec:
 containers:
 - image: nginx:alpine
   name: web-server
   volumeMounts:
   - name: html
     mountPath: /usr/share/nginx/html
     readOnly: true
   ports:
   - containerPort: 80
     protocol: TCP
 volumes:
 - name: html
   gitRepo:
     repository: https://github.com/luksa/kubia-website-example.git
     revision: master
     directory: .
```

默认情况下仓库会克隆在/kubia-website-example下，加了directory后会克隆在/下。

如果你希望能在推送git仓库时自动更新本地仓库，可以使用一些具有与git同步功能的镜像。并在镜像上挂载卷。

### hostPath

一个hostPath卷指向一个节点文件系统中的文件或目录。使用hostPath会使得pod的行为与其被调度到的节点关联。

所以记住只在你需要读写系统文件时使用hostPath，而不要为了跨pods持久化数据使用它。

### NAS

当应用需要持久化数据，并且希望在重新调度pod后依旧能访问数据，此时之前提到的卷类型都不适用。数据必须存储在网络存储（Network-attached storage，NAS）中。

### PersistentVolumes和PersistentVolumeClaims

为了让应用使用存储时与基础架构解耦，k8s引入了两种新的资源。分别是PersistentVolumes和PersistentVolumeClaims。

![](https://raw.githubusercontent.com/taodaling/assets/master/images/2019-01-30-kubernetes/persistent-volume-claim.PNG)

集群管理员建立了底层存储，并通过k8s提供的API将存储注册为Persistent Volume资源，当创建资源时，管理员需要指定它的大小和支持的访问模式。

当集群使用者需要使用持久化存储时，它们首先创建一个Persistent Volume Claim的清单，指定最小大小和需要的访问模式。之后用户向k8s的API提交清单，而k8s负责寻找一块合适的Persistent Volume并将它绑定到Persistent Volume Claim上。

之后Persistent Volume Claim就可以作为卷出现在pod中了。其它用户不能使用相同的PersistentVolume直到它被释放，即与它绑定的Persistent Volume Claim被删除。

接下来我们创建一个持久化卷，先创建一个文件mongodb-pv-host-path.yml。

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity:
    storage: 1Gi
  accessModes: #它可以被一个节点以读写模式挂载，或被多个节点以只读挂载
  - ReadWriteOnce 
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain # 在持久卷释放后，应该保留数据（不擦除数据或删除持久卷）
  hostPath:
    path: /tmp/mongodb
```

我们建立了一个持久卷，其底层存储由目录/tmp/mongodb提供。查看现有的持久卷。

```sh
$ kubectl get pv
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
mongodb-pv   1Gi        RWO,ROX        Retain           Available                                   33s
```

像节点一样，持久卷是全局资源，不属于任何命名空间。

之后我们要构建我们的持久卷声明，持久卷声明的创建与pod的创建是分开的过程，因为你希望pod重新调度后持久卷声明依旧可用。

创建一个mongodb-pvc.yml文件。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  resources:
    requests: #需要1GiB的存储
      storage: 1Gi
  accessModes:
  - ReadWriteOnce #需要持久化卷支持单客户端读写
  storageClassName: "" #之后会学
```

在持久化声明定义文件中，我们对持久化卷提出了需求，要求至少要提供1G的存储，并且支持ReadWriteOnce访问模式。由于我们之前创建的mongodb-pv完全满足这些要求，因此会被选中与声明绑定。

创建资源后查看现有的持久化卷声明。

```sh
$ kubectl get pvc
NAME          STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongodb-pvc   Bound    mongodb-pv   1Gi        RWO,ROX                       9s
$ kubectl get pv
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   REASON   AGE
mongodb-pv   1Gi        RWO,ROX        Retain           Bound    default/mongodb-pvc                           21m
```

支持的访问模式有：

- RWO，Read Write Once，只有一个节点可以挂载它用于读写
- ROX，Read Only Many，多个节点可以挂载它用于读
- RWX，Read Write Many，多个节点可以挂载它用于读写

RWO、ROX、RWX关联的时可以同时挂载它的工作节点数目，而非pods。

持久化卷声明资源时存在于命名空间下的，并且只能被相同命名空间下的pods使用。

接着我们创建一个pod，提供mongo作为NoSQL数据库。编辑文件mongodb-pod.yml。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
    volumes:
    - name: mongodb-data
      persistentVolumeClaim:
        claimName: mongodb-pvc
```

之后登陆mongo并手动写入数据。

```sh
$ kubectl exec -it mongodb -- mongo
> use mystore
switched to db mystore
> db.foo.insert({name: 'foo'})
WriteResult({ "nInserted" : 1 })
> db.foo.find()
{ "_id" : ObjectId("5c6b96833cb8067fbcc2fb5e"), "name" : "foo"
> exit
bye
```

重启mongodb豆荚，再尝试上面命令，看数据是否被持久化了。

```sh
$ kubectl delete pods mongodb
pod "mongodb" deleted
$ kubectl create -f mongodb-pod.yml
pod/mongodb created
$ kubectl exec -it mongodb -- mongo
> use mystore
switched to db mystore
> db.foo.find()
{ "_id" : ObjectId("5c6b96833cb8067fbcc2fb5e"), "name" : "foo" }
> exit
bye
```

使用持久化卷的最大好处是开发人员不需要管理底层存储，只需要再持久化卷声明中提出存储需求就可以了。

最后我们删除持久化卷声明。

```sh
$ kubectl delete pod mongodb
pod "mongodb" deleted
$ kubectl delete pvc mongodb-pvc
persistentvolumeclaim "mongodb-pvc" deleted
```

接下来重建持久化卷声明。

```sh
$ kubectl create -f mongodb-pvc.yml
persistentvolumeclaim/mongodb-pvc created
$ kubectl get pvc
NAME          STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongodb-pvc   Pending                                                     5s
```

可以看到持久化卷声明的状态保持为挂起，与第一次创建的状态不同。我们查看一下持久化卷的状态。

```sh
$ kubectl get pv
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                 STORAGECLASS   REASON   AGE
mongodb-pv   1Gi        RWO,ROX        Retain           Released   default/mongodb-pvc                           54m
```

可以看到持久化卷的状态时Released而非Available。因为你之前使用过了这个卷，它可能包含了数据并且不应该直接绑定到全新的声明上去，这样可以为管理员提供一个清理它的机会。否则一个新的pod可能会读取到之前pod遗留下来的数据，即使声明和pod处于不同的命名空间下。

这些行为源于你将它的persistentVolumeReclaimPolicy设置为Retain，你希望k8s在卷释放后为你保存volume和它的内容。据我所知，要复用卷的唯一方法就是删除后重建。而潜在的文件存储，你可以自行决定是否要保留。

还存在另外两种持久化卷回收策略，Recycle和Delete。前者删除卷内容并使卷可用，后者删除该卷，你可以随时变更策略。

可以看到持久化卷帮助我们开发人员从底层存储解脱了出来，但是运维人员还是需要手动添加持久化卷以供应不同的pod使用。幸运的是，k8s可以通过动态提供持久化卷自动地完成这项任务。

集群管理员可以部署一个持久化卷供应器并定义多个StorageClass对象，允许用户选择他们希望的持久化卷类型，来取代手动创建持久化卷。用户可以在持久化卷声明的定义文件中引用StorageClass。

类似于持久化卷，StorageClass资源属于全局。K8s为大多数流行的云服务提供者提供了供应器。使用供应器的好处是你会有用不完的持久化卷（当然你可能用完了存储空间）。

我们首先需要定义StorageClass资源。新建文件storageclass-fast.yml。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: k8s.io/minikube-hostpath #供应器插件
parameters: #传递给供应器的参数
  type: pd-ssd
```

创建了StorageClass后，我们可以在持久化卷声明定义中引用StorageClass。之后新建文件mongodb-pvc-dp.yml。

```yaml

```

创建资源后查看状态。

```sh
$ kubectl get pvc
NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongodb-pvc   Bound    pvc-382a9399-3419-11e9-b21a-0800279f3491   100Mi      RWO            fast           5s
```

可以看到一个名为pvc-382a9399-3419-11e9-b21a-0800279f3491的持久化卷被自动创建了。

查看一下已有的StorageClass。

```sh
$ kubectl get sc
NAME                 PROVISIONER                AGE
fast                 k8s.io/minikube-hostpath   38m
standard (default)   k8s.io/minikube-hostpath   8d
```

名为standard的StorageClass被标记为default。如果你在创建持久化卷声明时既没有指定storageClassName，那么会使用standard为你提供动态持久化卷。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc-standard
spec:
  resources:
    requests:
      storage: 100Mi
  accessModes:
  - ReadWriteOnce
```

创建持久化卷声明后查看StorageClass。

```sh
$ kubectl get pvc
NAME                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongodb-pvc            Bound    pvc-382a9399-3419-11e9-b21a-0800279f3491   100Mi      RWO            fast           17m
mongodb-pvc-standard   Bound    pvc-92f34bc1-341b-11e9-b21a-0800279f3491   100Mi      RWO            standard       11s
```

这也是我们之前希望持久化卷声明绑定预先创建的持久化卷时将storageClassName设置为""的原因，只要这样才会使用预先创建的持久化卷。

## ConfigMap和Secrets

一般应用在开始时，会直接通过命令行传递配置信息。但是随着配置项越来越多，通过命令行传递所有的参数变得不现实了，因此开始采用配置文件的方式。还有另外一种替代命令行参数的方式，通过环境变量传递配置信息。比如官方的MYSQL镜像，就是通过读取环境变量中的MYSQL_ROOT_PASSWORD来设置密码。

为什么在容器世界中，环境变量会这么流行？因为使用配置文件，你需要将配置文件硬备份到容器镜像中或者挂载包含配置文件的卷到容器文件系统。硬备份类似于硬编码，因为你每次修改配置文件后都需要重新构建镜像，并且所有有权接触到这个镜像的人都能得到你的配置信息，包括那些隐私信息，比如证书以及密钥等。使用卷会稍微好一些，但是依旧要求你在启动容器之间预先把配置文件保存到目录下。

当然你也可以使用之前提到的gitRepo挂载的方式，这样可以控制版本，并且保证全局一致。但是k8s也提供了自己的存储配置数据的方式，在k8s中，存储配置数据的资源称为ConfigMap（配置表）。

你可以通过下面几种方式配置你的应用。

- 向容器传递命令行参数
- 为每个容器设置自定义环境变量
- 通过特殊的卷挂载配置文件到容器中

### 命令行传递

你可以覆盖镜像中自带的CMD。

```yaml
kind: Pod
Spec:
  containers:
  - image: some/image
    command: ["/bin/command"]
    args: ["arg1", "arg2", "arg3"]
```

command和args在容器运行期间不能改变。

### 环境变量传递

```yaml
kind: Pod
Spec:
  containers:
  - image: luksa/fortune:env
    env:
    - name: INTERVAL
      value: "30"
```

在定义环境变量时后定义的环境变量可以引用先定义的环境变量。

```yaml
env:
- name: FIRST_VAR
  value: "foo"
- name: SECOND_VAR
  value: "${FIRST_VAR}bar"
```

使用环境变量传递配置信息的缺点时你需要为不同的环境准备不同的pod定义文件。

### ConfigMap

应用配置的关注点是按照不同的环境选择不同的配置，并与代码分离。在微服务的场景下，你将自己的服务组合到系统之中，此时你可以认为pod的定义是你的应用的代码的一部分，因此pod定义文件中不应该包含配置数据。

k8s允许将配置分割到不同的对象中，这类对象称为ConfigMap，由键值对组成，值可以是文本或文件。

应用不需要直接读取ConfigMap，甚至不需要知道它的存在。表的内容会通过环境变量或文件的形式传递到容器中，当然你也可以选择通过命令行参数的方式进行传递。

我们使用create configmap来创建一个configmap。

```sh
$ kubectl create configmap fortune-config --from-literal=sleep-interval=25
```

之后查看configmap。

```sh
$ kubectl get cm
NAME             DATA   AGE
fortune-config   1      5s
$ kubectl describe cm fortune-config
Name:         fortune-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
sleep-interval:
----
25
```

configmap中的关键字只能包含字母、数字、破折线、下划线、点号。

要建立一个包含多个键值对的configmap。

```sh
$ kubectl create configmap myconfgimap --from-literal=foo=bar --from-literal=bar=baz
```

我们也可以使用定义文件的方式来创建configmap。

```sh
$ kubectl get configmap fortune-config -o yaml
apiVersion: v1
data:
  sleep-interval: "25"
kind: ConfigMap
metadata:
  creationTimestamp: "2019-02-20T01:54:23Z"
  name: fortune-config
  namespace: default
  resourceVersion: "231594"
  selfLink: /api/v1/namespaces/default/configmaps/fortune-config
  uid: 723fd4dc-34b2-11e9-b149-0800279f3491
```

你也可以通过配置文件来创建configmap。

```sh
$ kubectl create configmap my-config --from-file=config-file.conf
```

在这种情况下，my-config中仅包含一对键值对，键为文件名，值为文件内容。你也可以手动设置键。

```sh
$ kubectl create configmap my-config --from-file=customkey=config-file.conf
```

--from-file和--from-literal一样允许出现多次，并且支持混用。

除了导入一个文件，你还可以导入整个目录。

```sh
$ kubectl create configmap my-config --from-file=/path/to/dir
```

这等价于为/path/to/dir目录下的每个文件在my-config中创建一个键值对，键为文件名，值为文件内容。

![](https://raw.githubusercontent.com/taodaling/assets/master/images/2019-01-30-kubernetes/configmap.PNG)

要将configmap中的值以环境变量形式进行传递。

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: fortune-env-from-configmap
spec:
 containers:
 - image: luksa/fortune:env
   env:
   - name: INTERVAL
     valueFrom:
       configMapKeyRef:
         name: fortune-config
         key: sleep-interval
```

但是如果你要暴露configmap中的所有键值对作为环境变量，一个个设置会很麻烦，k8s允许我们直接导入整个configmap。

```yaml
spec:
 containers:
 - image: some-image
   envFrom:
   - prefix: CONFIG_ #所有的键都会带上CONFIG_前缀
     configMapRef:
       name: my-config-map
```

在导入整个configmap时，如果键包含类似破折号或者点号等不允许出现在环境变量名称中的符号，那么这个键和对应的值会被跳过（不会被设置到环境变量中）。

configmap不能直接使用于命令行参数传递，但是你可以通过环境变量间接使用。

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: fortune-args-from-configmap
spec:
 containers:
 - image: luksa/fortune:args
   env:
   - name: INTERVAL
     valueFrom:
       configMapKeyRef:
         name: fortune-config
         key: sleep-interval
   args: ["$(INTERVAL)"] 
```

一般环境变量和命令行参数是用来传递较短的配置项。configmap中的值可以是整个文件，你可以通过一种特殊的卷——configMap类型来将configmap中的键值对以文件的形式暴露给容器。

configMap类型的卷会将configmap中的每个键值对都作为一个独立的文件进行暴露。容器可以通过读取卷下的文件来获取配置信息。

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: fortune-configmap-volume
spec:
 containers:
 - image: nginx:alpine
   name: web-server
   volumeMounts:
 ...
   - name: config
   mountPath: /etc/nginx/conf.d
   readOnly: true
 ...
 volumes:
 ...
 - name: config
   configMap:
     name: fortune-config
 ...
```

你也可以仅通过configVolume暴露一部分的配置。

```yaml
volumes:
 - name: config
   configMap:
     name: fortune-config
     items:
     - key: my-nginx-config.conf
       path: gzip.conf 
```

如果单独指定暴露项，必须为每个不同的暴露项指定一个文件。

由于之前都是以目录的形式挂载，以目录形式挂载的问题在于会覆盖同名目录。比如你将卷挂载到/etc，那么原先/etc下的所有内容都会丢失。这是linux的底层挂载机制导致的，当你挂载一个卷到非空目录，该非空目录下的所有原先文件都会丢失。为了避免影响原先的文件，我们不能以目录的形式挂载，而选择以文件的形式挂载。

```yaml
spec:
 containers:
 - image: some/image
   volumeMounts:
   - name: myvolume
     mountPath: /etc/someconfig.conf
     subPath: myconfig.conf 
```

默认情况下所有configMap卷下的文件的权限都是664。你可以变更defaultMode模式来修改权限。

```yaml
volumes:
- name: config
  configMap:
    name: fortune-config
    defaultMode: "6600"
```

### 热更新配置

使用环境变量和命令行参数的缺点是无法在容器启动后更新配置信息。但是使用configmap结合卷的方式允许你在不重启容器的前提下更新配置信息。

当你更新了configmap，所有引用它的卷都会被更新。因此只要进程能检测到配置文件更新并重载它们就可以实现热更新。注意热更新仅在挂载卷到目录时生效，而挂载单独文件是无效的。

k8s也在支持在更新完配置文件后向容器发送一个信号。

也许你会好奇，是否有可能在k8s更新文件到一半，你的应用就发现了配置文件变动并重载。但是这是不可能的，因为k8s使用了链接的方式，使得更新的完成对于我们来说是原子性操作。卷的挂载目录实际上是链接，当你变更了configmap后，k8s会创建一个新的目录，重新生成文件后替换链接。

### secrets

我们之前谈论的都是向容器传递非敏感信息，但是实际配置中往往会包含诸如密钥证书等敏感信息。要保存这类信息，k8s提供了一类称为Secret的资源。Secret类似Configmap，它们保存的也是键值对。它们的用法也是类似，你可以将键值对通过文件或环境变量的方式传递到容器中。

k8s通过确保只有pods需要敏感数据时才为pods所在节点提供secret数据，并且secret数据只能保存在内存中，而不会写入到物理存储中。

在主节点上，secret过去以非加密的形式存放，这意味着必须先确保master是安全的，才能保证secret中存储的数据是安全的。这不仅仅意味着我们需要保证master是安全的，还要保证，未经授权的用户不能使用API服务器，因为任何人建立的pod都有权访问secret中的数据。在k8s的1.7版本后，主节点以加密的形式存储secret。

在使用secret之前，我们可以通过describe pods查看任意pods的描述信息。其中应该包含这样一段内容。

```yaml
......
  Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-rb9kd (ro)
......
  default-token-rb9kd:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-rb9kd
    Optional:    false
......
```

每个pods都有一个secret类型的卷自动挂载到它上面。由于secret也是一类资源，我们可以列出所有的secret。

```sh
$ kubectl get secrets
NAME                  TYPE                                  DATA   AGE
default-token-rb9kd   kubernetes.io/service-account-token   3      9d
tls-secret            kubernetes.io/tls                     2      44h

$ kubectl describe secrets default-token-rb9kd
Name:         default-token-rb9kd
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: default
              kubernetes.io/service-account.uid: abb56903-2da0-11e9-a6d4-0800279f3491

Type:  kubernetes.io/service-account-token

Data
====
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9...
ca.crt:     1066 bytes
```

可以看到一个secret对象包含了三个键值对，ca.crt、namespace、token。这里面包含了能让你的pod和API服务交流所需的所有信物。当然你需要能将你的应用和k8s解耦，但是如果应用必须直接和k8s交互，你就需要这些信物了。

首先我们创建一个密钥和证书文件。

```sh
$ openssl genrsa -out https.key 2048
$ openssl req -new -x509 -key https.key -out https.cert -days 3650 -subj /CN=www.kubia-example.com
$ echo bar > foo
```

之后根据这些文件创建secret对象。

```sh
$ kubectl create secret generic fortune-https --from-file=https.key --from-file=https.cert --from-file=foo
secret/fortune-https created
```

这里我们创建了一个generic类型的secret，除了generic类型外，你还可以创建tls类型的secret。

查看之前创建的secret的yaml格式。

```sh
$ kubectl get secret fortune-https -o yaml
apiVersion: v1
data:
  foo: YmFyIA0K
  https.cert: LS0tLS1CRUdJTiBDRVJUSUZJQ0......
  https.key: LS0tLS1CRUdJTiBSU0EgUFJJ......
kind: Secret
metadata:
  creationTimestamp: "2019-02-20T06:22:00Z"
  name: fortune-https
  namespace: default
  resourceVersion: "249305"
  selfLink: /api/v1/namespaces/default/secrets/fortune-https
  uid: d4f5d26c-34d7-11e9-b149-0800279f3491
type: Opaque
```

可以看到我们之前的foo:bar现在变成了foo:YmFyIA0K，这是因为保存在secret中的值会通过Base64编码。而ConfigMap存储的明文，因此你可以通过定义文件来创建编辑ConfigMap，但不能以相同的方式创建Secret。

之所以选择使用Base64的原因很简单，为了在值含有二进制数据时，值能在定义文件中正常展示。你也可以使用secret来存储一些二进制数据，但是要记住secret的最大长度不能超过1MB。

事实上你也可以通过stringData项来为secret追加明文属性。但是stringData时只写的。

```yaml
kind: Secret
apiVersion: v1
stringData:
 foo: plain text
data:
 https.cert: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCekNDQ...
 https.key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcE...
```

当你将secret通过secret类型卷暴露给容器时，会将secret中的解密后保存到文件中，规则类似于configmap。

接着通过secret卷挂载到容器中。

```yaml
volumeMounts:
- name: certs
  mountPath: /etc/https/certs
......
volumes:
- name: certs
  secret:
    secretName: fortune-https
```

要记住secret卷使用的存储是内存，因此secret中的数据是永远不会落盘的。

同理，要在环境变量中使用secret，和configmap一样的方式。

```yaml
env:
  - name: FOO_SECRET
    valueFrom:
      secretKeyRef:
        name: fortune-https
        key: foo
```

尽管k8s允许你通过环境变量暴露secret，但是这不是一种好的方式。因为环境变量会打印在日志中，并且由于子进程会继承父进程的所有环境变量，因此如果你的应用调用了第三方二进制文件，那么它们就可以得到你的secret中的数据。因此你总是应该避免使用环境变量传递secret，而应该使用secret卷。

## 向k8s提供secret

你现在已经学会了怎么向自己的应用提供secret，但是有时候你需要为k8s提供secret，比如你的镜像存储在私有仓库中，这样你就必须向k8s提供必要的认证资料。

要使用docker私有仓库，你需要创建一个docker-registry类型的secret。

```sh
$ kubectl create secret docker-registry mydockerhubsecret --docker-username=myusername --docker-passwrod=mypassword --docker-email=my.email@provider.com
```

要使用之前创建的mydockerhubsecret从docker仓库拉取镜像。

```yaml
spec:
  imagePullSecrets:
  - name: mydockerhubsecret
```

## 状态集合

你可能遇到这种情况，每个pod都自带状态。为了保证pod重启后状态不丢失，因此pod的状态会落盘。但是每个pod的状态是独立的，因此它们需要独立的存储。而为了让pod支持自动管理和水平扩展，我们需要使用类似副本集合来管理它们。这常见于一些存储数据库，它们拥有独立的日志。

一种简单的方式是创建若干个副本集合，每个副本集合管理一个pod，并在每个副本集合的template里面分配一个独立的存储。这种方式的缺点是水平扩展变得笨重。

还有一种方式是在分配的卷下建立自己的子目录。应用重启后，随机抢占一个目录，但是缺点是应用必须内部实现抢占算法，并且多个pod使用一个存储很可能会使得存储的读写速度成为性能瓶颈。

除了存储外，一个应用可能需要一个长时间可用的身份证，比如IP地址。但是一个pod被重启后，会拥有全新的域名和IP，虽然存储是不变的，但是还是可能会带来问题。这常见于一些分布式存储应用，其中特定的应用需要知道所有其它集群成员的IP地址或域名，比如zookeeper。而由于pod重启带来的身份丢失，因此整个集群必须在其中一个节点重启后重新配置。

要解决上面这个问题，我们可以为每个pod创建一个单独的service，由于service拥有不变的域名和IP，因此可以解决身份改变的问题。但是它的瓶颈也在于无法水平扩展，并且需要更改每个pod的定义文件以提供不同的标签。

K8s为我们提供了StatefulSets（状态集）来解决这些问题。状态集用于管理不可替代的应用，每个应用有一个稳定的名字和状态。

我们用宠物vs牛来帮助我们理解状态集和副本集的区别。每头牛都是一样的，没有自己的名字，牛死去了，可以清理后购入一头新的牛，没人会注意到它的发生。但是宠物不一样，每只宠物多是独一无二的，因此要替换宠物时，我们需要找到一只完全一样的宠物。显然管理牛要比管理宠物要简单的多。

副本集合管理的pod就时牛，而状态集合管理的pod是宠物。一旦状态集合的pod死去，那么会创建一个拥有相同名称、网络和状态的新pod替代它。

除了为你重启相同的pod外，状态集合还能进行水平扩展。

由状态集合第一次启动的pod的状态是可以预测的，而不必随机分配。每个副本获得一个从0起始递增的互不相同的序号，而pod的名称、域名、存储都可以通过序号计算得到。与常规pod不同，带状态的pod有时需要能通过域名进行定位。

一个状态集合还要求你创建一个对应的去中心服务（headless service），用于提供为每个pod提供网络ID。通过这个服务，每个pod都能得到自己的DNS入口，因此集群中的客户端可以通过pod的域名进行定位。如果你的去中心服务属于default命名空间并且命名为foo，它后面存在一个叫做A-0的pod，你可以通过FQDN来访问它，a-0.foo.default.svc.cluster.local。

状态集合可以在不同的节点上重启pod，但是会保证二者拥有相同的域名、状态、名字。

当你增加状态集合的副本数时，比如从2增加到3，则会创建一个新的Pod，pod的序号是2。当你减少状态集合的副本数时，比如从3减少2，则会删除序号最大的pod，即序号为2的pod。

由于大多数带状态应用都不能快速处理减员，因此每次水平缩减最多减少一个副本。因为分布式存储可能会在多台机器上存储数据副本来保证数据不会丢失，但是如果同时移除多个节点，可能这几个节点上拥有某份数据的所有副本，那么这份数据将永远丢失。而由于数据的副本数一般至少有2，因此一次性减少一个副本，可以为应用提供一定的时间进行副本的重建。也因为这个原因，状态集合同样不允许在存在不健康副本时缩减副本数目，因为这可能会导致应用同时失去两个节点。

为了能为每个副本提供一个独立的持久化卷声明。状态集合也支持一个或多个卷声明模板，用来为每个副本创建独立的副本卷声明。

要增加状态集合的副本数，会自动创建多个对象，pod、持久化卷声明。但是在减少状态集合的副本数时仅会删除pod。因为带状态的pod意味着运行带状态的应用，即应用的状态（持久化在卷上）是非常重要的。如果在副本移除时删除卷声明，会顺带清空卷内容，这可能会带来灾难性后果。尤其在你缩减副本数只需要修改replica属性这么简单，因此你被要求必须手动删除持久化卷声明来释放底层的存储。

实际上在你缩减了副本并释放出空余的持久化卷声明后，之后增加副本会重用这个持久化卷声明。这意味着如果你是因为不小心缩减了副本，那么你可以通过增加副本来恢复到之前的状态而不会丢失数据。