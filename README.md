构建基于 Kubernetes 的大数据分析平台
=================================================

构建基于 Kubernetes 的大数据分析平台

## 使用Kind创建本地Kubernetes集群环境
[kind][kind] (Kubernetes IN Docker) 让你能够快速在本地计算机上运行 Kubernetes集群。 kind 要求你安装并配置好 Docker。

下面是使用kind创建一个多worker的Kubernetes集群，将下面的内容保存为`kind-config.yaml`:

```
kind: Cluster 
apiVersion: kind.x-k8s.io/v1alpha4 
nodes: 
- role: control-plane 
- role: worker 
- role: worker 
- role: worker 
networking: 
  disableDefaultCNI: false
```

然后通过`kind`的命令行程序创建Kubernetes集群
```
kind create cluster --name bigdata --config kind-config.yaml 
```

[kind][kind] 会自动的pull需要的docker镜像(镜像比较大)，并创建由一个控制节点，3个工作节点组成的Kubernetes集群。

```shell
λ kind create cluster --name bigdata --config kind-config.yaml    
Creating cluster "bigdata" ...                                    
 • Ensuring node image (kindest/node:v1.20.2) �  ...              
 ✓ Ensuring node image (kindest/node:v1.20.2) �                   
 • Preparing nodes � � � �   ...                                  
 ✓ Preparing nodes � � � �                                        
 • Writing configuration �  ...                                   
 ✓ Writing configuration �                                        
 • Starting control-plane �️  ...                                 
 ✓ Starting control-plane �️                                      
 • Installing CNI �  ...                                          
 ✓ Installing CNI �                                               
 • Installing StorageClass �  ...                                 
 ✓ Installing StorageClass �                                      
 • Joining worker nodes �  ...                                    
 ✓ Joining worker nodes �                                         
Set kubectl context to "kind-bigdata"                             
You can now use your cluster with:                                
                                                                  
kubectl cluster-info --context kind-bigdata                       
                                                                  
Have a nice day! �                                                
```

完毕之后，可以通过`kubectl`命令来查看集群的信息:

```shell
$ kubectl cluster-info --context kind-bigdata
Kubernetes control plane is running at https://127.0.0.1:50204
KubeDNS is running at https://127.0.0.1:50204/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

$ kubectl get nodes
NAME                    STATUS   ROLES                  AGE     VERSION
bigdata-control-plane   Ready    control-plane,master   3m58s   v1.20.2
bigdata-worker          Ready    <none>                 3m6s    v1.20.2
bigdata-worker2         Ready    <none>                 3m6s    v1.20.2
bigdata-worker3         Ready    <none>                 3m6s    v1.20.2
```

这里可以通过`docker ps`命令来检查一个运行的Docker容器:

```shell
$ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS         PORTS                       NAMES
7f9fb033c531   kindest/node:v1.20.2   "/usr/local/bin/entr…"   12 minutes ago   Up 6 minutes   127.0.0.1:55289->6443/tcp   bigdata-control-plane
4ac75ba79030   kindest/node:v1.20.2   "/usr/local/bin/entr…"   12 minutes ago   Up 6 minutes                               bigdata-worker2
5a27977d042a   kindest/node:v1.20.2   "/usr/local/bin/entr…"   12 minutes ago   Up 6 minutes                               bigdata-worker
8589f4d65b40   kindest/node:v1.20.2   "/usr/local/bin/entr…"   12 minutes ago   Up 6 minutes                               bigdata-worker3
```


然后需要将Master的kubeconfig备份出来，通过kubeconfig配置拷贝到相应管理节点，就可以通过`kubectl`命令来管理Kubernetes集群了。

```
$ kind get kubeconfig --name=bigdata > kubeconfig
$ cp kubeconfig ~/.kube/config
```

## Helm3
Helm 是Kubernetes的包管理器，可以方便的查找、分享和使用软件构建 Kubernetes 应用包。
基于Helm，可以大大的优化基础运维建设及业务应用运维： 
- 基础设施，更方便地部署与升级基础设施，如 gitlab，prometheus，grafana，ES 等
- 业务应用，更方便地部署，管理与升级公司内部应用，为公司内部的项目配置 Chart，使用 helm 结合 CI，在 k8s 中部署应用如一行命令般简单方便

### 安装Helm3

