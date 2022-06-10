[TOC]

# 监控方法论

## 云原生时代的监控系统

- 指标监控（Metrics）：随时间产生的一些与监控相关的可聚合数据点。如，Prometheus。
- 日志监控（Logging）：离散式的日志或事件。如，ElasticStack，PLG Stack。
- 链路跟踪（Tracing）：分布式应用调用链跟踪。如，Zipkin，Jaeger，SkyWalking，Pinpoint。

## 监控方法论

### 黄金指标

Google的四个黄金指标，用于在服务级别帮助衡量终端用户体验。适用于服务监控。

- 延迟，Latency

  服务请求所需要的时长，需要区分失败请求和成功请求。例如，http请求平均延迟。

- 流量，Traffic

  衡量服务的容量需求。例如，app的qps，数据库系统的tps。

- 错误，Errors

  请求失败的速率，用于评估错误发生的情况。例如，http返回码500的显示错误，返回错误内容或无效内容的隐式错误。

- 饱和度，Saturation

  app的资源使用情况。例如，内存，cpu，i/o，磁盘等资源的使用量。

### USE方法

Netfix的USE方法，Utilization Saturation and Errors Method，主要用于分析系统性能问题，可以指导用户快速识别资源瓶颈以及错误的方法。应用于主机指标监控。

- 使用率，Utilization

  资源被有效利用起来提供服务的平均时间占比。

- 饱和度，Saturation

  资源的拥挤程度。例如，cpu工作队列的长度，也即cpu平均负载。

- 错误，Errors

  错误的数量。

### RED方法

Weave Cloud基于Google的四个黄金指标，结合Prometheus以及Kubernetes，推出的适合于云原生应用以及微服务架构应用的监控方法论。适用于服务监控。

- 每秒请求数量，Rate
- 每秒错误数量，Errors
- 服务响应时间，Duration

## whiteBox，blackBox

- whiteBox，**通过白盒能够了解其内部的实际运行状态，通过对监控指标的观察能够预判可能出现的问题，**从而对潜在的不确定因素进行优化。
- blackBox，黑盒监控即**以用户的身份测试服务的外部可见性**，常见的黑盒监控包括 HTTP探针、 TCP探针 等用于检测站点或者服务的可访问性，以及访问效率等。

黑盒监控相较于白盒监控最大的不同在于：黑盒监控是以故障为导向，当故障发生时，黑盒监控能快速发现故障。而白盒监控则侧重于主动发现或者预测潜在的问题。

一个完善的监控目标是要能够从白盒的角度发现潜在问题，能够在黑盒的角度快速发现已经发生的问题。



# Prometheus

Prometheus不仅仅是一款时序数据库（TSDB），更是一款设计用于目标（包括主机，应用等）监控的关键组件。结合其他组件，例如Pushgateway，Altermanager，Grafana等，可以构成完整的监控系统。

## 架构

prometheus只负责时序型指标数据的采集和存储。数据的分析聚合，可视化，告警等功能由其他组件完成。

<img src="https://github.com/NieGuanglin/docs/blob/main/pics/CNCF/prometheus/1prom-架构.png">

<img src="/Users/nieguanglin/docs/pics/CNCF/prometheus/1prom-架构.png" alt="1prom-架构.png" style="zoom:100%;" />

- Prometheus Server

收集和存储时间序列数据。

- Pushgateway

接收由短期作业生成的指标数据，并支持由Prometheus Server拉取。

- Exporters

对于不支持Instrumentation的app，用于暴露现有app的指标给Prometheus Server。

Instrumentation是指app内部本身就有支持prom的数据格式。

- AlterManager

从Prometheus Server接收到告警通知后，向用户告警。

- Data Visualization

内建Web UI，Grafana，以及API。

- Service Discovery

动态发现待监控的Target，从而完成监控配置的组件。在kubernetes环境中很有用。



## job与instance

能够接收Prometheus Server进行scrape操作的每个端点(endpoint)，即为一个instance。

具有类似功能的instance的集合，即为一个job。

（能让prom找到instance的入口，即为target。比如：被监控主机ip+port就是一个target。或者服务发现target可以发现多个instance。）



## 数据

### 数据来源

prometheus主动从各Target拉取数据，而非等待被监控对象的推送。pull的原因在于，它的目标是收集在Target上预先完成聚合的数据，而不是作为一个由事件驱动的存储系统。而且，这样有利于集中控制，可以把配置集中在prometheus上。

- Exporters，属于whiteBox，需要加一个附加组件，采集监控指标并格式化。比如，node exporter。
- Instrumentation，属于blackBox，内建了metrics api，直接兼容prometheus的监控指标，无需侵入。比如，k8s的kubelet等组件。
- Pushgateway，短期任务或批处理任务，它会将指标push给pushgateway，prometheus再从pushgateway拉取。

