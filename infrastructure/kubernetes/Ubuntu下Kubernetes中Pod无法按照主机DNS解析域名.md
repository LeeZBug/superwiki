### 问题
节点上原先配置了公网DNS，Pod中调用的服务在内外网都有对应的解析(但是通过外网会经过waf)，但更新/etc/resolv.conf中的nameserver为内网DNS后，服务依然调用是通过外网。

### 排查
1. 检查Pod的dnsPolicy，发现使用的是默认的ClusterFirst，将解析发送到了CoreDNS
&emsp;&emsp;dnsPolicy 是 Kubernetes Pod 规范中的一个字段，用于控制 Pod 的 DNS 解析行为。它决定了 Pod 如何继承和使用 DNS 配置。

|dnsPolicy选项|说明|特点|
|-|-|-|
|ClusterFirst(默认)|Pod使用集群的DNS服务(通常是CoreDNS或kube-dns)|集群内部域名可解析，外部域名查询会被转发到上游 DNS 服务器|
|Default|从运行Pod的节点继承DNS配置|Pod 使用节点/etc/resolv.conf中的配置，不会自动解析集群服务域名|
|ClusterFirstWithHostNet|当Pod使用hostNetwork: true时，仍优先使用集群 DNS|适用于需要主机网络但仍需访问集群服务的 Pod，结合了 hostNetwork 和 ClusterFirst 的特性|
|None|不设置任何 DNS 策略，必须通过 dnsConfig 字段显式指定|灵活但是必须指定dnsConfig字段|

~~2. 检查coredns pod的配置，未发现异常。但是发现coredns pod使用的dnsPolicy是Default，但是在debug pod的时候发现不管怎么修改/etc/resolv.conf的时候，coredns pod里的/etc/resolv.conf都还是使用公网DNS的
![](https://cdn.jsdelivr.net/gh/LeeZBug/imageRepository/img/6e817a231f22fee4ee8e0bcf6249879b.png)~~
```
# debug pod的技巧
kubectl debug -n kube-system coredns-76f75df574-lk5b9 -it --image=harbor.supcon.com/base-images/busybox:latest -- cat /etc/resolv.conf
```

3. 寻找文档发现，因为ubuntu默认会有个systemd-resolved.service，在Ubuntu配置有systemd-resolved.service的情况下，kubeadm默认安装集群时，会指定kubelet的--resolv-conf为/run/systemd/resolve/resolv.conf。
![](https://cdn.jsdelivr.net/gh/LeeZBug/imageRepository/img/1734a320e91003063bfd6033161da04e.png)
![](https://cdn.jsdelivr.net/gh/LeeZBug/imageRepository/img/20250612150811.png)
   该参数会控制：
    -  kubelet 从哪里读取 DNS 配置来生成 Pod 的 /etc/resolv.conf
    - 影响 Pod 的 DNS 解析行为（特别是当 dnsPolicy 设置为 ClusterFirst 或 Default 时）

    当 dnsPolicy 为：
     - ClusterFirst：kubelet 会基于该文件内容生成 Pod 的 DNS 配置
     - Default：Pod 直接继承该文件内容
因为这个配置的原因，导致dnsPolicy为Default的coredns pod的/etc/resolv.conf一直使用的/run/systemd/resolve/resolv.conf，修改节点的/etc/resolv.conf是没有用的。
## 解决
修改/run/systemd/resolve/resolv.conf文件的时候，发现只是上写着不要修改该文件，发现/run/systemd/resolve/resolv.conf文件由systemd-resolved服务动态生成
![](https://cdn.jsdelivr.net/gh/LeeZBug/imageRepository/img/db7965229610b87c629d2fec86bbd8ad.png)
实际需要修改该配置文件/etc/systemd/resolved.conf，重启systemd-resolv服务
![](https://cdn.jsdelivr.net/gh/LeeZBug/imageRepository/img/9bdaea714276a01701bd837ba7a9cd87.png)
修改后重启pod问题解决