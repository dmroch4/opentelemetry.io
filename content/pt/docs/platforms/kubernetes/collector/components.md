---
title: Componentes Importantes para Kubernetes
linkTitle: Componentes
# prettier-ignore
cSpell:ignore: alertmanagers filelog horizontalpodautoscalers hostfs hostmetrics k8sattributes kubelet kubeletstats replicasets replicationcontrollers resourcequotas statefulsets varlibdockercontainers varlogpods
default_lang_commit: dd8216983d52b89b2b95ca640364b3292740a5db
---

O [OpenTelemetry Collector](/docs/collector/) suporta muitos receivers e
processors diferentes para facilitar o monitoramento do Kubernetes. Esta seção
aborda os componentes mais importantes para coletar dados do Kubernetes e
enriquecê-los.

Componentes abordados nesta página:

- [Kubernetes Attributes Processor](#kubernetes-attributes-processor): adiciona
  metadados do Kubernetes à telemetria de aplicações recebida.
- [Kubeletstats Receiver](#kubeletstats-receiver): extrai métricas de nós, pods
  e containers do servidor de API em um kubelet.
- [Filelog Receiver](#filelog-receiver): coleta logs do Kubernetes e logs de
  aplicações gravados em stdout/stderr.
- [Kubernetes Cluster Receiver](#kubernetes-cluster-receiver): coleta métricas a
  nível de cluster e eventos de entidade.
- [Kubernetes Objects Receiver](#kubernetes-objects-receiver): coleta objetos,
  como eventos, do servidor de API do Kubernetes.
- [Prometheus Receiver](#prometheus-receiver): recebe métricas no formato
  [Prometheus](https://prometheus.io/).
- [Host Metrics Receiver](#host-metrics-receiver): coleta métricas do host em
  nós do Kubernetes.

Para traces, métricas ou logs de aplicações, recomendamos o
[OTLP receiver](https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver/otlpreceiver),
mas qualquer receiver adequado aos seus dados é apropriado.

## Kubernetes Attributes Processor

| Padrão de Implantação | Utilizável |
| --------------------- | ---------- |
| DaemonSet (agent)     | Sim        |
| Deployment (gateway)  | Sim        |
| Sidecar               | Não        |

O Kubernetes Attributes Processor descobre automaticamente os pods do
Kubernetes, extrai seus metadados e adiciona os metadados extraídos a spans,
métricas e logs como atributos de recurso.

**O Kubernetes Attributes Processor é um dos componentes mais importantes para
um collector executado no Kubernetes. Qualquer collector que receba dados de
aplicação deve utilizá-lo.** Por adicionar contexto do Kubernetes à sua
telemetria, o Kubernetes Attributes Processor permite correlacionar os traces,
métricas e logs da sua aplicação com a telemetria do Kubernetes, como métricas e
traces de pods.

O Kubernetes Attributes Processor usa a API do Kubernetes para descobrir todos
os pods em execução em um cluster e mantém um registro de seus endereços IP,
UIDs de pod e metadados relevantes. Por padrão, os dados que passam pelo
processor são associados a um pod pelo endereço IP da requisição recebida, mas
regras diferentes podem ser configuradas. Como o processor usa a API do
Kubernetes, ele requer permissões especiais (veja o exemplo abaixo). Se você
estiver usando o
[OpenTelemetry Collector Helm chart](/docs/platforms/kubernetes/helm/collector/)
você pode usar o preset
[`kubernetesAttributes`](/docs/platforms/kubernetes/helm/collector/#kubernetes-attributes-preset)
para começar.

Os seguintes atributos são adicionados por padrão:

- `k8s.namespace.name`
- `k8s.pod.name`
- `k8s.pod.uid`
- `k8s.pod.start_time`
- `k8s.deployment.name`
- `k8s.node.name`

O Kubernetes Attributes Processor também pode definir atributos de recurso
personalizados para traces, métricas e logs usando as labels e anotações do
Kubernetes que você adicionou aos seus pods e namespaces.

```yaml
k8sattributes:
  auth_type: 'serviceAccount'
  extract:
    metadata: # extracted from the pod
      - k8s.namespace.name
      - k8s.pod.name
      - k8s.pod.start_time
      - k8s.pod.uid
      - k8s.deployment.name
      - k8s.node.name
    annotations:
      # Extracts the value of a pod annotation with key `annotation-one` and inserts it as a resource attribute with key `a1`
      - tag_name: a1
        key: annotation-one
        from: pod
      # Extracts the value of a namespaces annotation with key `annotation-two` with regexp and inserts it as a resource  with key `a2`
      - tag_name: a2
        key: annotation-two
        regex: field=(?P<value>.+)
        from: namespace
    labels:
      # Extracts the value of a namespaces label with key `label1` and inserts it as a resource attribute with key `l1`
      - tag_name: l1
        key: label1
        from: namespace
      # Extracts the value of a pod label with key `label2` with regexp and inserts it as a resource attribute with key `l2`
      - tag_name: l2
        key: label2
        regex: field=(?P<value>.+)
        from: pod
  pod_association: # How to associate the data to a pod (order matters)
    - sources: # First try to use the value of the resource attribute k8s.pod.ip
        - from: resource_attribute
          name: k8s.pod.ip
    - sources: # Then try to use the value of the resource attribute k8s.pod.uid
        - from: resource_attribute
          name: k8s.pod.uid
    - sources: # If neither of those work, use the request's connection to get the pod IP.
        - from: connection
```

Também existem opções de configuração especiais para quando o collector é
implantado como DaemonSet (agent) ou como Deployment (gateway) do Kubernetes.
Para detalhes, consulte
[Deployment Scenarios](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/k8sattributesprocessor#deployment-scenarios)

Para detalhes de configuração do Kubernetes Attributes Processor, consulte
[Kubernetes Attributes Processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/k8sattributesprocessor).

Como o processor usa a API do Kubernetes, ele precisa da permissão correta para
funcionar. Para a maioria dos casos de uso, você deve conceder à service account
que executa o collector as seguintes permissões por meio de um ClusterRole.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: collector
  namespace: <OTEL_COL_NAMESPACE>
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector
rules:
  - apiGroups:
      - ''
    resources:
      - 'pods'
      - 'namespaces'
    verbs:
      - 'get'
      - 'watch'
      - 'list'
  - apiGroups:
      - 'apps'
    resources:
      - 'replicasets'
    verbs:
      - 'get'
      - 'list'
      - 'watch'
  - apiGroups:
      - 'extensions'
    resources:
      - 'replicasets'
    verbs:
      - 'get'
      - 'list'
      - 'watch'
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector
subjects:
  - kind: ServiceAccount
    name: collector
    namespace: <OTEL_COL_NAMESPACE>
roleRef:
  kind: ClusterRole
  name: otel-collector
  apiGroup: rbac.authorization.k8s.io
```

## Kubeletstats Receiver

| Padrão de Implantação | Utilizável                                                     |
| --------------------- | -------------------------------------------------------------- |
| DaemonSet (agent)     | Preferido                                                      |
| Deployment (gateway)  | Sim, mas coletará métricas apenas do nó em que está implantado |
| Sidecar               | Não                                                            |

Cada nó do Kubernetes executa um kubelet que inclui um servidor de API. O
Kubeletstats Receiver se conecta a esse kubelet via servidor de API para coletar
métricas sobre o nó e as cargas de trabalho em execução nele.

Existem diferentes métodos de autenticação, mas normalmente uma service account
é utilizada. A service account também precisará das permissões corretas para
extrair dados do Kubelet (veja abaixo). Se você estiver usando o
[OpenTelemetry Collector Helm chart](/docs/platforms/kubernetes/helm/collector/)
você pode usar o preset
[`kubeletMetrics`](/docs/platforms/kubernetes/helm/collector/#kubelet-metrics-preset)
para começar.

Por padrão, as métricas serão coletadas para pods e nós, mas você pode
configurar o receiver para coletar métricas de containers e volumes também. O
receiver também permite configurar a frequência com que as métricas são
coletadas:

```yaml
receivers:
  kubeletstats:
    collection_interval: 10s
    auth_type: 'serviceAccount'
    endpoint: '${env:K8S_NODE_NAME}:10250'
    insecure_skip_verify: true
    metric_groups:
      - node
      - pod
      - container
```

Para detalhes específicos sobre quais métricas são coletadas, consulte
[Default Metrics](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/kubeletstatsreceiver/documentation.md).
Para detalhes específicos de configuração, consulte
[Kubeletstats Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/kubeletstatsreceiver).

Como o processor usa a API do Kubernetes, ele precisa da permissão correta para
funcionar. Para a maioria dos casos de uso, você deve conceder à service account
que executa o Collector as seguintes permissões por meio de um ClusterRole.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-collector
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector
rules:
  - apiGroups: ['']
    resources: ['nodes/stats']
    verbs: ['get', 'watch', 'list']
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: otel-collector
subjects:
  - kind: ServiceAccount
    name: otel-collector
    namespace: default
```

## Filelog Receiver

| Padrão de Implantação | Utilizável                                                 |
| --------------------- | ---------------------------------------------------------- |
| DaemonSet (agent)     | Preferido                                                  |
| Deployment (gateway)  | Sim, mas coletará logs apenas do nó em que está implantado |
| Sidecar               | Sim, mas isso seria considerado configuração avançada      |

O Filelog Receiver acompanha e analisa logs de arquivos. Embora não seja um
receiver específico do Kubernetes, ainda é a solução de fato para coletar
quaisquer logs do Kubernetes.

O Filelog Receiver é composto por Operators encadeados para processar um log.
Cada Operator executa uma responsabilidade simples, como analisar um timestamp
ou JSON. Configurar um Filelog Receiver não é trivial. Se você estiver usando o
[OpenTelemetry Collector Helm chart](/docs/platforms/kubernetes/helm/collector/)
você pode usar o preset
[`logsCollection`](/docs/platforms/kubernetes/helm/collector/#logs-collection-preset)
para começar.

Como os logs do Kubernetes normalmente seguem um conjunto de formatos padrão,
uma configuração típica do Filelog Receiver para Kubernetes tem a seguinte
aparência:

```yaml
filelog:
  include:
    - /var/log/pods/*/*/*.log
  exclude:
    # Exclude logs from all containers named otel-collector
    - /var/log/pods/*/otel-collector/*.log
  start_at: end
  include_file_path: true
  include_file_name: false
  operators:
    # parse container logs
    - type: container
      id: container-parser
```

Para detalhes de configuração do Filelog Receiver, consulte
[Filelog Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver).

Além da configuração do Filelog Receiver, a instalação do OpenTelemetry
Collector no Kubernetes precisará de acesso aos logs que deseja coletar.
Tipicamente isso significa adicionar alguns volumes e volumeMounts ao manifesto
do seu collector:

```yaml
---
apiVersion: apps/v1
kind: DaemonSet
...
spec:
  ...
  template:
    ...
    spec:
      ...
      containers:
        - name: opentelemetry-collector
          ...
          volumeMounts:
            ...
            # Mount the volumes to the collector container
            - name: varlogpods
              mountPath: /var/log/pods
              readOnly: true
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            ...
      volumes:
        ...
        # Typically the collector will want access to pod logs and container logs
        - name: varlogpods
          hostPath:
            path: /var/log/pods
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        ...
```

## Kubernetes Cluster Receiver

| Padrão de Implantação | Utilizável                                               |
| --------------------- | -------------------------------------------------------- |
| DaemonSet (agent)     | Sim, mas resultará em dados duplicados                   |
| Deployment (gateway)  | Sim, mas mais de uma réplica resulta em dados duplicados |
| Sidecar               | Não                                                      |

O Kubernetes Cluster Receiver coleta métricas e eventos de entidade sobre o
cluster como um todo usando o servidor de API do Kubernetes. Use este receiver
para responder perguntas sobre fases de pods, condições de nós e outras questões
a nível de cluster. Como o receiver coleta telemetria para o cluster como um
todo, apenas uma instância do receiver é necessária em todo o cluster para
coletar todos os dados.

Existem diferentes métodos de autenticação, mas normalmente uma service account
é utilizada. A service account também precisa das permissões corretas para
extrair dados do servidor de API do Kubernetes (veja abaixo). Se você estiver
usando o
[OpenTelemetry Collector Helm chart](/docs/platforms/kubernetes/helm/collector/)
você pode usar o preset
[`clusterMetrics`](/docs/platforms/kubernetes/helm/collector/#cluster-metrics-preset)
para começar.

Para condições de nós, o receiver coleta apenas `Ready` por padrão, mas pode ser
configurado para coletar mais. O receiver também pode ser configurado para
reportar um conjunto de recursos alocáveis, como `cpu` e `memory`:

```yaml
k8s_cluster:
  auth_type: serviceAccount
  node_conditions_to_report:
    - Ready
    - MemoryPressure
  allocatable_types_to_report:
    - cpu
    - memory
```

Para saber mais sobre as métricas coletadas, consulte
[Default Metrics](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/k8sclusterreceiver/documentation.md)
Para detalhes de configuração, consulte
[Kubernetes Cluster Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/k8sclusterreceiver).

Como o processor usa a API do Kubernetes, ele precisa da permissão correta para
funcionar. Para a maioria dos casos de uso, você deve conceder à service account
que executa o Collector as seguintes permissões por meio de um ClusterRole.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-collector-opentelemetry-collector
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector-opentelemetry-collector
rules:
  - apiGroups:
      - ''
    resources:
      - events
      - namespaces
      - namespaces/status
      - nodes
      - nodes/spec
      - pods
      - pods/status
      - replicationcontrollers
      - replicationcontrollers/status
      - resourcequotas
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - apps
    resources:
      - daemonsets
      - deployments
      - replicasets
      - statefulsets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - daemonsets
      - deployments
      - replicasets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - batch
    resources:
      - jobs
      - cronjobs
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - autoscaling
    resources:
      - horizontalpodautoscalers
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector-opentelemetry-collector
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: otel-collector-opentelemetry-collector
subjects:
  - kind: ServiceAccount
    name: otel-collector-opentelemetry-collector
    namespace: default
```

## Kubernetes Objects Receiver

| Padrão de Implantação | Utilizável                                               |
| --------------------- | -------------------------------------------------------- |
| DaemonSet (agent)     | Sim, mas resultará em dados duplicados                   |
| Deployment (gateway)  | Sim, mas mais de uma réplica resulta em dados duplicados |
| Sidecar               | Não                                                      |

O Kubernetes Objects receiver coleta objetos, seja por polling ou watching, do
servidor de API do Kubernetes. O caso de uso mais comum para este receiver é
observar eventos do Kubernetes, mas ele pode ser usado para coletar qualquer
tipo de objeto do Kubernetes. Como o receiver coleta telemetria para o cluster
como um todo, apenas uma instância do receiver é necessária em todo o cluster
para coletar todos os dados.

Atualmente, apenas uma service account pode ser usada para autenticação. A
service account também precisa das permissões corretas para extrair dados do
servidor de API do Kubernetes (veja abaixo). Se você estiver usando o
[OpenTelemetry Collector Helm chart](/docs/platforms/kubernetes/helm/collector/)
e quiser ingerir eventos, pode usar o preset
[`kubernetesEvents`](/docs/platforms/kubernetes/helm/collector/#cluster-metrics-preset)
para começar.

Para objetos configurados para polling, o receiver usará a API do Kubernetes
para listar periodicamente todos os objetos no Cluster. Cada objeto será
convertido em seu próprio log. Para objetos configurados para watching, o
receiver cria um stream com a API do Kubernetes que recebe atualizações à medida
que os objetos mudam.

Para ver quais objetos estão disponíveis para coleta, execute em seu cluster
`kubectl api-resources`:

<!-- cspell:disable -->

```console
kubectl api-resources
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
bindings                                       v1                                     true         Binding
componentstatuses                 cs           v1                                     false        ComponentStatus
configmaps                        cm           v1                                     true         ConfigMap
endpoints                         ep           v1                                     true         Endpoints
events                            ev           v1                                     true         Event
limitranges                       limits       v1                                     true         LimitRange
namespaces                        ns           v1                                     false        Namespace
nodes                             no           v1                                     false        Node
persistentvolumeclaims            pvc          v1                                     true         PersistentVolumeClaim
persistentvolumes                 pv           v1                                     false        PersistentVolume
pods                              po           v1                                     true         Pod
podtemplates                                   v1                                     true         PodTemplate
replicationcontrollers            rc           v1                                     true         ReplicationController
resourcequotas                    quota        v1                                     true         ResourceQuota
secrets                                        v1                                     true         Secret
serviceaccounts                   sa           v1                                     true         ServiceAccount
services                          svc          v1                                     true         Service
mutatingwebhookconfigurations                  admissionregistration.k8s.io/v1        false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io/v1        false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io/v1                false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io/v1              false        APIService
controllerrevisions                            apps/v1                                true         ControllerRevision
daemonsets                        ds           apps/v1                                true         DaemonSet
deployments                       deploy       apps/v1                                true         Deployment
replicasets                       rs           apps/v1                                true         ReplicaSet
statefulsets                      sts          apps/v1                                true         StatefulSet
tokenreviews                                   authentication.k8s.io/v1               false        TokenReview
localsubjectaccessreviews                      authorization.k8s.io/v1                true         LocalSubjectAccessReview
selfsubjectaccessreviews                       authorization.k8s.io/v1                false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io/v1                false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io/v1                false        SubjectAccessReview
horizontalpodautoscalers          hpa          autoscaling/v2                         true         HorizontalPodAutoscaler
cronjobs                          cj           batch/v1                               true         CronJob
jobs                                           batch/v1                               true         Job
certificatesigningrequests        csr          certificates.k8s.io/v1                 false        CertificateSigningRequest
leases                                         coordination.k8s.io/v1                 true         Lease
endpointslices                                 discovery.k8s.io/v1                    true         EndpointSlice
events                            ev           events.k8s.io/v1                       true         Event
flowschemas                                    flowcontrol.apiserver.k8s.io/v1beta2   false        FlowSchema
prioritylevelconfigurations                    flowcontrol.apiserver.k8s.io/v1beta2   false        PriorityLevelConfiguration
ingressclasses                                 networking.k8s.io/v1                   false        IngressClass
ingresses                         ing          networking.k8s.io/v1                   true         Ingress
networkpolicies                   netpol       networking.k8s.io/v1                   true         NetworkPolicy
runtimeclasses                                 node.k8s.io/v1                         false        RuntimeClass
poddisruptionbudgets              pdb          policy/v1                              true         PodDisruptionBudget
clusterrolebindings                            rbac.authorization.k8s.io/v1           false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io/v1           false        ClusterRole
rolebindings                                   rbac.authorization.k8s.io/v1           true         RoleBinding
roles                                          rbac.authorization.k8s.io/v1           true         Role
priorityclasses                   pc           scheduling.k8s.io/v1                   false        PriorityClass
csidrivers                                     storage.k8s.io/v1                      false        CSIDriver
csinodes                                       storage.k8s.io/v1                      false        CSINode
csistoragecapacities                           storage.k8s.io/v1                      true         CSIStorageCapacity
storageclasses                    sc           storage.k8s.io/v1                      false        StorageClass
volumeattachments                              storage.k8s.io/v1                      false        VolumeAttachment
```

<!-- cspell:enable -->

Para detalhes específicos de configuração, consulte
[Kubernetes Objects Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/k8sobjectsreceiver).

Como o processor usa a API do Kubernetes, ele precisa da permissão correta para
funcionar. Como service accounts são a única opção de autenticação, você deve
dar à service account o acesso adequado. Para qualquer objeto que quiser
coletar, você precisa garantir que o nome seja adicionado ao cluster role. Por
exemplo, se você quisesse coletar pods, o cluster role seria:

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-collector-opentelemetry-collector
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector-opentelemetry-collector
rules:
  - apiGroups:
      - ''
    resources:
      - pods
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector-opentelemetry-collector
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: otel-collector-opentelemetry-collector
subjects:
  - kind: ServiceAccount
    name: otel-collector-opentelemetry-collector
    namespace: default
```

## Prometheus Receiver

| Padrão de Implantação | Utilizável |
| --------------------- | ---------- |
| DaemonSet (agent)     | Sim        |
| Deployment (gateway)  | Sim        |
| Sidecar               | Não        |

O Prometheus é um formato de métricas comum tanto para o Kubernetes quanto para
serviços em execução no Kubernetes. O Prometheus receiver é um substituto
drop-in mínimo para a coleta dessas métricas. Ele suporta o conjunto completo de
opções do Prometheus
[`scrape_config`](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#scrape_config).

Há alguns recursos avançados do Prometheus que o receiver não suporta. O
receiver retorna um erro se o YAML/código de configuração contiver qualquer um
dos seguintes:

- `alert_config.alertmanagers`
- `alert_config.relabel_configs`
- `remote_read`
- `remote_write`
- `rule_files`

Para detalhes específicos de configuração, consulte
[Prometheus Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/prometheusreceiver).

O Prometheus receiver é
[Stateful](https://github.com/open-telemetry/opentelemetry-collector/blob/main/docs/standard-warnings.md#statefulness),
o que significa que há detalhes importantes a considerar ao usá-lo:

- O collector não pode escalar automaticamente o processo de scraping quando
  várias réplicas do collector estão em execução.
- Ao executar múltiplas réplicas do collector com a mesma configuração, ele fará
  scraping dos targets múltiplas vezes.
- Os usuários precisam configurar cada réplica com uma configuração de scraping
  diferente se quiserem distribuir manualmente o processo de scraping.

Para facilitar a configuração do Prometheus receiver, o OpenTelemetry Operator
inclui um componente opcional chamado
[Target Allocator](/docs/platforms/kubernetes/operator/target-allocator). Este
componente pode ser usado para informar a um collector quais endpoints
Prometheus ele deve fazer scraping.

Para mais informações sobre o design do receiver, consulte
[Design](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/receiver/prometheusreceiver/DESIGN.md).

## Host Metrics Receiver

| Padrão de Implantação | Utilizável                                                   |
| --------------------- | ------------------------------------------------------------ |
| DaemonSet (agent)     | Preferido                                                    |
| Deployment (gateway)  | Sim, mas coleta métricas apenas do nó em que está implantado |
| Sidecar               | Não                                                          |

O Host Metrics Receiver coleta métricas de um host usando uma variedade de
scrapers. Há alguma sobreposição com o
[Kubeletstats Receiver](#kubeletstats-receiver), então se você decidir usar
ambos, pode valer a pena desabilitar essas métricas duplicadas.

No Kubernetes, o receiver precisa de acesso ao volume `hostfs` para funcionar
corretamente. Se você estiver usando o
[OpenTelemetry Collector Helm chart](/docs/platforms/kubernetes/helm/collector/)
você pode usar o preset
[`hostMetrics`](/docs/platforms/kubernetes/helm/collector/#host-metrics-preset)
para começar.

Os scrapers disponíveis são:

| Scraper    | SOs Suportados         | Descrição                                            |
| ---------- | ---------------------- | ---------------------------------------------------- |
| cpu        | Todos exceto macOS[^1] | Métricas de utilização de CPU                        |
| disk       | Todos exceto macOS[^1] | Métricas de I/O de disco                             |
| load       | Todos                  | Métricas de carga de CPU                             |
| filesystem | Todos                  | Métricas de utilização do sistema de arquivos        |
| memory     | Todos                  | Métricas de utilização de memória                    |
| network    | Todos                  | Métricas de I/O de interface de rede e conexões TCP  |
| paging     | Todos                  | Métricas de utilização e I/O de paging/swap          |
| processes  | Linux, macOS           | Métricas de contagem de processos                    |
| process    | Linux, macOS, Windows  | Métricas de CPU, memória e I/O de disco por processo |

[^1]:
    Não suportado no macOS quando compilado sem cgo, que é o padrão para as
    imagens lançadas pelo Collector SIG.

Para detalhes específicos sobre quais métricas são coletadas e detalhes
específicos de configuração, consulte
[Host Metrics Receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/hostmetricsreceiver).

Se você precisar configurar o componente você mesmo, certifique-se de montar o
volume `hostfs` se quiser coletar as métricas do nó e não as do container.

```yaml
---
apiVersion: apps/v1
kind: DaemonSet
...
spec:
  ...
  template:
    ...
    spec:
      ...
      containers:
        - name: opentelemetry-collector
          ...
          volumeMounts:
            ...
            - name: hostfs
              mountPath: /hostfs
              readOnly: true
              mountPropagation: HostToContainer
      volumes:
        ...
        - name: hostfs
          hostPath:
            path: /
      ...
```

e então configure o Host Metrics Receiver para usar o `volumeMount`:

```yaml
receivers:
  hostmetrics:
    root_path: /hostfs
    collection_interval: 10s
    scrapers:
      cpu:
      load:
      memory:
      disk:
      filesystem:
      network:
```

Para mais detalhes sobre o uso do receiver em um container, consulte
[Collecting host metrics from inside a container (Linux only)](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/hostmetricsreceiver#collecting-host-metrics-from-inside-a-container-linux-only)