#### 基于文件的服务发现

基于文件的服务发现略优于静态配置的服务发现方式，它不依赖其他组件，因此比较简单。prom定期从文件中加载target信息。文件可以使用json和yaml格式。这些文件可以通过另外一个系统生成，比如，ansible运维工具，由脚本基于CMDB定期查询生成。

```yaml
# targets的配置
- targets:
  - xxx.xx.xx.xx:9090
  labels:
    app: prometheus
    job: prometheus
---
# prom的配置
...
scrape_configs:
  - job_name: "prometheus"
    file_sd_configs:
    # 指定要加载的文件列表
    - files:
      - targes/prometheus*.yaml
      # 加载文件的间隔
      refresh_interval: 2m

  - job_name: "nodes"
    file_sd_configs:
    # 指定要加载的文件列表
    - files:
      - targes/node*.yaml
      # 加载文件的间隔
      refresh_interval: 2m
```

#### 基于k8s的服务发现

k8s的node，service，endpoint，pod和ingress资源分别由各自的发现机制进行定义。

负责发现每种类型资源对象的组件，在prom中称为role，与k8s的rbac的role毫无关系。

prom使用daemonset部署node-exporter，发现各节点。

在k8s的kube-system下，prom server体现为一个statefulset。

指标的relabel过程：

<img src="https://github.com/NieGuanglin/docs/blob/main/pics/CNCF/prometheus/2prom-重新打标.png">

<img src="/Users/nieguanglin/docs/pics/CNCF/prometheus/2prom-重新打标.png" alt="2prom-重新打标" style="zoom:100%;" />

指标的重新打标，是为了：

删除不需要的指标；编辑标签，避免泄露敏感信息。

```yaml
global:
  evaluation_interval: 1m
  scrape_interval: 60s
  external_labels:
    app_id: default
    cluster_display_name: global_cluster
    cluster_id: global_cluster
    prometheus: kube-system/k8s
    prometheus_replica: prometheus-k8s-0
rule_files:
- <通过configmap挂载进pod的路径>/*.yaml
scrape_configs:
- job_name: avalanche/jobA
  honor_labels: false
  kubernetes_sd_configs:
  - role: endpoints
    namespaces:
      names:
      - avalanche
  scrape_interval: 30s
  scheme: http
  relabel_configs:
  - action: keep
    source_labels:
    - __meta_kubernetes_service_label_app
    regex: avalanche
  - source_labels:
    - __meta_kubernetes_endpoint_address_target_kind
    - __meta_kubernetes_endpoint_address_target_name
    separator: ;
    regex: Node;(.*)
    replacement: ${1}
    target_label: node
  - source_labels:
    - __meta_kubernetes_namespace
    target_label: namespace

- job_name: kube-system/monitor-blackbox-exporter/0
  honor_labels: false
  kubernetes_sd_configs:
  - role: endpoints
    namespaces:
      names:
      - kube-system
  scrape_interval: 3s
  metrics_path: /probe
  params:
    module:
    - tcp_connect
    target:
    - kong-proxy.kong:80
    - kong-proxy.kong:443
  scheme: http
  relabel_configs: 
    ...

- job_name: kubernetes-nodes
  scrape_timeout: 60s
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/srviceaccount/ca.crt
    insecure_skip_verify: true
  bearer_token_file: /var/run/secrets/kubernetes.io/srviceaccount/token
  kubernetes_sd_configs:
  - role: node
  # 标签的relabel
  relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
  - source_labels:
    - __meta_kubernetes_node_annotation_device_type
    target_label: device_type
  # 标签的metric_relabel
  metric_relabel_configs:
  - source_labels:
    - __name__
    regex: kubelet_running_pod_count|kubelet_volume_stats_used_bytes
    action: keep
  - regex: (__name__|instance)  
    action: labelkeep
  - source_label:
    - __name__
    target_label: node_role
    replacement: Node
  - regex: instance|node_role_kubernetes_io_master
    action: labeldrop
    ...
 ...
 
 alerting:
   alert_relabel_configs:
   - action: labeldrop
     regex: prometheus_replica
   alertmanagers:
   - path_prefix: /
     scheme: http
     # 基于k8s的服务发现，指定alertmanager
     kubernetes_sd_configs:
     - role: endpoints
       namespaces:
         names:
         - kube-system
     relabel_configs:
     - action: keep
       source_labels:
       - __meta_kubernetes_service_name
       regex: alertmanager
     - action: keep
       source_labels:
       - __meta_kubernetes_endpoint_port_name
       regex: http
```

