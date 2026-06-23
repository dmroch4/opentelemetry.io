---
title: OpenTelemetry Collector Chart
linkTitle: Collector Chart
# prettier-ignore
cSpell:ignore: filelog filelogreceiver hostmetricsreceiver kubelet kubeletstats kubeletstatsreceiver otlp_http sattributesprocessor sclusterreceiver sobjectsreceiver statefulset
default_lang_commit: 638af010dde026663ffa75f96c0d9310a09d6714
---

## Introdução

O [OpenTelemetry Collector](/docs/collector) é uma ferramenta importante para
monitorar um cluster Kubernetes e todos os serviços que operam dentro dele. Para
facilitar a instalação e o gerenciamento de uma implantação do collector em um
cluster Kubernetes, a comunidade OpenTelemetry criou o
[OpenTelemetry Collector Helm Chart](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-collector).
Este helm chart pode ser usado para instalar um collector como Deployment,
Daemonset ou Statefulset.

### Instalando o Chart

Para instalar o chart com o nome de release `my-opentelemetry-collector`,
execute os seguintes comandos:

```sh
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm install my-opentelemetry-collector open-telemetry/opentelemetry-collector \
   --set image.repository="otel/opentelemetry-collector-k8s" \
   --set mode=<daemonset|deployment|statefulset>
```

### Configuração

O OpenTelemetry Collector Chart requer que `mode` seja definido. `mode` pode ser
`daemonset`, `deployment` ou `statefulset`, dependendo do tipo de implantação
Kubernetes que seu caso de uso requer.

Quando instalado, o chart fornece alguns componentes padrão do collector para
você começar. Por padrão, a configuração do collector será:

```yaml
exporters:
  # NOTE: Prior to v0.86.0 use `logging` instead of `debug`.
  debug: {}
extensions:
  health_check: {}
processors:
  batch: {}
  memory_limiter:
    check_interval: 5s
    limit_percentage: 80
    spike_limit_percentage: 25
receivers:
  jaeger:
    protocols:
      grpc:
        endpoint: ${env:MY_POD_IP}:14250
      thrift_compact:
        endpoint: ${env:MY_POD_IP}:6831
      thrift_http:
        endpoint: ${env:MY_POD_IP}:14268
  otlp:
    protocols:
      grpc:
        endpoint: ${env:MY_POD_IP}:4317
      http:
        endpoint: ${env:MY_POD_IP}:4318
  prometheus:
    config:
      scrape_configs:
        - job_name: opentelemetry-collector
          scrape_interval: 10s
          static_configs:
            - targets:
                - ${env:MY_POD_IP}:8888
  zipkin:
    endpoint: ${env:MY_POD_IP}:9411
service:
  extensions:
    - health_check
  pipelines:
    logs:
      exporters:
        - debug
      processors:
        - memory_limiter
        - batch
      receivers:
        - otlp
    metrics:
      exporters:
        - debug
      processors:
        - memory_limiter
        - batch
      receivers:
        - otlp
        - prometheus
    traces:
      exporters:
        - debug
      processors:
        - memory_limiter
        - batch
      receivers:
        - otlp
        - jaeger
        - zipkin
  telemetry:
    metrics:
      address: ${env:MY_POD_IP}:8888
```

O chart também habilitará ports com base nos receivers padrão. A configuração
padrão pode ser removida definindo o valor como `null` no seu `values.yaml`. Os
ports também podem ser desabilitados no `values.yaml`.

Você pode adicionar/modificar qualquer parte da configuração usando a seção
`config` no seu `values.yaml`. Ao alterar um pipeline, você deve listar
explicitamente todos os componentes que estão no pipeline, incluindo quaisquer
componentes padrão.

Por exemplo, para desabilitar os pipelines de métricas e logs e os receivers
não-otlp:

```yaml
config:
  receivers:
    jaeger: null
    prometheus: null
    zipkin: null
  service:
    pipelines:
      traces:
        receivers:
          - otlp
      metrics: null
      logs: null
ports:
  jaeger-compact:
    enabled: false
  jaeger-thrift:
    enabled: false
  jaeger-grpc:
    enabled: false
  zipkin:
    enabled: false
```

