---
title: gocrane/crane研究
date: 2022-06-01 22:26:09
tags: kubernetes
---



# 安装方法

https://docs.gocrane.io/dev/zh/installation/#crane-url

# 安装grafana后获取登录地址

```
NOTES:
1. Get your 'admin' user password by running:

   kubectl get secret --namespace crane-system grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo

2. The Grafana server can be accessed via port 8082 on the following DNS name from within your cluster:

   grafana.crane-system.svc.cluster.local

   Get the Grafana URL to visit by running these commands in the same shell:

     export POD_NAME=$(kubectl get pods --namespace crane-system -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
     kubectl --namespace crane-system port-forward $POD_NAME 3000

3. Login with the password from step 1 and the username: admin
#################################################################################
######   WARNING: Persistence is disabled!!! You will lose your data when   #####
######            the Grafana pod is terminated.                            #####
#################################################################################
```

# 资源推荐使用

1. 创建Analytics对象

```yaml
apiVersion: analysis.crane.io/v1alpha1
kind: Analytics
metadata:
  name: wordpress-resource
spec:
  type: Resource                        # This can only be "Resource" or "HPA".
  completionStrategy:
    completionStrategyType: Periodical  # This can only be "Once" or "Periodical".
    periodSeconds: 86400                # analytics selected resources every 1 day
  resourceSelectors:                    # defines all the resources to be select with
    - kind: Deployment
      apiVersion: apps/v1
      name: happy-panda-wordpress
```

2. 查看Analytics对象的status部分，找到对应的recommend对象

kubectl get Analytics wordpress-resource -o yaml

```yaml
apiVersion: analysis.crane.io/v1alpha1
kind: Analytics
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"analysis.crane.io/v1alpha1","kind":"Analytics","metadata":{"annotations":{},"name":"wordpress-resource","namespace":"default"},"spec":{"completionStrategy":{"completionStrategyType":"Periodical","periodSeconds":86400},"resourceSelectors":[{"apiVersion":"apps/v1","kind":"Deployment","name":"happy-panda-wordpress"}],"type":"Resource"}}
  creationTimestamp: "2022-06-05T13:40:23Z"
  generation: 2
  name: wordpress-resource
  namespace: default
  resourceVersion: "2849474"
  uid: 19225211-0805-4116-96b6-79cc801e0cc0
spec:
  completionStrategy:
    completionStrategyType: Periodical
    periodSeconds: 86400
  resourceSelectors:
  - apiVersion: apps/v1
    kind: Deployment
    labelSelector: {}
    name: happy-panda-wordpress
  type: Resource
status:
  conditions:
  - lastTransitionTime: "2022-06-05T13:40:23Z"
    message: Analytics is ready
    reason: AnalyticsReady
    status: "True"
    type: Ready
  lastUpdateTime: "2022-06-05T13:40:23Z"
  recommendations:
  - name: wordpress-resource-resource-dkk5t
    namespace: default
    targetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: happy-panda-wordpress
      namespace: default
    uid: 23f5b701-031b-440c-a646-e7aa818b4bfe
```

3. 查看recommend对象的targetRef即为系统推荐的CPU和内存建议

kubectl get recommend wordpress-resource-resource-dkk5t -o yaml

```yaml
apiVersion: analysis.crane.io/v1alpha1
kind: Recommendation
metadata:
  creationTimestamp: "2022-06-05T13:40:23Z"
  generateName: wordpress-resource-resource-
  generation: 2
  labels:
    app.kubernetes.io/instance: happy-panda
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: wordpress
    helm.sh/chart: wordpress-11.1.2
  name: wordpress-resource-resource-dkk5t
  namespace: default
  ownerReferences:
  - apiVersion: analysis.crane.io/v1alpha1
    blockOwnerDeletion: false
    controller: false
    kind: Analytics
    name: wordpress-resource
    uid: 19225211-0805-4116-96b6-79cc801e0cc0
  resourceVersion: "2849476"
  uid: 23f5b701-031b-440c-a646-e7aa818b4bfe
spec:
  adoptionType: StatusAndAnnotation
  completionStrategy:
    completionStrategyType: Periodical
    periodSeconds: 86400
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: happy-panda-wordpress
    namespace: default
  type: Resource
status:
  conditions:
  - lastTransitionTime: "2022-06-05T13:40:23Z"
    message: Recommendation is ready
    reason: RecommendationReady
    status: "True"
    type: Ready
  lastUpdateTime: "2022-06-05T13:40:23Z"
  recommendedValue: |
    containers:
    - containerName: wordpress
      target:
        cpu: 114m
        memory: "241172479"
```