### 指标定义

prom自身的指标，可以通过访问prom server的/metrics接口，可以获取部分指标。

在k8s里，还有些指标定义在configMap中，一般通过关键字“rule”可以查到。

### 指标类型（metric type）

仅以**键值**形式存储**时序型**聚合数据。键会使用**标签**作为元数据，标签可以用于过滤数据。值即为双精度浮点型。

prom基于指标名称（metric），以及附属的标签集（labelset），来唯一定义一条时间序列（series）。

#### Counter

计数器，单调递增。不能为负，但可以置零。

counter的总数通常没什么用，需要借助rate，topk，increase，irate等函数生成增长率。

- rate(http_requests_total[2h])，2h内，该指标各时间序列的增长速率。
- topk(3, http_requests_total)，该指标排名前3的时间序列。
- irate(http_requests_total[2h])，高灵敏度函数，用于计算指标的瞬时速率。基于样本范围内的最后两个样本进行计算，与rate对比，irate更适合于短期内的变化率分析。

#### Gauge

仪表盘，有增减的指标数据。

常用来求和，取平均值，最大值，最小值等聚合计算。经常结合predict_linear，delta函数使用。

- predict_linear(v range-vector, t, scalar)函数可以预测时间序列v在t秒后的值，它通过线性回归的方式来预测样本数据的变化趋势。

- delta(v range-vector)函数计算范围向量中每个时间序列的第一个值与最后一个值的差。

  delta(cpu_temp_celsisu{host="x.x.x.x"}[2h])，该host的cpu温度与2小时前的温度的差值。

#### Histogram

直方图。**在一段时间范围内对数据进行采样，并将其计入bucket中。存储样本值在各个bucket中的数量，**所有样本值之和，总的样本数量等信息。

它**可以分析，因为异常值而引起的平均值不准的问题**。它不直接保存分位数，而是使用histogram_quantile函数计算分位数（即，某个bucket的样本数在所有样本数中占据的比例）。

注意，取值间隔的划分采用的是累积区间间隔机制。每个bucket的样本，都包含了前面所有bucket中的样本。

histogram类型的指标都有一个基础指标名称\<basename>。

- \<basename>_bucket{le="\<upper inclusive bound>"}，桶上限，也即样本统计区间。
- \<basename>_bucket{le="+lnf"}，最大区间的名称。
- \<basename>_sum，所有样本的总和。
- \<basename>_count，总观测次数。

#### Summary

摘要。Histogram的扩展，但它是由被监控端自行聚合计算并上报分位数。



## PromQL

Prometheus内置的查询语句，并内置了用于数据处理的函数。

### 数据类型

表达式的返回值，有4种数据类型

- **instance vector**，特定时间序列集合上，具有**相同时间戳**的一组样本值。
- **range vector**，特定时间序列集合上，在指定的**同一段时间内**，所有的样本值。
- scalar，一个浮点型数据值。
- string，使用单引号，双引号，反引号引用。反引号不会对转义字符转义。

### 标签的标签匹配符

- =，完全相等。
- !=，不等。
- =~，正则匹配。正则表达式需要匹配指定的标签的整个值。
- !~，正则不匹配。

注意

- 特别的，使用\__name__作为标签名称，还能够对指标名称进行过滤。

- 范围向量选择器与即时向量选择器相比，唯一区别是：它需要在表达式后紧跟一个[]，表达时间范围。

- 默认情况下，即时向量选择器和范围向量选择器都以当前时间为基准时间点，而offset关键字可以修改该基准。

  如，http_requests_total offset 5m，http_requests_total[5m] offset 1d。

### 操作数的运算

- 操作数类型

  - 两个标量。

  - 即时向量与标量。

  - 即时向量与即时向量。可以基于向量匹配模式定义运算机制，它又分为一对一，一对多/多对一。

    - 一对一匹配。

      ```sh
      <vector expr> <bin-op> ignoring(<label list>) <vector expr>
      <vector expr> <bin-op> on(<label list>) <vector expr>
      ```

      举例：

      **method_code:http_errors**:rate5m{code="500"} / ignoring(code) **method:http_requests**:rate5m

      左侧：
      {method="get", code="500"}                    
      {method="post", code="500"}
      右侧：
      {method="get"} //包含各种状态码
      {method="del"}
      {method="post"}
      找到两边标签完全相同的一对参数，运算。**左侧标签集的一个，对应右侧标签集的一个，也即一对一。**
      返回的是：返回码500的各类请求的rate，占各类请求的rate的比例。

    - 一对多/多对一。

      ```sh
      // 左侧为多
      <vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
      <vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
      // 右侧为多
      <vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
      <vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>
      ```

      举例：

      **method_code:http_errors**:rate5m / ignoring(code) group_left **method:http_requests**:rate5m

      左侧，也即多侧：
      {method="get", code="500"} 
      {method="get", code="404"}
      {method="put", code="501"}
      {method="post", code="500"}
      {method="post", code="404"}
      右侧，也即一侧：
      {method="get"}
      {method="del"}
      {method="post"}
      找到两边标签完全相同的一对参数，运算。**左侧标签集的多个，对应右侧标签集的一个，也即多对一。**
      返回的是：各类返回码的各类请求的rate，占各类请求的rate的比例。

