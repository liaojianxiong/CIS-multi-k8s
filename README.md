# CIS-multi-k8s
## 项目说明
测试验证F5 CIS在多K8S集群场景下进行请求调度的能力，同时验证cilium CNI以及CIS HUB的能力。

## 部署架构
<img width="415" alt="image" src="https://github.com/liaojianxiong/CIS-multi-k8s/assets/8012953/94d5551f-150b-43da-b87c-e9e11518a87e">

部署一个F5 BIGIP VE实例对应两个K8S集群，VE以VTEP的形式接入cilium。CIS与cilium的对接可参考：https://github.com/f5devcentral/f5-ci-docs/blob/master/docs/cilium/cilium-bigip-info.rst

F5上分别配置一个与各K8S集群Node同网段的接口地址用于建立Cilium通信；另外部署业务发布网段（即VS网段），用于对外进行业务发布，供客户端访问。
每个k8s集群中各部署一个bigip-ctlr，该Pod会通过监听k8s APIServer事件的形式读取集群中的NS/Pod/Service等配置信息，并自动配置到BIGIP VE上。

## 配置要点
cilium安装  <br />
helm install cilium cilium/cilium --namespace kube-system --set hubble.relay.enabled=true --set hubble.ui.enabled=true --set prometheus.enabled=true --set operator.prometheus.enabled=true --set hubble.enabled=true --set kubeProxyReplacement=strict --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,http}" --set vtep.endpoint="55.32.170.41" --set vtep.cidr="100.64.1.1/24" --set vtep.mask="255.255.255.0" --set vtep.mac="00:50:56:3d:a9:46" --set vtep.enabled="true" --set ipam.mode="kubernetes"

这里 vtep.endpoint 是BIGIP VE underlay 接口IP, vtep.mac 是tunnel Mac 地址，在bigip上通过show net tunnels tunnel cilium-vxlan-tunnel-mp all-properties 查询

cis配置  <br />
需要增加一项"--flannel-name=cilium-vxlan-tunnel-mp” ，原因是cis复用了flannel相关代码来实现cilium overlay

## 限制说明

1.	BIGIP 的underlay接口需要与k8s集群的Node保持相同网段，这样在进行pod地址的arp广播时，node节点可以接收到并响应。
2.	两个集群中的Pod CIDR需要规划成不同网段，避免pod ip重叠导致F5上无法准确路由。

## 数据流说明

1.	K8S创建一个service，并进行对外发布。
2.	CIS监听到相关事件，写入到F5上（VIP，Pool member等内容）。
3.	客户端发起请求到VIP上，F5根据负载均衡算法或iRules规则转发到某个member上。这个member是某个集群中具体的Pod。
4.	F5需获取member ip对应的mac地址，才能够进行tunnel封装。F5在tunnel层面广播arp请求，underlay层面会发送给每个node节点，这样每个node节点都收到该请求。
5.	有该ip pod的Node节点响应此arp请求，其他Node不响应。
6.	F5接收arp响应，将member ip对应的mac地址以及node ip记录到转发表中。
7.	根据F5上路由规则以及转发表，F5封装数据包后，发送到对应Node。Node进行解封装后转给对应Pod进行处理。
8.	Node节点根据cilium vtep的设置，可以学习到VTEP endpoint is F5 vlan self-ip地址并使用这些信息进行回包vxlan封装。

## 跨集群业务配置逻辑说明
1. 非Hub模式
   在集群1上发布service，但仅发布pool，不发布vs <br />
   在集群2上发布相同的service，整体发布vs + pool + iRules调度逻辑。
   
2. Hub模式
   在集群1上发布service，仅发布pool，不发布vs <br />
   在集群2上发布相同service，仅发布pool，不发布vs <br />
   在任意一个集群上发布vs+irules调度逻辑，vs+irules的配置可以和service配置不在同一个namespace下。以方便用户根据不同组织架构进行灵活管理。

## 单集群测试项
| 序号 | 测试项 | 测试内容 | 说明 |
| -----|------ | ----------|---- |
| app001 | 基础功能测试 | 连通性测试 |1. service通过部署configmap的方式暴露到BIGIP上 <br />2. 业务访问正常 <br />3. Service的Pod动态变化可以及时反应到BIGIP上 <br />4. BIGIP上的Pool member为Pod IP，即BIGIP直达Pod。|
|  app002	 | 负载均衡算法 | 丰富的负载均衡算法  |      |                                     
|  app003 | 可定制的健康检查策略 | 定制HTTP健康检查   |  |      	                                        
|  app004	 | 空闲连接超时设置   | TCP连接超时设置     |     	                                        
|  app005	| Pod连接数限制  | Pod连接数限制 | 通过YAML可限制Pod的连接数| 
| app006 | Cookie<br />源地址会话保持 | Cookie会话保持    | 通过YAML可设置Cookie或源地址会话保持|
| app007	| 基于XFF的源地址透传 | XFF透传源地址  |通过YAML可设置XFF透传源地址 |
| app008	| 非HTTP应用源地址透传 | TCP Option透传源地址| 通过YAML可设置通过TCP Option字段透传源地址---TCP Option iRule |
| app009	|不同服务不同入口 | 不同服务不同入口 | 针对不同服务可设置不同入口 |      
|  app010	| 不同服务相同入口 | 不同服务相同入口 | 针对不同服务可设置相同入口  |        
| app011	| SSL卸载 | SSL卸载 | 通过YAML可发布HTTPS应用并配置SSL卸载  |
| app012	| 多证书绑定 | 同一入口绑定多个业务证书 | 一个入口可绑定多个域名的证书  |
| app013	| 丰富的分发策略（基于Host、基于Path、基于User-Agent等）| 基于Path进行分发 |     	                                        
| app014	| 租户隔离 | 不同namespace的应用，在BIGIP上实现配置隔离	| 不同namespace的应用，在BIGIP上实现配置隔离 |
| app015	| 连接复用 | OneConnection | 通过YAML配置OneConnection，可减少后端Pod上的连接数 |
| app016	| 高速日志输出构建流量可视化 | F5与ELK结合 | 通过F5 HSL能够将容器内应用访问流量做可视化输出，结合第三方数据平台进行数据展示---HSL iRule |

