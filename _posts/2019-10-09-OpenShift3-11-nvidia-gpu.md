---
layout:     post
title:      Openshift3.11 nvidia gpu任务调度
subtitle:   安装部署过程介绍
date:       2019-10-09
author:     Frederick
header-img: img/ldap2.jpg
catalog: true
tags:
    - 大数据
    - 分布式
    - Docker
    - kubernets
    - 机器学习
---


# Openshift3.11 如何调度GPU任务POD
**参考**：https://blog.openshift.com/how-to-use-gpus-with-deviceplugin-in-openshift-3-10/

**基础环境安装**：NVIDIA驱动、nvidia-docker安装与测试详见参考链接。
本文主要提供经过实践验证过的创建SCC服务的nvidia-deviceplugin-scc.yaml 文件和 创建NVIDIA Device Plugin daemonset  的nvidia-deviceplugin.yaml文件以及测试调用GPU POD test-gpu.yaml 文件。

- **1.创建SCC服务 "oc create -f nvidia-deviceplugin-scc.yaml"**

        allowHostDirVolumePlugin: true
        allowHostIPC: true
        allowHostNetwork: true
        allowHostPID: true
        allowHostPorts: true
        allowPrivilegedContainer: true
        allowedCapabilities:
        - '*'
        allowedFlexVolumes: null
        apiVersion: v1
        defaultAddCapabilities:
        - '*'
        fsGroup:
        type: RunAsAny
        groups:
        - system:cluster-admins
        - system:nodes
        - system:masters
        kind: SecurityContextConstraints
        metadata:
        annotations:
        kubernetes.io/description: anyuid provides all features of the restricted SCC
            but allows users to run with any UID and any GID.
        creationTimestamp: null
        name: nvidia-deviceplugin
        priority: 10
        readOnlyRootFilesystem: false
        requiredDropCapabilities:
        runAsUser:
        type: RunAsAny
        seLinuxContext:
        type: RunAsAny
        seccompProfiles:
        - '*'
        supplementalGroups:
        type: RunAsAny
        users:
        - system:serviceaccount:nvidia:nvidia-deviceplugin
        volumes:
        - '*':

- **2.创建 NVIDIA Device Plugin daemonset  "oc create -f nvidia-deviceplugin.yaml"**


        apiVersion: extensions/v1beta1
        kind: DaemonSet
        metadata:
        name: nvidia-device-plugin-daemonset
        namespace: nvidia
        spec:
        template:
            metadata:
            # Mark this pod as a critical add-on; when enabled, the critical add-on scheduler
            # reserves resources for critical add-on pods so that they can be rescheduled after
            # a failure.  This annotation works in tandem with the toleration below.
            annotations:
                scheduler.alpha.kubernetes.io/critical-pod: ""
            labels:
                name: nvidia-device-plugin-ds
            spec:
            affinity:
                nodeAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                    nodeSelectorTerms:
                    - matchExpressions:
                    - key: openshift.com/gpu-accelerator
                        operator: Exists
            tolerations:
            # Allow this pod to be rescheduled while the node is in "critical add-ons only" mode.
            # This, along with the annotation above marks this pod as a critical add-on.
            - key: CriticalAddonsOnly
                operator: Exists
            - key: nvidia.com/gpu
                operator: Exists
                effect: NoSchedule
            serviceAccount: nvidia-deviceplugin
            serivceAccountName: nvidia-deviceplugin
            hostNetwork: true
            hostPID: true
            containers:
            - image: nvidia/k8s-device-plugin:1.11
                name: nvidia-device-plugin-ctr
                securityContext:
                allowPrivilegeEscalation: false
                capabilities:
                    drop: ["ALL"]
                seLinuxOptions:
                    type: nvidia_container_t
                volumeMounts:
                - name: device-plugin
                    mountPath: /var/lib/kubelet/device-plugins
            volumes:
                - name: device-plugin
                hostPath:
                    path: /var/lib/kubelet/device-plugins

- **3.测试调用GPU "测试调用GPU POD test-gpu.yaml"**

 
        apiVersion: v1
        kind: Pod
        metadata:
        name: cuda3
        namespace: nvidia
        spec:
        restartPolicy: OnFailure
        containers:
            - name: cuda3
            image: "docker.io/nvidia/cuda:9.0-base"
            args: ["nvidia-smi"]
            resources:
                limits:
                nvidia.com/gpu: 3 # requesting 3 GPU