- 算数运算符

  +，-，*，/，%（取模），^（幂运算）

- 比较运算

  ==，!=，>，<，>=，<=

- 逻辑运算

  and，or，unless。

  它们只允许在两个即时向量间进行，不支持标量参与运算。

- 顺序

  先乘除，再加减；再比较；再逻辑。可以使用括号改变顺序。

### 聚合函数

聚合函数只支持单个即时向量，返回值可能是新向量或标量。

表达式

- $$
  <aggr-op>([parameter,]<vector expression>)[without|by(<label list>)]
  $$

- $$
  <aggr-op>[without|by(<label list>)]([parameter,]<vector expression>)
  $$

without与by

- without，结果向量中排除由without指定的标签，没指定的标签作为分组依据。
- by，结果向量中使用by指定的标签，没指定的标签则忽略。by需要显式指出job，instance一类的标签。

聚合函数

- sum，样本值求和。
- avg，样本值求平均。
- count，分组内时间序列统计数量。
- stddev，样本值求标准差，用于了解数据的波动程度。
- stdvar，样本值求方差，是求标准差的中间状态。
- min，样本值中最小值。
- max，样本值中最大值。
- topk，分组内样本值最大的前k个时间序列及其值。
- bottomk，分组内样本值最小的前k个时间序列及其值。
- quantile，分位数用于评估数据的分布状态，它会返回数值落在<=指定分位区间的比例。
- count_values，分组内时间序列样本值进行数量统计。



# Thanos

默认的单机Prometheus配置难以满足一些需求，比如，通过单个API进行跨分布式Prometheus查询，以及合并多个Prometheus数据。Thanos提供了一系列组件，通过跨集群联合，跨集群无限存储和全局查询实现了Prometheus的高可用。

## 架构

- Sidecar

Sidecar与原生的prometheus server部署在同一个pod内，Sidecar既是一个proxy组件，代理了对prometheus server的访问；又是一个agent组件，上传数据到云端存储。

**使用Sidercar模式部署时，会使用到它。**

- Store Gateway

Store Gateway实现了一套和Sidecar一致的API，提供给Query用于查询在云端对象存储的数据。因为Sidecar在完成数据备份后或者远程写入Receiver后，prom会清理本地数据保证本地空间可用。所以当监控人员需要查询历史数据的时候，Store Gateway就提供了一个查询接口。

Store Gateway只会缓存对象存储的基本信息，例如存储块的索引，从而实现快速查询的同时占用本地较少空间。

- Compactor

Compactor对采集到的数据进行压缩，节省对象存储的空间。

- Receiver

Receiver接收来自于prom server的remote-write WAL。对外暴露API，并且上传数据到云端对象存储。

**使用Receiver模式部署时，会使用到它。**

- Ruler

Ruler统一管理多个AlertManager的告警规则配置。

- Query

Query实现了一套prom官方的HTTP API，从而保证对外提供与prom一致的数据源接口。客户端可以通过Query，使用同一个查询接口请求到不同集群的数据。

Query本身可以水平扩展，因而可以实现高可用部署。并且，它还可以对高可用部署的prom的数据进行合并，保证查询一致性。从而解决全局视图和高可用的问题。

## 部署方式

### Sidecar模式

使用sidecar容器代理查询接口，并上传数据。不使用receiver组件。

<img src="https://github.com/NieGuanglin/docs/blob/main/pics/CNCF/prometheus/3prom-thanos-sidecar-架构.png">

<img src="/Users/nieguanglin/docs/pics/CNCF/prometheus/3prom-thanos-sidecar-架构.png" alt="3prom-thanos-sidecar-架构" style="zoom:100%;" />

### Receive模式

prom server远程写入到receiver，receiver代理查询接口，并上传数据。不使用sidecar组件。

<img src="https://github.com/NieGuanglin/docs/blob/main/pics/CNCF/prometheus/4prom-thanos-receive-架构.png">

<img src="/Users/nieguanglin/docs/pics/CNCF/prometheus/4prom-thanos-receive-架构.png" alt="3prom-thanos-receive-架构" style="zoom:100%;" />