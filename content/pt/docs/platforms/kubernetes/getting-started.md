---
title: Primeiros Passos
weight: 1
# prettier-ignore
cSpell:ignore: filelog filelogreceiver kubelet kubeletstats kubeletstatsreceiver sattributes sattributesprocessor sclusterreceiver sobjectsreceiver
default_lang_commit: 4cb7e22f1e45d17854b309efc730499880aa7197
---

Esta página guiará você pela maneira mais rápida de começar a monitorar seu
cluster Kubernetes usando OpenTelemetry. O foco será na coleta de métricas e
logs para clusters, nós, pods e containers do Kubernetes, além de habilitar o
cluster para suportar serviços que emitem dados OTLP.

Se você deseja ver o OpenTelemetry em ação com Kubernetes, o melhor lugar para
começar é o [OpenTelemetry Demo](/docs/demo/kubernetes-deployment/). A demo tem
como objetivo ilustrar a implementação do OpenTelemetry, mas não se destina a
ser um exemplo de como monitorar o próprio Kubernetes. Ao terminar este guia,
pode ser um experimento interessante instalar a demo e ver como todo o
monitoramento responde a uma carga de trabalho ativa.

Se você deseja começar a migrar do Prometheus para o OpenTelemetry, ou se está
interessado em usar o OpenTelemetry Collector para coletar métricas do
Prometheus, consulte
[Prometheus Receiver](/docs/platforms/kubernetes/collector/components/#prometheus-receiver).

## Visão Geral

O Kubernetes expõe muita telemetria importante de diversas maneiras diferentes.
Ele possui logs, eventos, métricas para muitos objetos diferentes e os dados
gerados por suas cargas de trabalho.

Para coletar todos esses dados, utilizaremos o
[OpenTelemetry Collector](/docs/collector/). O collector possui muitas
ferramentas diferentes à sua disposição, que permitem coletar todos esses dados
de forma eficiente e enriquecê-los de maneiras significativas.

Para coletar todos os dados, precisaremos de duas instalações do collector: uma
como [Daemonset](/docs/collector/deploy/agent/) e outra como
[Deployment](/docs/collector/deploy/gateway/). A instalação do collector como
Daemonset será usada para coletar telemetria emitida por serviços, logs e
métricas de nós, pods e containers. A instalação como deployment será usada para
coletar métricas do cluster e eventos.

Para instalar o collector, utilizaremos o
[OpenTelemetry Collector Helm chart](/docs/platforms/kubernetes/helm/collector/),
que vem com algumas opções de configuração que facilitarão a configuração do
collector. Se você não está familiarizado com Helm, consulte
[o site do projeto Helm](https://helm.sh/). Se você tem interesse em usar um
operador Kubernetes, consulte
[OpenTelemetry Operator](/docs/platforms/kubernetes/operator/), mas este guia se
concentrará no Helm chart.

## Preparação

Este guia assumirá o uso de um [cluster Kind](https://kind.sigs.k8s.io/), mas
você é livre para usar qualquer cluster Kubernetes que considerar adequado.

Supondo que você já tenha o
[Kind instalado](https://kind.sigs.k8s.io/#installation-and-usage), crie um novo
cluster kind:

```sh
kind create cluster
```

Supondo que você já tenha o
[Helm instalado](https://helm.sh/docs/intro/install/), adicione o OpenTelemetry
Collector Helm chart para que ele possa ser instalado posteriormente:

```sh
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
```

## Collector como Daemonset

O primeiro passo para coletar telemetria do Kubernetes é implantar uma instância
do OpenTelemetry Collector como daemonset, para coletar telemetria relacionada
aos nós e às cargas de trabalho executadas nesses nós. Um daemonset é usado para
garantir que essa instância do collector seja instalada em todos os nós. Cada
instância do collector no daemonset coletará dados apenas do nó em que está
sendo executada.

Esta instância do collector utilizará os seguintes componentes:

- [OTLP Receiver](https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver/otlpreceiver):
  para coletar traces, métricas e logs de aplicações.
- [Kubernetes Attributes Processor](/docs/platforms/kubernetes/collector/components/#kubernetes-attributes-processor):
  para adicionar metadados do Kubernetes à telemetria de aplicações recebida.
- [Kubeletstats Receiver](/docs/platforms/kubernetes/collector/components/#kubeletstats-receiver):
  para extrair métricas de nós, pods e containers do servidor de API em um
  kubelet.
- [Filelog Receiver](/docs/platforms/kubernetes/collector/components/#filelog-receiver):
  para coletar logs do Kubernetes e logs de aplicações gravados em
  stdout/stderr.

Vamos analisar cada um deles.

### OTLP Receiver

O
[OTLP Receiver](https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver/otlpreceiver)
é a melhor solução para coletar traces, métricas e logs no
[formato OTLP](/docs/specs/otel/protocol/). Se você está emitindo telemetria de
aplicação em outro formato, há uma boa chance de que
[o Collector tenha um receiver para ele](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver),
mas neste tutorial assumiremos que a telemetria está no formato OTLP.

Embora não seja um requisito, é uma prática comum que as aplicações executadas
em um nó emitam seus traces, métricas e logs para um collector em execução no
mesmo nó. Isso mantém as interações de rede simples e permite a correlação fácil
de metadados do Kubernetes usando o processador `k8sattributes`.

### Kubernetes Attributes Processor

O
[Kubernetes Attributes Processor](/docs/platforms/kubernetes/collector/components/#kubernetes-attributes-processor)
é um componente altamente recomendado em qualquer collector que receba
telemetria de pods do Kubernetes. Este processor descobre automaticamente os
pods do Kubernetes, extrai seus metadados como nome do pod ou nome do nó, e
adiciona os metadados extraídos a spans, métricas e logs como atributos de
recurso. Por adicionar contexto do Kubernetes à sua telemetria, o Kubernetes
Attributes Processor permite correlacionar os traces, métricas e logs da sua
aplicação com a telemetria do Kubernetes, como métricas e traces de pods.

### Kubeletstats Receiver

O
[Kubeletstats Receiver](/docs/platforms/kubernetes/collector/components/#kubeletstats-receiver)
é o receiver que coleta métricas sobre o nó. Ele coletará métricas como uso de
memória de containers, uso de CPU de pods e erros de rede de nós. Toda a
telemetria inclui metadados do Kubernetes como nome do pod ou nome do nó. Como
estamos usando o Kubernetes Attributes Processor, conseguiremos correlacionar
nossos traces, métricas e logs de aplicação com as métricas produzidas pelo
Kubeletstats Receiver.

### Filelog Receiver

O
[Filelog Receiver](/docs/platforms/kubernetes/collector/components/#filelog-receiver)
coletará logs gravados em stdout/stderr acompanhando os logs que o Kubernetes
grava em `/var/log/pods/*/*/*.log`. Como a maioria dos coletores de logs, o
filelog receiver fornece um conjunto robusto de ações que permitem analisar o
arquivo da forma que você precisar.

Um dia você pode precisar configurar um Filelog Receiver por conta própria, mas
para este guia o OpenTelemetry Helm Chart cuidará de toda a configuração
complexa para você. Além disso, ele extrairá metadados úteis do Kubernetes com
base no nome do arquivo. Como estamos usando o Kubernetes Attributes Processor,
conseguiremos correlacionar os traces, métricas e logs de aplicação com os logs
produzidos pelo Filelog Receiver.

---

O OpenTelemetry Collector Helm chart facilita a configuração de todos esses
componentes em uma instalação do collector como daemonset. Ele também cuidará de
todos os detalhes específicos do Kubernetes, como RBAC, mounts e ports do host.

Uma ressalva — o chart não envia os dados para nenhum backend por padrão. Se
você quiser realmente usar seus dados no seu backend favorito, precisará
configurar um exporter você mesmo.

O seguinte `values.yaml` é o que utilizaremos:

```yaml
mode: daemonset

image:
  repository: otel/opentelemetry-collector-k8s

presets:
  # enables the k8sattributesprocessor and adds it to the traces, metrics, and logs pipelines
  kubernetesAttributes:
    enabled: true
  # enables the kubeletstatsreceiver and adds it to the metrics pipelines
  kubeletMetrics:
    enabled: true
  # Enables the filelogreceiver and adds it to the logs pipelines
  logsCollection:
    enabled: true
## The chart only includes the debugexporter by default
## If you want to send your data somewhere you need to
## configure an exporter, such as the otlp exporter
# config:
#   exporters:
#     otlp:
#       endpoint: "<SOME BACKEND>"
#   service:
#     pipelines:
#       traces:
#         exporters: [ otlp ]
#       metrics:
#         exporters: [ otlp ]
#       logs:
#         exporters: [ otlp ]
```

Para usar este `values.yaml` com o chart, salve-o no local de arquivo de sua
preferência e execute o seguinte comando para instalar o chart:

```sh
helm install otel-collector open-telemetry/opentelemetry-collector --values <path where you saved the chart>
```

Agora você deve ter uma instalação do OpenTelemetry Collector como daemonset em
execução no seu cluster, coletando telemetria de cada nó!

## Collector como Deployment

O próximo passo para coletar telemetria do Kubernetes é implantar uma instância
do Collector como deployment, para coletar telemetria relacionada ao cluster
como um todo. Um deployment com exatamente uma réplica garante que não
produziremos dados duplicados.

Esta instância do Collector utilizará os seguintes componentes:

- [Kubernetes Cluster Receiver](/docs/platforms/kubernetes/collector/components/#kubernetes-cluster-receiver):
  para coletar métricas a nível de cluster e eventos de entidade.
- [Kubernetes Objects Receiver](/docs/platforms/kubernetes/collector/components/#kubernetes-objects-receiver):
  para coletar objetos, como eventos, do servidor de API do Kubernetes.

Vamos analisar cada um deles.

### Kubernetes Cluster Receiver

O
[Kubernetes Cluster Receiver](/docs/platforms/kubernetes/collector/components/#kubernetes-cluster-receiver)
é a solução do Collector para coletar métricas sobre o estado do cluster como um
todo. Este receiver pode coletar métricas sobre condições de nós, fases de pods,
reinicializações de containers, deployments disponíveis e desejados, e muito
mais.

### Kubernetes Objects Receiver

O
[Kubernetes Objects Receiver](/docs/platforms/kubernetes/collector/components/#kubernetes-objects-receiver)
é a solução do Collector para coletar objetos do Kubernetes como logs. Embora
qualquer objeto possa ser coletado, um caso de uso comum e importante é coletar
eventos do Kubernetes.

---

O OpenTelemetry Collector Helm chart simplifica a configuração de todos esses
componentes em uma instalação do Collector como deployment. Ele também cuidará
de todos os detalhes específicos do Kubernetes, como RBAC e mounts.

Uma ressalva — o chart não envia os dados para nenhum backend por padrão. Se
você quiser realmente usar seus dados no seu backend preferido, precisará
configurar um exporter você mesmo.

O seguinte `values.yaml` é o que utilizaremos:

```yaml
mode: deployment

image:
  repository: otel/opentelemetry-collector-k8s

# We only want one of these collectors - any more and we'd produce duplicate data
replicaCount: 1

presets:
  # enables the k8sclusterreceiver and adds it to the metrics pipelines
  clusterMetrics:
    enabled: true
  # enables the k8sobjectsreceiver to collect events only and adds it to the logs pipelines
  kubernetesEvents:
    enabled: true
## The chart only includes the debugexporter by default
## If you want to send your data somewhere you need to
## configure an exporter, such as the otlp exporter
# config:
# exporters:
#   otlp:
#     endpoint: "<SOME BACKEND>"
# service:
#   pipelines:
#     traces:
#       exporters: [ otlp ]
#     metrics:
#       exporters: [ otlp ]
#     logs:
#       exporters: [ otlp ]
```

Para usar este `values.yaml` com o chart, salve-o no local de arquivo de sua
preferência e execute o seguinte comando para instalar o chart:

```sh
helm install otel-collector-cluster open-telemetry/opentelemetry-collector --values <path where you saved the chart>
```

Agora você deve ter uma instalação do collector como deployment em execução no
seu cluster, coletando métricas e eventos do cluster!