更多单集群测试项，可以参考kylin的github： https://github.com/cloudadc/cis-scripts/tree/cis-hub

## 多集群
| 序号 | 测试项 | 测试内容 | 说明 |
| -----|------ | ----------|---- |
| app101 | 流量分发| 同一个入口，按照比率将流量分发到不同k8s集群 | svc1在k8s集群1上，svc2在k8s集群2上，通过同一个入口进行发布，按照比率分发到不同的K8S集群的svc上 |
| app102 | 流量分发	| 同一个入口，基于Host分发流量 | svc1在k8s集群1上，svc2在k8s集群2上，通过同一个入口进行发布，按照Host分发到不同的K8S集群中对应的svc上 | 
| app103 | 灰度发布	| 按比例对特定User Agent进行灰度发布 |	svc1在k8s集群1上，svc2在k8s集群2上，通过同一个入口进行发布，针对特定User Agent按照比率分发到不同的K8S集群对应的svc上 |
| app104	| 灰度发布 | 根据Header对特定IP地址段进行灰度发布 | svc1在k8s集群1上，svc2在k8s集群2上，通过同一个入口进行发布，针对特定IP地址段根据不同的Header分发到不同的K8S集群对应的svc上 | 
| app105	| 灰度发布 |  根据Cookie内容进行灰度发布 | svc1在k8s集群1上，svc2在k8s集群2上，通过同一个入口进行发布，根据Cookie内容的不同分发到不同的K8S集群中 |
| app106	| 流量分发 |  同一个入口，基于Path分发流量 | svc1在k8s集群1上，svc2在k8s集群2上，通过同一个入口进行发布，根据Path不同分发到不同的K8S集群对应的svc上 |

## 多集群-Hub模式
| 序号 | 测试项 | 测试内容 | 说明 | 操作步骤
| -----|------ | ----------|---- |---- |
| app201	| 流量分发	|  同一个入口，基于Host分发流量 | svc1在k8s集群1上，svc2在k8s集群2上，通过同一个入口进行发布，根据Host不同分发到不同的K8S集群对应的svc上 |
| app202	| 流量分发	|  同一个入口，复杂分发流量 | svc1同时部署在两个k8s集群上，但svc2仅k8s集群2部署，通过同一个入口进行发布，首先根据Host匹配到svc，对于两个k8s集群同时部署的svc，再根据不同规则（比如比率）进行跨集群流量分发 |
| app203	| 流量分发	|  同一个入口，智能分发流量 | svc1、svc3、svc4同时部署在两个k8s集群上，但svc2仅k8s集群2部署，通过同一个入口进行发布，首先根据Host匹配到svc，对于两个k8s集群同时部署的svc，svc1根据比率进行分发,svc3根据cookie内容进行分发，svc4根据header字段值进行分发 |


## 扩容测试
| 序号 | 测试项 | 测试内容 | 说明 |
| -----|------ | ----------|---- |
| app301 | CIS扩容 | 按需扩容CIS | 原本一个CIS监听所有namespace（假设为ns1和ns2），把配置写入到bigip上。新增一个CIS实例监听ns2，原CIS实例监听ns1，配置写入相同bigip上 |
| app302 | BIGIP扩容 | 按需扩容BIGIP | 原本两个CIS各监听一个namespace（假设CIS1监听ns1，CIS2监听ns2），把配置写入到同一个bigip上。扩容部署bigip2，调整cis2 配置写入到bigip2上 |

## 配置容量测试
| 序号 | 测试项 | 测试内容 | 说明 |
| -----|------ | ----------|---- |
| app402	| VS的配置容量   | 验证VS的配置容量 |	在N条记录基础上，增加1条，验证CIS是否能够在可接受的时间内发布到bigip上 |
| app401 | datagroup的配置容量	| 验证datagroup的配置容量	| 在N条记录基础上，增加1条，验证CIS是否能够在可接受的时间内发布到bigip上 |
| app403	| Pool的配置容量 | 验证Pool的配置容量 | 在N条记录基础上，增加1条，验证CIS是否能够在可接受的时间内发布到bigip上 |

## 高可用测试
| 序号 | 测试项 | 测试内容 | 说明 |
| -----|------ | ----------|---- |
| app501	| F5 HA Failover | 	验证VPC里面F5设备的主备切换 	  |   验证VPC里面F5设备的主备切换      |
| app502 |	CIS Controller高可用 |	CIS Controller高可用 |	验证Controller Pod故障后是否对业务有影响 |



