---
keyword: [Nvidia-container-runtime, 升级, 共享GPU, 昊天cGPU]
---

# 升级GPU节点的Docker

共享GPU的隔离能力依赖Docker 19.03.5以及与其对应的Nvidia-container-runtime版本，如果Kubernetes集群节点安装的Docker版本低于19.03.5，您需要将其升级至19.03.5。本文介绍如何升级Docker以及与其对应的Nvidia-container-runtime，从而使节点支持共享GPU。

Nvidia-container-runtime允许用户构建和运行GPU加速的Docker容器，能够自动对容器进行配置，以达到容器使用Nvidia GPU的目的。

## 操作步骤

**说明：** 本文操作步骤仅适用于CentOS和Alibaba Cloud Linux 2操作系统。

在执行以下操作前，您需要使用命令行工具连接您的Kubernetes集群。详情请参见[通过kubectl连接Kubernetes集群](/intl.zh-CN/Kubernetes集群用户指南/集群管理/管理与访问集群/通过kubectl连接Kubernetes集群.md)。

1.  在Master节点执行以下命令，下线节点。

    为避免在Docker升级期间，有Pod调度到该节点，您需要将节点标记为不可调度。

    ```
    kubectl cordon <NODE_NAME>
    ```

    **说明：** <NODE\_NAME\>为待升级Docker版本的节点名称。执行命令kubectl get nodes查询节点名称。

2.  在Master节点上执行以下命令，迁移节点中的Pod。

    节点下线以后，需要将该节点上的Pod迁移至其他节点。

    ```
    kubectl drain <NODE_NAME> --ignore-daemonsets --delete-local-data --force
    ```

    **说明：** <NODE\_NAME\>为待升级Docker版本的节点。

3.  执行以下命令，暂停kubelet和Docker服务。

    在待升级Docker版本的节点上停止kubelet和Docker服务。

    ```
    service kubelet stop
    docker rm -f $(docker ps -aq)
    service docker stop
    ```

4.  执行以下命令，卸载Docker和Nvidia-container-runtime。

    在待升级Docker版本的节点上卸载旧版Docker和Nvidia-container-runtime。

    ```
    yum remove -y docker-ce docker-ce-cli containerd
    yum remove -y nvidia-container-runtime*  libnvidia-container*
    ```

5.  执行以下命令，备份并移除daemon.json文件。

    ```
    cat /etc/docker/daemon.json
    {
        "default-runtime": "nvidia",
        "runtimes": {
            "nvidia": {
                "path": "/usr/bin/nvidia-container-runtime",
                "runtimeArgs": []
            }
        },
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "json-file",
        "log-opts": {
            "max-size": "100m",
            "max-file": "10"
        },
        "bip": "169.254.123.1/24",
        "oom-score-adjust": -1000,
        "storage-driver": "overlay2",
        "storage-opts":["overlay2.override_kernel_check=true"],
        "live-restore": true
    }
    mv /etc/docker/daemon.json /tmp
    ```

6.  执行以下命令，安装Docker。

    在待升级Docker版本的节点上下载Docker安装包。

    ```
    VERSION=19.03.5 
    URL=http://aliacs-k8s-cn-beijing.oss-cn-beijing.aliyuncs.com/public/pkg/docker/docker-${VERSION}.tar.gz 
    curl -ssL $URL -o /tmp/docker-${VERSION}.tar.gz  
    cd /tmp
    tar -xf docker-${VERSION}.tar.gz
    cd /tmp/pkg/docker/${VERSION}/rpm
    yum localinstall -y $(ls .)
    ```

7.  执行以下命令，在节点上安装Nvidia-container-runtime。

    ```
    cd /tmp
    yum install -y unzip
    wget http://kubeflow.oss-cn-beijing.aliyuncs.com/nvidia.zip
    unzip nvidia.zip
    yum -y -q --nogpgcheck localinstall /tmp/nvidia/*
    ```

8.  执行以下命令，配置daemon.json。

    将上述的daemon.json覆盖/etc/docker/daemon.json，使原有配置生效。

    ```
    mv /tmp/daemon.json  /etc/docker/daemon.json 
    cat /etc/docker/daemon.json
    {
        "default-runtime": "nvidia",
        "runtimes": {
            "nvidia": {
                "path": "/usr/bin/nvidia-container-runtime",
                "runtimeArgs": []
            }
        },
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "json-file",
        "log-opts": {
            "max-size": "100m",
            "max-file": "10"
        },
        "bip": "169.254.123.1/24",
        "oom-score-adjust": -1000,
        "storage-driver": "overlay2",
        "storage-opts":["overlay2.override_kernel_check=true"],
        "live-restore": true
    }
    ```

9.  执行以下命令，重启Docker和kubelet服务。

    ```
    service docker start
    service kubelet start
    ```

10. 执行以下命令，上线节点。

    节点Docker升级完成后，使节点在集群中的状态变成可调度。

    ```
    kubectl uncordon <NODE_NAME>
    ```

    **说明：** <NODE\_NAME\>为升级完Docker版本的节点名称。

11. 执行以下命令，在该GPU节点重新启动GPU安装程序。

    ```
    docker ps |grep cgpu-installer | awk '{print $1}' | xargs docker rm -f
    ```


**相关文档**  


[共享GPU概述](/intl.zh-CN/Kubernetes集群用户指南/GPU/NPU管理/共享GPU调度/共享GPU概述.md)

[安装共享GPU组件](/intl.zh-CN/Kubernetes集群用户指南/GPU/NPU管理/共享GPU调度/安装共享GPU组件.md)

