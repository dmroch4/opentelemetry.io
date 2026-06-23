---
title: OpenTelemetry Operator para Kubernetes
linkTitle: Kubernetes Operator
description:
  Uma implementação de um Kubernetes Operator que gerencia collectors e
  auto-instrumentação de workloads usando bibliotecas de instrumentação
  OpenTelemetry.
aliases:
  - /docs/operator
  - /docs/k8s-operator
  - /docs/platforms/kubernetes-operator
redirects:
  - { from: /docs/operator/*, to: ':splat' }
  - { from: /docs/k8s-operator/*, to: ':splat' }
  - { from: /docs/platforms/kubernetes-operator/*, to: ':splat' }
default_lang_commit: 6bf06ddb9fc057dd6e8092f26d988ffe7b1af5ed
---

## Introdução

O
[OpenTelemetry Operator](https://github.com/open-telemetry/opentelemetry-operator)
é uma implementação de um
[Kubernetes Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).

O operator gerencia:

- [OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-collector)
- [auto-instrumentação dos workloads usando bibliotecas de instrumentação OpenTelemetry](https://github.com/open-telemetry/opentelemetry-operator#opentelemetry-auto-instrumentation-injection)

## Primeiros passos

Para instalar o operator em um cluster existente, certifique-se de ter o
[`cert-manager`](https://cert-manager.io/docs/installation/) instalado e
execute:

```bash
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```

Quando o deployment do `opentelemetry-operator` estiver pronto, crie uma
instância do OpenTelemetry Collector (otelcol), como:

```console
$ kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: simplest
spec:
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      memory_limiter:
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 15

    exporters:
      # NOTA: Antes da v0.86.0, use `logging` em vez de `debug`.
      debug: {}

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter]
          exporters: [debug]
EOF
```

> [!NOTE]
>
> Por padrão, o `opentelemetry-operator` usa a
> [imagem `opentelemetry-collector`](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector).
> Quando o operator é instalado usando
> [Helm charts](/docs/platforms/kubernetes/helm/), a
> [imagem `opentelemetry-collector-k8s`](https://github.com/open-telemetry/opentelemetry-collector-releases/pkgs/container/opentelemetry-collector-releases%2Fopentelemetry-collector-k8s)
> é usada. Se você precisar de um componente não encontrado nesses releases,
> pode ser necessário
> [compilar seu próprio collector](/docs/collector/extend/ocb/).

Para mais opções de configuração e para configurar a injeção de
auto-instrumentação dos workloads usando bibliotecas de instrumentação
OpenTelemetry, consulte
[OpenTelemetry Operator para Kubernetes](https://github.com/open-telemetry/opentelemetry-operator/blob/main/README.md).
