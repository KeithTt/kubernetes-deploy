# kubernetes-deploy
一键部署高可用kubernetes集群

三台master，四台node，系统版本为CentOS7。

| IP            | ROLE          | 
|:-------------:|:-------------:| 
| 172.60.0.226  | master01      | 
| 172.60.0.86   | master02      |   
| 172.60.0.106  | master03      |  
| 172.60.0.227  | node01        | 
| 172.60.0.228  | node02        |   
| 172.60.0.44   | node03        |  
| 172.60.0.46   | node04        |

这里安装的kubernetes版本为1.5.1，docker版本为1.12.3

三个master节点通过keepalived实现高可用。结构如下图，参考官网。

![](http://kubernetes.io/images/docs/ha.svg)

## installation guide

1、在一台单独的server上启动一个http-server，用来存放image和rpm包等文件，脚本会从此处拉取文件。
```
# nohup python -m SimpleHTTPServer &
Serving HTTP on 0.0.0.0 port 8000 ...
```
这是我的http-server地址：http://172.16.200.90:8000/ 
```
# tree 
.
├── etcd
│   ├── deploy-etcd.sh
│   └── temp-etcd
│       ├── etcd
│       └── etcdctl
├── images
│   ├── dnsmasq-metrics-amd64_1.0.tar
│   ├── etcd_v3.0.15.tar
│   ├── exechealthz-amd64_1.2.tar
│   ├── flannel-git_0.7.0.tar
│   ├── kube-apiserver-amd64_v1.5.1.tar
│   ├── kube-controller-manager-amd64_v1.5.1.tar
│   ├── kube-discovery-amd64_1.0.tar
│   ├── kubedns-amd64_1.9.tar
│   ├── kube-dnsmasq-amd64_1.4.tar
│   ├── kube-proxy-amd64_v1.5.1.tar
│   ├── kubernetes-dashboard-amd64.tar
│   ├── kube-scheduler-amd64_v1.5.1.tar
│   └── pause-amd64_3.0.tar
├── k8s-deploy.sh
├── network
│   └── kube-flannel.yaml
├── nohup.out
├── README.md
└── rpms
    ├── docker.tar.gz
    ├── haproxy.tar.gz
    ├── k8s.tar.gz
    └── keepalived.tar.gz
```

2、部署master01，在master01上执行脚本。
```
# curl -L http://172.60.0.43:8000/k8s-deploy.sh | bash -s master --api-advertise-addresses=172.60.0.87 \
	--external-etcd-endpoints=http://172.60.0.226:2379,http://172.60.0.86:2379,http://172.60.0.106:2379

172.60.0.43:8000 	是http-server

--api-advertise-addresses 	 是vip地址

--external-etcd-endpoints 	 是etcd集群的地址

记录下你的 token 输出，node节点加入集群时需要使用该token。
```

3、部署master02和master03。这里需要分别设置两个节点与master01的ssh互信。然后分别在master02和master03上执行脚本。完成后会自动和master01组成冗余。
```
# curl -L http://172.60.0.43:8000/k8s-deploy.sh | bash -s replica --api-advertise-addresses=172.60.0.87 \
	--external-etcd-endpoints=http://172.60.0.226:2379,http://172.60.0.86:2379,http://172.60.0.106:2379
```

上面步骤完成之后，就实现了master节点的高可用。

4、部署node。在每个node上分别执行脚本就即可。
```
# curl -L http://172.60.0.43:8000/k8s-deploy.sh |  bash -s join --token=3635d0.6d0caa140b219bc0 172.60.0.87   	
```

这里的token就是部署master01完成后记录下的token。

加入集群时，这里有可能会报 refuse 错误，将 kube-discovery 扩容到三个副本即可。

```
# kubectl scale deployment --replicas 3 kube-discovery -n kube-system 
```

5、完成后就得到了一个完整的高可用集群。
```
# kubectl get node
NAME             STATUS         AGE
kube-node02      Ready          22h
kuber-master01   Ready,master   23h
kuber-master02   Ready,master   23h
kuber-master03   Ready,master   23h
kuber-node01     Ready          23h
kuber-node03     Ready          23h
kuber-node04     Ready          23h

# kubectl get pod --all-namespaces -o wide
NAMESPACE     NAME                                     READY     STATUS    RESTARTS   AGE       IP             NODE
kube-system   dummy-2088944543-191tw                   1/1       Running   0          1d        172.60.0.87    kuber-master01
kube-system   kube-apiserver-kuber-master01            1/1       Running   0          1d        172.60.0.87    kuber-master01
kube-system   kube-apiserver-kuber-master02            1/1       Running   0          23h       172.60.0.86    kuber-master02
kube-system   kube-apiserver-kuber-master03            1/1       Running   0          23h       172.60.0.87    kuber-master03
kube-system   kube-controller-manager-kuber-master01   1/1       Running   0          1d        172.60.0.87    kuber-master01
kube-system   kube-controller-manager-kuber-master02   1/1       Running   0          23h       172.60.0.86    kuber-master02
kube-system   kube-controller-manager-kuber-master03   1/1       Running   0          23h       172.60.0.87    kuber-master03
kube-system   kube-discovery-1769846148-53vs5          1/1       Running   0          1d        172.60.0.87    kuber-master01
kube-system   kube-discovery-1769846148-m18d0          1/1       Running   0          23h       172.60.0.87    kuber-master03
kube-system   kube-discovery-1769846148-tf0m9          1/1       Running   0          23h       172.60.0.86    kuber-master02
kube-system   kube-dns-2924299975-80fnn                4/4       Running   0          1d        10.244.0.2     kuber-master01
kube-system   kube-flannel-ds-51db4                    2/2       Running   0          23h       172.60.0.87    kuber-master01
kube-system   kube-flannel-ds-gsn3m                    2/2       Running   4          23h       172.60.0.227   kuber-node01
kube-system   kube-flannel-ds-httmj                    2/2       Running   0          23h       172.60.0.86    kuber-master02
kube-system   kube-flannel-ds-tq4xn                    2/2       Running   0          23h       172.60.0.87    kuber-master03
kube-system   kube-flannel-ds-w206v                    2/2       Running   1          23h       172.60.0.44    kuber-node03
kube-system   kube-flannel-ds-x1qv3                    2/2       Running   0          22h       172.60.0.228   kube-node02
kube-system   kube-flannel-ds-xzn9l                    2/2       Running   1          23h       172.60.0.46    kuber-node04
kube-system   kube-proxy-67m5m                         1/1       Running   0          23h       172.60.0.44    kuber-node03
kube-system   kube-proxy-6gkm4                         1/1       Running   0          23h       172.60.0.86    kuber-master02
kube-system   kube-proxy-7l8c8                         1/1       Running   0          1d        172.60.0.87    kuber-master01
kube-system   kube-proxy-mb650                         1/1       Running   0          23h       172.60.0.87    kuber-master03
kube-system   kube-proxy-nb24x                         1/1       Running   0          23h       172.60.0.46    kuber-node04
kube-system   kube-proxy-qlwhj                         1/1       Running   0          22h       172.60.0.228   kube-node02
kube-system   kube-proxy-rhwrw                         1/1       Running   1          23h       172.60.0.227   kuber-node01
kube-system   kube-scheduler-kuber-master01            1/1       Running   0          1d        172.60.0.87    kuber-master01
kube-system   kube-scheduler-kuber-master02            1/1       Running   0          23h       172.60.0.86    kuber-master02
kube-system   kube-scheduler-kuber-master03            1/1       Running   0          23h       172.60.0.87    kuber-master03
kube-system   kubernetes-dashboard-3000605155-s5f7t    1/1       Running   0          22h       10.244.6.2     kuber-node01
```