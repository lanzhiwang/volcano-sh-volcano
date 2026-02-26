通过 Deployment Yaml 安装

这种安装方式支持x86_64/arm64两种架构. 在你的kubernetes集群上, 执行如下的kubectl指令.

```bash

$ kubectl apply -f https://raw.githubusercontent.com/volcano-sh/volcano/refs/tags/v1.14.1/installer/volcano-development.yaml

Namespace
    volcano-system
    volcano-monitoring

ServiceAccount
    volcano-admission
    volcano-admission-init
    volcano-controllers
    volcano-scheduler

ConfigMap
    volcano-admission-configmap
    volcano-controller-configmap
    volcano-scheduler-configmap

ClusterRole
    volcano-admission
    volcano-controllers
    volcano-scheduler

ClusterRoleBinding
    volcano-admission-role
    volcano-controllers-role
    volcano-scheduler-role

Role
    volcano-admission-init

RoleBinding
    volcano-admission-init-role

Service
    volcano-admission-service
    volcano-controllers-service
    volcano-scheduler-service

Deployment
    volcano-admission
    volcano-controllers
    volcano-scheduler

Job
    volcano-admission-init

CustomResourceDefinition
    jobs.batch.volcano.sh
    podgroups.scheduling.volcano.sh
    
    cronjobs.batch.volcano.sh
    commands.bus.volcano.sh
    
    queues.scheduling.volcano.sh
    numatopologies.nodeinfo.volcano.sh
    hypernodes.topology.volcano.sh
    nodeshards.shard.volcano.sh
    colocationconfigurations.config.volcano.sh
    jobtemplates.flow.volcano.sh
    jobflows.flow.volcano.sh

MutatingWebhookConfiguration
    volcano-admission-service-queues-mutate
    volcano-admission-service-jobs-mutate

ValidatingWebhookConfiguration
    volcano-admission-service-jobs-validate
    volcano-admission-service-queues-validate
    volcano-admission-service-podgroups-validate
    volcano-admission-service-hypernodes-validate
    volcano-admission-service-cronjobs-validate


```