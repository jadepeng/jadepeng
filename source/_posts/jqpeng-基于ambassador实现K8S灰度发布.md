---
title: 基于ambassador实现K8S灰度发布
tags: ["灰度发布","ambassador","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-01-02 20:32
---
文章作者:jqpeng
原文链接: [基于ambassador实现K8S灰度发布](https://www.cnblogs.com/xiaoqi/p/ambassador.html)

## 为什么需要灰度发布

灰度发布（又名金丝雀发布）是指在黑与白之间，能够平滑过渡的一种发布方式。在其上可以进行A/B testing，即让一部分用户继续用产品特性A，一部分用户开始用产品特性B，如果用户对B没有什么反对意见，那么逐步扩大范围，把所有用户都迁移到B上面来。

总结下一些应用场景：

- 微服务依赖很多组件，需要在实际环境验证
- 部署新功能有风险，然后可以通过导流一小部分用户实际使用，来减小风险
- 让特定的用户访问新版本，比如部署一个版本，只让测试使用
- A/B Testing，部署两个版本，进行版本对比，比如验证两个推荐服务的推荐效果


灰度发布可以保证整体系统的稳定，在初始灰度的时候就可以发现、调整问题，以保证其影响度。

## ambassador介绍

ambassador[æmˈbæsədər]，是Kubernetes微服务 API gateway,基于Envoy Proxy。


> Open Source Kubernetes-Native API Gateway built on the Envoy Proxy


官方地址：

[https://www.getambassador.io/](https://www.getambassador.io/)

## 部署ambassador

按官网提示部署ambassador


    cat <<EOF | kubectl apply -f -
    ---
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        service: ambassador-admin
      name: ambassador-admin
    spec:
      type: NodePort
      ports:
      - name: ambassador-admin
        port: 8877
        targetPort: 8877
      selector:
        service: ambassador
    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRole
    metadata:
      name: ambassador
    rules:
    - apiGroups: [""]
      resources: [ "endpoints", "namespaces", "secrets", "services" ]
      verbs: ["get", "list", "watch"]
    - apiGroups: [ "getambassador.io" ]
      resources: [ "*" ]
      verbs: ["get", "list", "watch"]
    - apiGroups: [ "apiextensions.k8s.io" ]
      resources: [ "customresourcedefinitions" ]
      verbs: ["get", "list", "watch"]
    - apiGroups: [ "networking.internal.knative.dev" ]
      resources: [ "clusteringresses", "ingresses" ]
      verbs: ["get", "list", "watch"]
    - apiGroups: [ "networking.internal.knative.dev" ]
      resources: [ "ingresses/status", "clusteringresses/status" ]
      verbs: ["update"]
    - apiGroups: [ "extensions" ]
      resources: [ "ingresses" ]
      verbs: ["get", "list", "watch"]
    - apiGroups: [ "extensions" ]
      resources: [ "ingresses/status" ]
      verbs: ["update"]
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: ambassador
    ---
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRoleBinding
    metadata:
      name: ambassador
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: ambassador
    subjects:
    - kind: ServiceAccount
      name: ambassador
      namespace: kube-system
    ---
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: authservices.getambassador.io
    spec:
      group: getambassador.io
      version: v1
      versions:
      - name: v1
        served: true
        storage: true
      scope: Namespaced
      names:
        plural: authservices
        singular: authservice
        kind: AuthService
        categories:
        - ambassador-crds
    ---
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: consulresolvers.getambassador.io
    spec:
      group: getambassador.io
      version: v1
      versions:
      - name: v1
        served: true
        storage: true
      scope: Namespaced
      names:
        plural: consulresolvers
        singular: consulresolver
        kind: ConsulResolver
    ---
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: kubernetesendpointresolvers.getambassador.io
    spec:
      group: getambassador.io
      version: v1
      versions:
      - name: v1
        served: true
        storage: true
      scope: Namespaced
      names:
        plural: kubernetesendpointresolvers
        singular: kubernetesendpointresolver
        kind: KubernetesEndpointResolver
    ---
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: kubernetesserviceresolvers.getambassador.io
    spec:
      group: getambassador.io
      version: v1
      versions:
      - name: v1
        served: true
        storage: true
      scope: Namespaced
      names:
        plural: kubernetesserviceresolvers
        singular: kubernetesserviceresolver
        kind: KubernetesServiceResolver
    ---
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: mappings.getambassador.io
    spec:
      group: getambassador.io
      version: v1
      versions:
      - name: v1
        served: true
        storage: true
      scope: Namespaced
      names:
        plural: mappings
        singular: mapping
        kind: Mapping
        categories:
        - ambassador-crds
    ---
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: modules.getambassador.io
    spec:
      group: getambassador.io
      version: v1
      versions:
      - name: v1
        served: true
        storage: true
      scope: Namespaced
      names:
        plural: modules
        singular: module
        kind: Module
        categories:
        - ambassador-crds
    ---
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: ratelimitservices.getambassador.io
    spec:
      group: getambassador.io
      version: v1
      versions:
      - name: v1
        served: true
        storage: true
      scope: Namespaced
      names:
        plural: ratelimitservices
        singular: ratelimitservice
        kind: RateLimitService
        categories:
        - ambassador-crds
    ---
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: tcpmappings.getambassador.io
    spec:
      group: getambassador.io
      version: v1
      versions:
      - name: v1
        served: true
        storage: true
      scope: Namespaced
      names:
        plural: tcpmappings
        singular: tcpmapping
        kind: TCPMapping
        categories:
        - ambassador-crds
    ---
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: tlscontexts.getambassador.io
    spec:
      group: getambassador.io
      version: v1
      versions:
      - name: v1
        served: true
        storage: true
      scope: Namespaced
      names:
        plural: tlscontexts
        singular: tlscontext
        kind: TLSContext
        categories:
        - ambassador-crds
    ---
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: tracingservices.getambassador.io
    spec:
      group: getambassador.io
      version: v1
      versions:
      - name: v1
        served: true
        storage: true
      scope: Namespaced
      names:
        plural: tracingservices
        singular: tracingservice
        kind: TracingService
        categories:
        - ambassador-crds
    ---
    apiVersion: apiextensions.k8s.io/v1beta1
    kind: CustomResourceDefinition
    metadata:
      name: logservices.getambassador.io
    spec:
      group: getambassador.io
      version: v1
      versions:
      - name: v1
        served: true
        storage: true
      scope: Namespaced
      names:
        plural: logservices
        singular: logservice
        kind: LogService
        categories:
        - ambassador-crds
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ambassador
    spec:
      replicas: 3
      selector:
        matchLabels:
          service: ambassador
      template:
        metadata:
          annotations:
            sidecar.istio.io/inject: "false"
            "consul.hashicorp.com/connect-inject": "false"
          labels:
            service: ambassador
        spec:
          affinity:
            podAntiAffinity:
              preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 100
                podAffinityTerm:
                  labelSelector:
                    matchLabels:
                      service: ambassador
                  topologyKey: kubernetes.io/hostname
          serviceAccountName: ambassador
          containers:
          - name: ambassador
            image: quay.azk8s.cn/datawire/ambassador:0.86.1
            resources:
              limits:
                cpu: 1
                memory: 400Mi
              requests:
                cpu: 200m
                memory: 100Mi
            env:
            - name: AMBASSADOR_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            ports:
            - name: http
              containerPort: 8080
            - name: https
              containerPort: 8443
            - name: admin
              containerPort: 8877
            livenessProbe:
              httpGet:
                path: /ambassador/v0/check_alive
                port: 8877
              initialDelaySeconds: 30
              periodSeconds: 3
            readinessProbe:
              httpGet:
                path: /ambassador/v0/check_ready
                port: 8877
              initialDelaySeconds: 30
              periodSeconds: 3
            volumeMounts:
            - name: ambassador-pod-info
              mountPath: /tmp/ambassador-pod-info
          volumes:
          - name: ambassador-pod-info
            downwardAPI:
              items:
              - path: "labels"
                fieldRef:
                  fieldPath: metadata.labels
          restartPolicy: Always
          securityContext:
            runAsUser: 8888
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: ambassador
    spec:
      type: NodePort
      externalTrafficPolicy: Local
      ports:
       - port: 80
         targetPort: 8080
      selector:
        service: ambassador
    
    
    EOF
    


为了方便访问网关，生成一个ingress：

* * *


    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
     annotations:
       nginx.ingress.kubernetes.io/proxy-body-size: "0"
       nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
       nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
       kubernetes.io/tls-acme: 'true'
     name: ambassador
    spec:
     rules:
     - host: ambassador.iflyresearch.com
       http:
         paths:
         - backend:
             serviceName: ambassador
             servicePort: 80
           path: /


## ambassador 配置

ambassador 使用envoy来实现相关的负载，而envoy类似nginx。ambassador的原理大概是读取service里的配置，然后自动生成envoy的配置，当service变更时，动态更新envoy的配置并重启，所以ambassador需要可以访问服务API。

ambassador  的配置是放到metadata的annotations,以`getambassador.io/config`开头：


      annotations:
        getambassador.io/config: |
          ---
          apiVersion: ambassador/v0
          kind:  Mapping
          name:  {{ .Values.service.name }}_mapping
          prefix: /{{ .Values.service.prefix }}
          service: {{ .Values.service.name }}.{{ .Release.Namespace }}


profix指定如何访问服务，service指定指向那个服务。注意，需要加上namespace名称，否则容易报找不到后端。

## ambassador 灰度

ambassador实现灰度可以根据weight权重，或者指定匹配特定的header来实现。

### 根据weight进行灰度

用法：

部署一个新版本的service，prefix和之前老服务保持一致，但是配置weight，比如20，这样20%的流量会流转到新服务，这样实现A/B Test


    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: svc-gray
      namespace: default
      annotations:
        getambassador.io/config: |
          ---
          apiVersion: ambassador/v0
          kind:  Mapping
          name:  svc1_mapping
          prefix: /svc/
          service: service-gray  weight: 20
    spec:
      selector:
        app: testservice
      ports:
      - port: 8080
        name: service-gray
        targetPort: http-api


### 根据请求头 header 进行灰度 (regex\_headers 正则匹配)

部署一个新版本，只需要特定的用户才能访问，可以通过该方案来实现。

例如：


    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: svc-gray
      namespace: default
      annotations:
        getambassador.io/config: |
          ---
          apiVersion: ambassador/v0
          kind:  Mapping
          name:  svc1_mapping
          prefix: /svc/
          service: service-gray  headers:
            gray: true
    spec:
      selector:
        app: testservice
      ports:
      - port: 8080
        name: service-gray
        targetPort: http-api


访问时，当指定gray: true时，访问灰度版本，可以用postman来测试：

![POSTMAN](https://gitee.com/jadepeng/pic/raw/master/pic/2020/1/2/1577967795378.png)

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


