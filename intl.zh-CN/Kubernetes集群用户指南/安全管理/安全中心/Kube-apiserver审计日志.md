---
keyword: [手动开通, 审计报表, 集群审计, 关闭审计]
---

# Kube-apiserver审计日志

在Kubernetes集群中，API Server的审计日志可以帮助集群管理人员记录或追溯不同用户的日常操作，是集群安全运维中重要的环节。本文旨在帮助您了解阿里云Kubernetes集群API Server审计日志的相关配置，如何通过日志服务收集、分析审计日志，并根据您的需求为审计日志设置自定义的告警规则，以及如何关闭集群审计功能。

## 配置介绍

当前创建Kubernetes集群时会默认选中**使用日志服务**，开启API Server审计功能，相关的参数配置功能如下。

**说明：** 登录到Master节点，API Server配置文件的目录是/etc/kubernetes/manifests/kube-apiserver.yaml。

|配置|说明|
|--|--|
|audit-log-maxbackup|审计日志最大分片存储10个日志文件。|
|audit-log-maxsize|单个审计日志最大内存为100 MB。|
|audit-log-path|审计日志输出路径为/var/log/kubernetes/kubernetes.audit。|
|audit-log-maxage|审计日志最多保存期为7天。|
|audit-policy-file|审计日志配置策略文件，文件路径为：/etc/kubernetes/audit-policy.yml。|

登录Master节点机器，审计配置策略文件的目录是/etc/kubernetes/audit-policy.yml，内容如下。

```
apiVersion: audit.k8s.io/v1beta1 # This is required.
kind: Policy
# Don't generate audit events for all requests in RequestReceived stage.
omitStages:
  - "RequestReceived"
rules:
  # The following requests were manually identified as high-volume and low-risk,
  # so drop them.
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
      - group: "" # core
        resources: ["endpoints", "services"]
  - level: None
    users: ["system:unsecured"]
    namespaces: ["kube-system"]
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["configmaps"]
  - level: None
    users: ["kubelet"] # legacy kubelet identity
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["nodes"]
  - level: None
    userGroups: ["system:nodes"]
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["nodes"]
  - level: None
    users:
      - system:kube-controller-manager
      - system:kube-scheduler
      - system:serviceaccount:kube-system:endpoint-controller
    verbs: ["get", "update"]
    namespaces: ["kube-system"]
    resources:
      - group: "" # core
        resources: ["endpoints"]
  - level: None
    users: ["system:apiserver"]
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["namespaces"]
  # Don't log these read-only URLs.
  - level: None
    nonResourceURLs:
      - /healthz*
      - /version
      - /swagger*
  # Don't log events requests.
  - level: None
    resources:
      - group: "" # core
        resources: ["events"]
  # Secrets, ConfigMaps, and TokenReviews can contain sensitive & binary data,
  # so only log at the Metadata level.
  - level: Metadata
    resources:
      - group: "" # core
        resources: ["secrets", "configmaps"]
      - group: authentication.k8s.io
        resources: ["tokenreviews"]
  # Get repsonses can be large; skip them.
  - level: Request
    verbs: ["get", "list", "watch"]
    resources:
      - group: "" # core
      - group: "admissionregistration.k8s.io"
      - group: "apps"
      - group: "authentication.k8s.io"
      - group: "authorization.k8s.io"
      - group: "autoscaling"
      - group: "batch"
      - group: "certificates.k8s.io"
      - group: "extensions"
      - group: "networking.k8s.io"
      - group: "policy"
      - group: "rbac.authorization.k8s.io"
      - group: "settings.k8s.io"
      - group: "storage.k8s.io"
  # Default level for known APIs
  - level: RequestResponse
    resources:
      - group: "" # core
      - group: "admissionregistration.k8s.io"
      - group: "apps"
      - group: "authentication.k8s.io"
      - group: "authorization.k8s.io"
      - group: "autoscaling"
      - group: "batch"
      - group: "certificates.k8s.io"
      - group: "extensions"
      - group: "networking.k8s.io"
      - group: "policy"
      - group: "rbac.authorization.k8s.io"
      - group: "settings.k8s.io"
      - group: "storage.k8s.io"
  # Default level for all other requests.
  - level: Metadata
```

