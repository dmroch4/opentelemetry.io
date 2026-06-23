---
title: Target Allocator
description:
  Uma ferramenta para distribuir alvos do PrometheusReceiver em todas as
  instâncias do Collector implantadas
cSpell:ignore: labeldrop labelmap statefulset
default_lang_commit: fe623719bc24346e9dcd77e9769026cf1c720cc5
---

O OpenTelemetry Operator vem com um componente opcional, o
[Target Allocator](https://github.com/open-telemetry/opentelemetry-operator/tree/main/cmd/otel-allocator)
(TA). Em resumo, o TA é um mecanismo para desacoplar as funções de descoberta de
serviços e coleta de métricas do Prometheus, para que possam ser escaladas
independentemente. O Collector gerencia métricas Prometheus sem precisar
instalar o Prometheus. O TA gerencia a configuração do Prometheus Receiver do
Collector.

O TA serve duas funções:

1. Distribuição uniforme de alvos Prometheus entre um pool de Collectors
2. Descoberta de Custom Resources Prometheus

## Primeiros Passos

Ao criar um Custom Resource (CR) OpenTelemetryCollector e definir o TA como
habilitado, o Operator criará um novo deployment e service para servir diretivas
`http_sd_config` específicas para cada pod do Collector como parte desse CR.
Também alterará a configuração do Prometheus receiver no CR para que use o
[http_sd_config](https://prometheus.io/docs/prometheus/latest/http_sd/) do TA. O
exemplo a seguir mostra como começar a usar o Target Allocator:

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: collector-with-ta
spec:
  mode: statefulset
  targetAllocator:
    enabled: true
  config: |
    receivers:
      prometheus:
        config:
          scrape_configs:
          - job_name: 'otel-collector'
            scrape_interval: 10s
            static_configs:
            - targets: [ '0.0.0.0:8888' ]
            metric_relabel_configs:
            - action: labeldrop
              regex: (id|name)
              replacement: $$1
            - action: labelmap
              regex: label_(.+)
              replacement: $$1

    exporters:
      # NOTA: Antes da v0.86.0, use `logging` em vez de `debug`.
      debug:

    service:
      pipelines:
        metrics:
          receivers: [prometheus]
          processors: []
          exporters: [debug]
```

Por trás dos panos, o OpenTelemetry Operator converterá a configuração do
Collector após a reconciliação para o seguinte:

```yaml
receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: otel-collector
          scrape_interval: 10s
          http_sd_configs:
            - url: http://collector-with-ta-targetallocator:80/jobs/otel-collector/targets?collector_id=$POD_NAME
          metric_relabel_configs:
            - action: labeldrop
              regex: (id|name)
              replacement: $$1
            - action: labelmap
              regex: label_(.+)
              replacement: $$1

exporters:
  debug:

service:
  pipelines:
    metrics:
      receivers: [prometheus]
      processors: []
      exporters: [debug]
```

Observe como o Operator remove quaisquer configurações de descoberta de serviços
existentes (por exemplo, `static_configs`, `file_sd_configs`, etc.) da seção
`scrape_configs` e adiciona uma configuração `http_sd_configs` apontando para
uma instância do Target Allocator provisionada por ele.

Para informações mais detalhadas sobre o TargetAllocator, consulte
[TargetAllocator](https://github.com/open-telemetry/opentelemetry-operator/tree/main/cmd/otel-allocator).
