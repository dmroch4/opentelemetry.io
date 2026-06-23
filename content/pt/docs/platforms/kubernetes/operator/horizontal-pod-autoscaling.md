---
title: Horizontal Pod Autoscaling
description:
  Configure o Horizontal Pod Autoscaling com seu OpenTelemetry Collector
cSpell:ignore: autoscaler mebibyte mebibytes statefulset
default_lang_commit: 6bf06ddb9fc057dd6e8092f26d988ffe7b1af5ed
---

Os Collectors gerenciados pelo OpenTelemetry Operator possuem suporte integrado
para
[horizontal pod autoscaling (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/).
O HPA aumenta ou diminui o número de réplicas (cópias) dos seus pods do
Kubernetes com base em um conjunto de métricas. Essas métricas são tipicamente
consumo de CPU e/ou memória.

Ter o OpenTelemetry Operator gerenciando a funcionalidade de HPA para o
Collector significa que você não precisa criar um recurso Kubernetes
`HorizontalPodAutoscaler` separado para o autoscaling do seu Collector.

Como o HPA só se aplica a `StatefulSets` e `Deployments` no Kubernetes,
certifique-se de que o `spec.mode` do seu Collector seja `deployment` ou
`statefulset`.

> [!NOTE]
>
> O HPA requer um
> [Metrics Server](https://github.com/kubernetes-sigs/metrics-server) em
> execução no seu cluster Kubernetes.
>
> - Clusters Kubernetes gerenciados como
>   [GKE (Google)](https://cloud.google.com/kubernetes-engine?hl=en) e
>   [AKS (Microsoft Azure)](https://azure.microsoft.com/en-us/products/kubernetes-service)
>   instalam um Metrics Server automaticamente como parte do provisionamento do
>   cluster.
> - [EKS (AWS) não vem com um Metrics Server instalado por padrão](https://docs.aws.amazon.com/eks/latest/userguide/metrics-server.html).
> - Clusters Kubernetes não gerenciados e clusters Kubernetes de desktop local
>   (por exemplo, [MiniKube](https://minikube.sigs.k8s.io/docs/),
>   [KinD](https://kind.sigs.k8s.io/), [k0s](https://k0sproject.io)) requerem
>   instalação manual do Metrics Server.
>
> Consulte a documentação do seu provedor de nuvem para determinar se o seu
> cluster Kubernetes gerenciado vem pré-instalado com um Metrics Server.

Para configurar o HPA, você deve primeiro definir suas solicitações e limites de
recursos adicionando uma configuração `spec.resources` ao seu YAML
`OpenTelemetryCollector`:

```yaml
resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 64Mi
```

> [!NOTE]
>
> Seus próprios valores podem variar.

A configuração `limits` especifica os valores máximos de memória e CPU. Neste
caso, esses limites são 100 millicores (0,1 core) de CPU e 128Mi (mebibytes,
onde 1 mebibyte == 1024 kilobytes) de RAM.

A configuração `requests` especifica a quantidade mínima garantida de recursos
alocados para o container. Neste caso, a alocação mínima é 100 millicores de CPU
e 64 mebibytes de RAM.

Em seguida, você configura as regras de autoscaling adicionando uma configuração
`spec.autoscaler` ao YAML `OpenTelemetryCollector`:

```yaml
autoscaler:
  minReplicas: 1
  maxReplicas: 2
  targetCPUUtilization: 50
  targetMemoryUtilization: 60
```

> [!NOTE]
>
> Seus próprios valores podem variar.

Juntando tudo, o início do YAML `OpenTelemetryCollector` deve ter uma aparência
semelhante a esta:

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otelcol
  namespace: opentelemetry
spec:
  mode: statefulset
  image:
    otel/opentelemetry-collector-contrib:{{% version-from-registry
    collector-processor-batch %}}
  serviceAccount: otelcontribcol
  autoscaler:
    minReplicas: 1
    maxReplicas: 2
    targetCPUUtilization: 50
    targetMemoryUtilization: 60
  resources:
    limits:
      cpu: 100m
      memory: 128Mi
    requests:
      cpu: 100m
      memory: 64Mi
```

Uma vez que o `OpenTelemetryCollector` é implantado no Kubernetes com o HPA
habilitado, o Operator cria um recurso `HorizontalPodAutoscaler` para o seu
Collector no Kubernetes. Você pode verificar isso executando

`kubectl get hpa -n <your_namespace>`

Se tudo funcionou conforme esperado, a saída do comando deve ser assim:

```nocode
NAME                REFERENCE                        TARGETS                         MINPODS   MAXPODS   REPLICAS   AGE
otelcol-collector   OpenTelemetryCollector/otelcol   memory: 68%/60%, cpu: 37%/50%   1         3         2          77s
```

Para obter informações mais detalhadas, você pode descrever seu recurso HPA
executando

`kubectl describe hpa <your_collector_name> -n <your_namespace>`

Se tudo funcionou conforme esperado, a saída do comando deve ser assim:

```nocode
Name:                                                     otelcol-collector
Namespace:                                                opentelemetry
Labels:                                                   app.kubernetes.io/benchmark-test=otelcol-contrib
                                                          app.kubernetes.io/component=opentelemetry-collector
                                                          app.kubernetes.io/destination=dynatrace
                                                          app.kubernetes.io/instance=opentelemetry.otelcol
                                                          app.kubernetes.io/managed-by=opentelemetry-operator
                                                          app.kubernetes.io/name=otelcol-collector
                                                          app.kubernetes.io/part-of=opentelemetry
                                                          app.kubernetes.io/version=0.126.0
Annotations:                                              <none>
CreationTimestamp:                                        Mon, 02 Jun 2025 17:23:52 +0000
Reference:                                                OpenTelemetryCollector/otelcol
Metrics:                                                  ( current / target )
  resource memory on pods  (as a percentage of request):  71% (95779498666m) / 60%
  resource cpu on pods  (as a percentage of request):     12% (12m) / 50%
Min replicas:                                             1
Max replicas:                                             3
OpenTelemetryCollector pods:                              3 current / 3 desired
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    ReadyForNewScale  recommended size matches current size
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from memory resource utilization (percentage of request)
  ScalingLimited  True    TooManyReplicas   the desired replica count is more than the maximum replica count
Events:
  Type     Reason                   Age                  From                       Message
  ----     ------                   ----                 ----                       -------
  Warning  FailedGetResourceMetric  2m (x4 over 2m29s)   horizontal-pod-autoscaler  unable to get metric memory: no metrics returned from resource metrics API
  Warning  FailedGetResourceMetric  89s (x7 over 2m29s)  horizontal-pod-autoscaler  No recommendation
  Normal   SuccessfulRescale        89s                  horizontal-pod-autoscaler  New size: 2; reason: memory resource utilization (percentage of request) above target
  Normal   SuccessfulRescale        59s                  horizontal-pod-autoscaler  New size: 3; reason: memory resource utilization (percentage of request) above target
```