**说明：**

-   在收到请求后不立即记录日志，当返回体Header发送后才开始记录。
-   对于大量冗余的kube-proxy watch请求，kubelet和system:nodes对于节点的get请求，kube组件在kube-system下对于endpoint的操作，以及API Server对于Namespaces的get请求等不作审计。
-   对于/healthz\*，/version\*/swagger\*等只读URL不作审计。
-   对于可能包含敏感信息或二进制文件的Secrets，ConfigMaps，tokenreviews接口的日志等级设为metadata，该level只记录请求事件的用户、时间戳、请求资源和动作，而不包含请求体和返回体。
-   对于一些如authenticatioin、rbac、certificates、autoscaling、storage等敏感接口，根据读写记录相应的请求体和返回体。

## 查看审计报表

容器服务Kubernetes版内置了3个审计日志报表。通过报表，您可以获取以下内容：

-   所有用户以及系统组件对集群执行的重要操作。
-   操作的源地址、源地址所属区域以及分布。
-   各类资源的详细操作列表。
-   子账号操作详细列表。
-   重要操作（登录容器、访问保密字典、删除资源等）的详细列表。

**说明：**

-   创建集群时候，默认选中使用日志服务，即会自动开通审计报表相关功能。日志服务的计费详情请参见[计费说明](#section_pq5_y0q_dgd)。若未开通，请参见[手动开通审计报表](#section_olz_426_8mm)。
-   请不要修改审计报表。如果您有自定义审计报表的需求，请在[日志服务管理控制台](https://sls.console.aliyun.com/)创建新的报表。

您可以通过以下两种方式访问审计报表：

-   登录[容器服务管理控制台](https://cs.console.aliyun.com)，在左侧导航栏单击**集群**，在集群列表目标集群的**操作**列中，单击**更多** \> **集群审计**。
-   登录[容器服务管理控制台](https://cs.console.aliyun.com)，在左侧导航栏单击**集群**，在集群列表中单击集群名称，进入到集群信息页面。在集群管理页左侧导航栏中，单击**安全管理** \> **集群审计**。

## 审计报表说明

API Server审计共3个报表。分别是：审计中心概览、资源操作概览以及资源操作详细列表。

-   审计中心概览

    审计中心概览展示Kubernetes集群中的事件整体概览以及重要事件（公网访问、命令执行、删除资源、访问保密字典等）的详细信息。

    ![审计中心概览](https://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/2295659951/p133068.png)

    **说明：** 在该报表中，默认显示一周的统计信息。您可以自定义选择统计时间范围。此外，该报表支持指定Namespace、子账号ID、状态码进行筛选。您可以选择任一一项或多项组合筛选指定范围的事件。

-   资源操作概览

    资源操作概览展示Kubernetes集群中常见的计算资源、网络资源以及存储资源的操作统计信息。操作包括创建、更新、删除、访问。其中：

    -   计算资源包括：Deployment、StatefulSet、CronJob、DaemonSet、Job、Pod。
    -   网络资源包括：Service、Ingress。
    -   存储资源包括：ConfigMap、Secret、PersistentVolumeClaim。
    ![资源操作概览](https://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/2295659951/p133069.png)

    **说明：**

    -   在该报表中，默认显示一周的统计信息。您可以自定义选择统计时间范围。此外，该报表支持指定Namespace、子账号ID进行筛选。您可以选择任一一项或多项组合筛选指定范围的事件。
    -   若您需要查看对应资源的详细操作事件，请使用资源操作详细列表。
-   资源操作详细列表

    该报表用于展示Kubernetes集群中某类资源的详细操作列表。您需要选择或输入指定的资源类型进行实时查询。该报表会显示：资源操作各类事件的总数、Namespace分布、成功率、时序趋势以及详细操作列表等。

    ![资源操作详细列表](https://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/3295659951/p133070.png)

    **说明：**

    -   若您需要查看Kubernetes中注册的CRD（CustomResourceDefinition）资源或列表中没有列举的其他资源，可以手动输入资源名的复数形式。例如CRD资源为AliyunLogConfig，则输入AliyunLogConfigs。
    -   在该报表中，默认显示一周的统计信息。您可以自定义选择统计时间范围。此外，该报表支持指定Namespace、子账号ID、状态码进行筛选。您可以选择任一一项或多项组合筛选指定范围的事件。

## 查看详细日志记录

如果您有自定义查询、分析审计日志的需求，可以进入日志服务管理控制台查看详细的日志记录。

**说明：** 集群审计日志在日志服务中对应的日志库数据默认保存时间为30天，如果您需要修改日志的默认保存时间，请参见[管理Logstore](/intl.zh-CN/数据采集/准备工作/管理Logstore.md)。

1.  登录[日志服务控制台](https://sls.console.aliyun.com/#/sls)。

2.  选择创建集群时设置的日志Project，单击名称进入日志Project页面。

3.  选择名称为audit-$\{clustered\}的日志库（logstore），单击![查询分析图标](https://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/3295659951/p75381.png)，选择**查询分析**，查看对应的审计日志。

    **说明：**

    -   在集群创建过程中，指定的日志Project中会自动添加一个名为`audit-${clustereid}`的日志库。
    -   审计日志的Logstore默认已经配置好索引。请不要修改索引，以免报表失效。

常见的审计日志搜索方式如下：

-   查询某一子账号的操作记录，直接输入子账号ID，单击**查询/分析**。
-   查询某一资源的操作，直接输入资源名，单击**查询/分析**。
-   过滤掉系统组件的操作，输入`NOT user.username: node NOT user.username: serviceaccount NOT user.username: apiserver NOT user.username: kube-scheduler NOT user.username: kube-controller-manager`，单击**查询/分析**。

更多查询、统计方式，请参见[日志服务查询分析方法](/intl.zh-CN/查询与分析/查询简介.md)。

## 设置告警

若您需要对某些资源的操作进行实时告警，可以通过日志服务的告警功能实现。告警方式支持钉钉机器人、自定义WebHook和通知中心。详细操作方式请参见[日志服务告警配置](/intl.zh-CN/可视化与告警/告警/简介.md)。

**说明：** 对于审计日志的更多查询方式，可以参考审计报表中的查询语句。操作方式为：在日志服务Project详情页中，单击左侧导航栏中的**仪表盘**，进入指定的仪表盘（报表），展开指定分析图表右上角的折叠列表，并单击**查看分析详情**。

示例1：对容器执行命令时告警

某公司对于Kubernetes集群使用有严格限制，不允许用户登录容器或对容器执行命令，如果有用户执行命令时需要立即给出告警，并希望告警时能够显示用户登录的具体容器、执行的命令、操作人、事件ID、时间、操作源IP等信息。

-   查询语句为：

    ```
    verb : create and objectRef.subresource:exec and stage:  ResponseStarted | SELECT auditID as "事件ID", date_format(from_unixtime(__time__), '%Y-%m-%d %T' ) as "操作时间",  regexp_extract("requestURI", '([^\?]*)/exec\?.*', 1)as "资源",  regexp_extract("requestURI", '\?(.*)', 1)as "命令" ,"responseStatus.code" as "状态码",
     CASE 
     WHEN "user.username" != 'kubernetes-admin' then "user.username"
     WHEN "user.username" = 'kubernetes-admin' and regexp_like("annotations.authorization.k8s.io/reason", 'RoleBinding') then regexp_extract("annotations.authorization.k8s.io/reason", ' to User "(\w+)"', 1)
     ELSE 'kubernetes-admin' END  
     as "操作账号", 
    CASE WHEN json_array_length(sourceIPs) = 1 then json_format(json_array_get(sourceIPs, 0)) ELSE  sourceIPs END
    as "源地址" limit 100
    ```

-   条件表达式为：`操作事件 =~ ".*"`。

示例2：APIServer公网访问失败告警

某集群开启了公网访问，为防止恶意攻击，需要监控公网访问的次数以及失败率，若访问次数到达一定阈值（10次）且失败率高于一定阈值（50%）则立即告警，并希望告警时能够显示用户的IP所属区域、操作源IP、是否高危IP等信息。

-   查询语句为：

    ```
    * | select ip as "源地址", total as "访问次数", round(rate * 100, 2) as "失败率%", failCount as "非法访问次数", CASE when security_check_ip(ip) = 1 then 'yes' else 'no' end  as "是否高危IP",  ip_to_country(ip) as "国家", ip_to_province(ip) as "省", ip_to_city(ip) as "市", ip_to_provider(ip) as "运营商" from (select CASE WHEN json_array_length(sourceIPs) = 1 then json_format(json_array_get(sourceIPs, 0)) ELSE  sourceIPs END
    as ip, count(1) as total,
    sum(CASE WHEN "responseStatus.code" < 400 then 0 
    ELSE 1 END) * 1.0 / count(1) as rate,
    count_if("responseStatus.code" = 403) as failCount
    from log  group by ip limit 10000) where ip_to_domain(ip) != 'intranet'  having "访问次数" > 10 and "失败率%" > 50 ORDER by "访问次数" desc limit 100
    ```

-   条件表达式为：`源地址 =~ ".*"`。

## 手动开通审计报表

创建Kubernetes集群时会默认选中**使用日志服务**，开启集群审计功能。若您还未开通审计报表，需手动开通审计报表。托管版集群暂不支持手动开通。

1.  登录[容器服务管理控制台](https://cs.console.aliyun.com)。

2.  在控制台左侧导航栏中，单击**集群**。

3.  在集群列表页面中，单击目标集群名称或者目标集群右侧**操作**列下的**详情**。

4.  在集群管理页左侧导航栏中，选择**安全管理** \> **集群审计**。若您还未开通审计功能，会提示您需要手动开启。

    ![集群审计开通](https://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/3295659951/p147894.png)

5.  单击**开启集群审计功能**，选择**使用已有 Project**或者**创建新Project**后，单击**确定**。

    当出现如下页面时，表示集群审计功能已开启。

    ![审计](https://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/zh-CN/3295659951/p133071.png)


## 关闭集群审计功能

如果您不再需要集群审计功能的话，可以通过以下方法关闭集群审计功能。

1.  登录[容器服务管理控制台](https://cs.console.aliyun.com)。

2.  在控制台左侧导航栏中，单击**集群**。

3.  在集群列表页面中，单击目标集群名称或者目标集群右侧**操作**列下的**详情**。

4.  在集群管理页左侧导航栏中，选择**安全管理** \> **集群审计**。

5.  在集群审计页面右上角，单击**关闭集群审计**即可关闭当前集群的审计功能。


## 计费说明

-   您可以在费用账单的账单总览页面，查看计费明细，包含审计日志的费用信息。请参见[费用账单](https://expense.console.aliyun.com/#/consumption/outline)。
-   审计日志计费方式，请参见[按量付费](/intl.zh-CN/产品定价/按量付费.md)。

## 支持第三方日志解决

您可以在集群Master各节点，在/var/log/kubernetes/kubernetes.audit路径下找到审计日志的源文件。该文件是标准的JSON格式，您可以在部署集群时选择不使用阿里云的日志服务，根据需要对接其他的日志解决方案，完成相关审计日志的采集和检索。