这里参考官方文档 [安装 helm](https://helm.sh/docs/intro/install/)


## Hadoop on Kubernetes
下面在Kubernetes集群上构建整个基于Hadoop体系的大数据平台。

Hadoop组件列表：
- HDFS

### HDFS on Kubernetes
首先安装HDFS的Helm repo
```
helm repo add gaffer https://gchq.github.io/gaffer-docker
```
然后通过Helm repo 安装 HDFS
```
helm install my-hdfs gaffer/hdfs --version 0.16.0 -n bigdata --create-namespace
```
如果需要查看默认的heml values配置，可以通过命令 `helm show values gaffer/hdfs` 来查看。

默认的HDFS配置为:
- 1个NameNode, 10GB的存储空间
- 3个DataNode, 20GB的存储空间
- replication 3副本
- 一个HDFS shell的POD

安装成功之后，可以通过`kubectl`命令查看pod的状态情况:
```
$ kubectl get pods -n bigdata
NAME                             READY   STATUS    RESTARTS   AGE
my-hdfs-datanode-0               1/1     Running   0          112s
my-hdfs-datanode-1               1/1     Running   0          112s
my-hdfs-datanode-2               1/1     Running   0          112s
my-hdfs-namenode-0               1/1     Running   0          112s
my-hdfs-shell-77799cd897-q279f   1/1     Running   0          112s
```

```
$ kubectl get all -n bigdata
NAME                                 READY   STATUS    RESTARTS   AGE
pod/my-hdfs-datanode-0               1/1     Running   0          9m54s
pod/my-hdfs-datanode-1               1/1     Running   0          9m54s
pod/my-hdfs-datanode-2               1/1     Running   0          9m54s
pod/my-hdfs-namenode-0               1/1     Running   0          9m54s
pod/my-hdfs-shell-77799cd897-q279f   1/1     Running   0          9m54s

NAME                        TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                               AGE
service/my-hdfs-datanodes   ClusterIP   10.96.127.2   <none>        80/TCP                                9m54s
service/my-hdfs-namenodes   ClusterIP   None          <none>        9870/TCP,8020/TCP,8021/TCP,8022/TCP   9m54s

NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-hdfs-shell   1/1     1            1           9m54s

NAME                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/my-hdfs-shell-77799cd897   1         1         1       9m54s

NAME                                READY   AGE
statefulset.apps/my-hdfs-datanode   3/3     9m54s
statefulset.apps/my-hdfs-namenode   1/1     9m54s
```

由于开源版本没有处理`dfs.permission`的权限，所以将Charts pull下来做了修改，在`hdfs-core.xml`中添加如下配置：

```
<property>
    <name>dfs.permissions</name>
    <value>false</value>
</property>
```

然后通过本地目录安装该Chart:
```
helm install my-hdfs ./helm/hdfs --version 0.16.0 -n bigdata --create-namespace
```

#### HDFS shell
由于创建了一个HDFS shell的POD，所以可以通过该POD的shell通过 HDFS 的shell来操作HDFS

```
$ kubectl exec -n bigdata -it my-hdfs-shell-77799cd897-q279f -- /bin/bash
hadoop@my-hdfs-shell-77799cd897-q279f:/$ hadoop fs -ls /
hadoop@my-hdfs-shell-77799cd897-q279f:/$ hadoop fs -mkdir /tmp
hadoop@my-hdfs-shell-77799cd897-q279f:/$ hadoop fs -ls /
Found 1 items
drwxr-xr-x   - hadoop supergroup          0 2021-11-02 03:01 /tmp
```

#### HDFS NameNode UI
下面通过` kubectl port-forward` 的方式将NameNode的UI暴露出来:

```
kubectl port-forward -n bigdata my-hdfs-namenode-0 29870:9870
```

然后就可以通过NameNode UI来查看HDFS的信息及状态了。

![image.png](https://ae02.alicdn.com/kf/H52fabe5b26e34b29aaa6fb5908690058f.png)







## Troubleshooting

- 无法通过DNS访问Service

```shell
$ kubectl run -it -n bigdata --rm --restart=Never busybox --image=busybox sh
If you don't see a command prompt, try pressing enter.
/ # cat /etc/resolv.conf
search bigdata.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

使用Debug容器
```
kubectl run -it --rm --restart=Never busybox --image=busybox sh
```

**DNS访问规则**
- POD_NAME.SVC_NAME.NS
- POD_NAME.SVC_NAME.NS.cluster.local



## 参考
- [HokStack - Run Hadoop on Kubernetes](https://docs.hokstack.io/en/latest/)
- [Helm](https://helm.sh/zh/)
- [Apache Spark on Kubernetes](https://github.com/apache-spark-on-k8s)
- [Ververica Platform](https://www.ververica.com/platform)


[kind]: https://kind.sigs.k8s.io/

