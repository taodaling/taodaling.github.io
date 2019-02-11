---
categories: tool
layout: post
---
- Table
{:toc}
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
$ kubectl run kubia --image=taodaling/kubia --port=8080 --generator=run-pod/v1
pod/kubia created
```

这样你实际上就基于镜像taodaling/kubia启动了自己的容器。

你可能希望列举出所有运行的容器，比如`kubectl get containers`，但是k8s并不会直接暴露单个容器，它使用了多个同地址容器的概念，这样一组容器称为豆荚（pod）。豆荚是一组紧密关联的容器，它们总是会在同一个工作节点运行，并处于相同的Linux命名空间下。每一个pod都像一个分离的逻辑机器一样，拥有自己的IP，域名，进程等。只有在同一个pod下才会有这样的待遇，属于不同pod的两个容器，即使运行在同一个工作节点上，也会像运行在不同的机器上一样无法直接交互。

![](https://raw.githubusercontent.com/taodaling/assets/master/images/2019-01-30-kubernetes/pod.PNG)

你可以拉取pod列表。

```sh
$ kubectl get pods
NAME          READY   STATUS         RESTARTS   AGE
kubia-2lhvt   0/1     ErrImagePull   0          99m
```

类似的，你可以用`kubectl describe pod`来查看一个pod的详细信息。