Todas as opções de configuração (com comentários) disponíveis no chart podem ser
visualizadas no seu
[arquivo values.yaml](https://github.com/open-telemetry/opentelemetry-helm-charts/blob/main/charts/opentelemetry-collector/values.yaml).

### Presets

Muitos dos componentes importantes que o OpenTelemetry Collector usa para
monitorar o Kubernetes requerem configuração especial na própria implantação
Kubernetes do Collector. Para facilitar o uso desses componentes, o
OpenTelemetry Collector Chart vem com alguns presets que, quando habilitados,
cuidam da configuração complexa para esses componentes importantes.

Os presets devem ser usados como ponto de partida. Eles configuram
funcionalidades básicas, mas ricas, para seus componentes relacionados. Se seu
caso de uso requer configuração extra desses componentes, é recomendado NÃO usar
o preset e, em vez disso, configurar manualmente o componente e tudo o que ele
requer (volumes, RBAC, etc.).

#### Preset de Coleta de Logs

O OpenTelemetry Collector pode ser usado para coletar logs enviados para a saída
padrão por containers do Kubernetes.

Este recurso está desabilitado por padrão. Ele possui os seguintes requisitos
para ser habilitado com segurança:

- Requer que o
  [Filelog receiver](/docs/platforms/kubernetes/collector/components/#filelog-receiver)
  esteja incluído na imagem do Collector, como a
  [distribuição Contrib do Collector](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector-contrib).
- Embora não seja um requisito estrito, é recomendado que este preset seja usado
  com `mode=daemonset`. O `filelogreceiver` só conseguirá coletar logs do nó em
  que o Collector está em execução e múltiplos Collectors configurados no mesmo
  nó produzirão dados duplicados.

Para habilitar este recurso, defina a propriedade
`presets.logsCollection.enabled` como `true`. Quando habilitado, o chart
adicionará um `filelogreceiver` ao pipeline `logs`. Este receiver está
configurado para ler os arquivos onde o runtime de containers do Kubernetes
grava a saída do console de todos os containers (`/var/log/pods/*/*/*.log`).

Aqui está um exemplo de `values.yaml`:

```yaml
mode: daemonset
presets:
  logsCollection:
    enabled: true
```

O pipeline de logs padrão do chart usa o exporter `debug`. Combinado com o
`filelogreceiver` do preset `logsCollection`, é fácil acidentalmente realimentar
os logs exportados de volta ao collector, o que pode causar uma "explosão de
logs".

Para evitar o loop, a configuração padrão do receiver exclui os próprios logs do
collector. Se você quiser incluir os logs do collector, certifique-se de
substituir o exporter `debug` por um exporter que não envie logs para a saída
padrão do collector.

Aqui está um exemplo de `values.yaml` que substitui o exporter padrão `debug` no
pipeline `logs` por um exporter `otlp_http` que envia os logs do container para
o endpoint `https://example.com:55681`. Ele também usa
`presets.logsCollection.includeCollectorLogs` para informar ao preset que
habilite a coleta dos logs do collector.

```yaml
mode: daemonset

presets:
  logsCollection:
    enabled: true
    includeCollectorLogs: true

config:
  exporters:
    otlp_http:
      endpoint: https://example.com:55681
  service:
    pipelines:
      logs:
        exporters:
          - otlp_http
```

#### Preset de Atributos do Kubernetes

O OpenTelemetry Collector pode ser configurado para adicionar metadados do
Kubernetes, como `k8s.pod.name`, `k8s.namespace.name` e `k8s.node.name`, a logs,
métricas e traces. É altamente recomendado usar o preset, ou habilitar o
`k8sattributesprocessor` manualmente.

Devido a considerações de RBAC, este recurso está desabilitado por padrão. Ele
possui os seguintes requisitos:

- Requer que o
  [Kubernetes Attributes processor](/docs/platforms/kubernetes/collector/components/#kubernetes-attributes-processor)
  esteja incluído na imagem do Collector, como a
  [distribuição Contrib do Collector](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector-contrib).

Para habilitar este recurso, defina a propriedade
`presets.kubernetesAttributes.enabled` como `true`. Quando habilitado, o chart
adicionará os papéis RBAC necessários ao ClusterRole e adicionará um
`k8sattributesprocessor` a cada pipeline habilitado.

Aqui está um exemplo de `values.yaml`:

```yaml
mode: daemonset
presets:
  kubernetesAttributes:
    enabled: true
```

#### Preset de Métricas do Kubelet

O OpenTelemetry Collector pode ser configurado para coletar métricas de nós,
pods e containers do servidor de API em um kubelet.

Este recurso está desabilitado por padrão. Ele possui os seguintes requisitos:

- Requer que o
  [Kubeletstats receiver](/docs/platforms/kubernetes/collector/components/#kubeletstats-receiver)
  esteja incluído na imagem do Collector, como a
  [distribuição Contrib do Collector](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector-contrib).
- Embora não seja um requisito estrito, é recomendado que este preset seja usado
  com `mode=daemonset`. O `kubeletstatsreceiver` só conseguirá coletar métricas
  do nó em que o Collector está em execução e múltiplos Collectors configurados
  no mesmo nó produzirão dados duplicados.

Para habilitar este recurso, defina a propriedade
`presets.kubeletMetrics.enabled` como `true`. Quando habilitado, o chart
adicionará os papéis RBAC necessários ao ClusterRole e adicionará um
`kubeletstatsreceiver` ao pipeline de métricas.

Aqui está um exemplo de `values.yaml`:

```yaml
mode: daemonset
presets:
  kubeletMetrics:
    enabled: true
```

#### Preset de Métricas do Cluster

O OpenTelemetry Collector pode ser configurado para coletar métricas a nível de
cluster do servidor de API do Kubernetes. Essas métricas incluem muitas das
métricas coletadas pelo Kube State Metrics.

Este recurso está desabilitado por padrão. Ele possui os seguintes requisitos:

- Requer que o
  [Kubernetes Cluster receiver](/docs/platforms/kubernetes/collector/components/#kubernetes-cluster-receiver)
  esteja incluído na imagem do Collector, como a
  [distribuição Contrib do Collector](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector-contrib).
- Embora não seja um requisito estrito, é recomendado que este preset seja usado
  com `mode=deployment` ou `mode=statefulset` com uma única réplica. Executar o
  `k8sclusterreceiver` em múltiplos Collectors produzirá dados duplicados.

Para habilitar este recurso, defina a propriedade
`presets.clusterMetrics.enabled` como `true`. Quando habilitado, o chart
adicionará os papéis RBAC necessários ao ClusterRole e adicionará um
`k8sclusterreceiver` ao pipeline de métricas.

Aqui está um exemplo de `values.yaml`:

```yaml
mode: deployment
replicaCount: 1
presets:
  clusterMetrics:
    enabled: true
```

#### Preset de Eventos do Kubernetes

O OpenTelemetry Collector pode ser configurado para coletar eventos do
Kubernetes.

Este recurso está desabilitado por padrão. Ele possui os seguintes requisitos:

- Requer que o
  [Kubernetes Objects receiver](/docs/platforms/kubernetes/collector/components/#kubernetes-objects-receiver)
  esteja incluído na imagem do Collector, como a
  [distribuição Contrib do Collector](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector-contrib).
- Embora não seja um requisito estrito, é recomendado que este preset seja usado
  com `mode=deployment` ou `mode=statefulset` com uma única réplica. Executar o
  `k8sclusterreceiver` em múltiplos Collectors produzirá dados duplicados.

Para habilitar este recurso, defina a propriedade
`presets.kubernetesEvents.enabled` como `true`. Quando habilitado, o chart
adicionará os papéis RBAC necessários ao ClusterRole e adicionará um
`k8sobjectsreceiver` ao pipeline de logs configurado para coletar apenas
eventos.

Aqui está um exemplo de `values.yaml`:

```yaml
mode: deployment
replicaCount: 1
presets:
  kubernetesEvents:
    enabled: true
```

#### Preset de Métricas do Host

O OpenTelemetry Collector pode ser configurado para coletar métricas do host em
nós do Kubernetes.

Este recurso está desabilitado por padrão. Ele possui os seguintes requisitos:

- Requer que o
  [Host Metrics receiver](/docs/platforms/kubernetes/collector/components/#host-metrics-receiver)
  esteja incluído na imagem do Collector, como a
  [distribuição Contrib do Collector](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector-contrib).
- Embora não seja um requisito estrito, é recomendado que este preset seja usado
  com `mode=daemonset`. O `hostmetricsreceiver` só conseguirá coletar métricas
  do nó em que o Collector está em execução e múltiplos Collectors configurados
  no mesmo nó produzirão dados duplicados.

Para habilitar este recurso, defina a propriedade `presets.hostMetrics.enabled`
como `true`. Quando habilitado, o chart adicionará os volumes e volumeMounts
necessários e adicionará um `hostmetricsreceiver` ao pipeline de métricas. Por
padrão, as métricas serão coletadas a cada 10 segundos e os seguintes scrapers
são habilitados:

- cpu
- load
- memory
- disk
- filesystem[^1]
- network

Aqui está um exemplo de `values.yaml`:

```yaml
mode: daemonset
presets:
  hostMetrics:
    enabled: true
```

[^1]:
    Devido a alguma sobreposição com o preset `kubeletMetrics`, alguns tipos de
    sistema de arquivos e pontos de montagem são excluídos por padrão.
