# CIS-multi-k8s
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

## 多集群
| 序号 | 测试项 | 测试内容 | 说明 |
| -----|------ | ----------|---- |
| app101 | 流量分发| 同一个入口，按照比率将流量分发到不同k8s集群 | svc1在k8s集群1上，svc2在k8s集群2上，通过同一个入口进行发布，按照比率分发到不同的K8S集群中 |
| app102 | 流量分发	| 同一个入口，基于Host分发海量svc | svc1在k8s集群1上，svc2在k8s集群2上，通过同一个入口进行发布，按照Host分发到不同的K8S集群中 | 
| app103 | 灰度发布	| 按比例对特定User Agent进行灰度发布 |	svc1在k8s集群1上，svc2在k8s集群2上，通过同一个入口进行发布，针对特定User Agent按照比率分发到不同的K8S集群中 |
|  app104	| 灰度发布 | 根据Header对特定IP地址段进行灰度发布 | svc1在k8s集群1上，svc2在k8s集群2上，通过同一个入口进行发布，针对特定IP地址段根据不同的Header分发到不同的K8S集群中 | 
| app105	| 灰度发布 |  根据Cookie内容进行灰度发布 | svc1在k8s集群1上，svc2在k8s集群2上，通过同一个入口进行发布，根据Cookie内容的不同分发到不同的K8S集群中 |

## 扩容测试
| 序号 | 测试项 | 测试内容 | 说明 |
| -----|------ | ----------|---- |
| app201 | 基于namespace扩容 | 基于namespace扩容 | 不同namespace配置到不同的bigip上，实现bigip的扩容 |

## 配置容量测试
| 序号 | 测试项 | 测试内容 | 说明 |
| -----|------ | ----------|---- |
| app301 | datagroup的配置容量	| 验证datagroup的配置容量	| 在N条记录基础上，增加1条，验证CIS是否能够在可接受的时间内发布到bigip上 |
| app302	| VS的配置容量   | 验证VS的配置容量 |	在N条记录基础上，增加1条，验证CIS是否能够在可接受的时间内发布到bigip上 |
| app303	| Pool的配置容量 | 验证Pool的配置容量 | 在N条记录基础上，增加1条，验证CIS是否能够在可接受的时间内发布到bigip上 |

## 高可用测试
| 序号 | 测试项 | 测试内容 | 说明 |
| -----|------ | ----------|---- |
|  app401	| F5 HA Failover | 	验证VPC里面F5设备的主备切换 	  |   验证VPC里面F5设备的主备切换      |
|  app402 |	CIS Controller高可用 |	CIS Controller高可用 |	验证Controller Pod故障后是否对业务有影响 |



