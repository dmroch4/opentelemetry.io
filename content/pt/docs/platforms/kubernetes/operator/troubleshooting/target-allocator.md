---
title: Target Allocator
cSpell:ignore: bleh targetallocator
default_lang_commit: 58e684763e8dd50a07ef5fbf428303973025428a
---

Se você habilitou a descoberta de serviço do
[Target Allocator](/docs/platforms/kubernetes/operator/target-allocator/) no
[OpenTelemetry Operator](/docs/platforms/kubernetes/operator/) e o Target
Allocator está falhando em descobrir targets de scraping, há algumas etapas de
solução de problemas que você pode executar para ajudá-lo a entender o que está
acontecendo e restaurar a operação normal.

## Etapas de solução de problemas

### Você implantou todos os seus recursos no Kubernetes?

Como primeiro passo, certifique-se de que implantou todos os recursos relevantes
no seu cluster Kubernetes.

### Você sabe se as métricas estão sendo coletadas?

Após implantar todos os seus recursos no Kubernetes, certifique-se de que o
Target Allocator está descobrindo targets de scraping dos seus
[`ServiceMonitor`](https://prometheus-operator.dev/docs/getting-started/design/#servicemonitor)(s)
ou [PodMonitor][]s.

Suponha que você tenha esta definição de `ServiceMonitor`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sm-example
  namespace: opentelemetry
  labels:
    app.kubernetes.io/name: py-prometheus-app
    release: prometheus
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    matchNames:
      - opentelemetry
  endpoints:
    - port: prom
      path: /metrics
    - port: py-client-port
      interval: 15s
    - port: py-server-port
```

esta definição de `Service`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: py-prometheus-app
  namespace: opentelemetry
  labels:
    app: my-app
    app.kubernetes.io/name: py-prometheus-app
spec:
  selector:
    app: my-app
    app.kubernetes.io/name: py-prometheus-app
  ports:
    - name: prom
      port: 8080
```

e esta definição de `OpenTelemetryCollector`:

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otelcol
  namespace: opentelemetry
spec:
  mode: statefulset
  targetAllocator:
    enabled: true
    serviceAccount: opentelemetry-targetallocator-sa
    prometheusCR:
      enabled: true
      podMonitorSelector: {}
      serviceMonitorSelector: {}
  config:
    receivers:
      otlp:
        protocols:
          grpc: {}
          http: {}
      prometheus:
        config:
          scrape_configs:
            - job_name: 'otel-collector'
              scrape_interval: 10s
              static_configs:
                - targets: ['0.0.0.0:8888']
    exporters:
      debug:
        verbosity: detailed

    service:
      pipelines:
        traces:
          receivers: [otlp]
          exporters: [debug]
        metrics:
          receivers: [otlp, prometheus]
          exporters: [debug]
        logs:
          receivers: [otlp]
          exporters: [debug]
```

Primeiro, configure um `port-forward` no Kubernetes para expor o serviço do
Target Allocator:

```shell
kubectl port-forward svc/otelcol-targetallocator -n opentelemetry 8080:80
```

Onde `otelcol-targetallocator` é o valor de `metadata.name` no seu CR
`OpenTelemetryCollector` concatenado com o sufixo `-targetallocator`, e
`opentelemetry` é o namespace para o qual o CR `OpenTelemetryCollector` está
implantado.

> [!TIP]
>
> Você também pode obter o nome do serviço executando
>
> ```shell
> kubectl get svc -l app.kubernetes.io/component=opentelemetry-targetallocator -n <namespace>
> ```

Em seguida, obtenha uma lista de jobs registrados com o Target Allocator:

```shell
curl localhost:8080/jobs | jq
```

Sua saída de exemplo deve ser assim:

```json
{
  "serviceMonitor/opentelemetry/sm-example/1": {
    "_link": "/jobs/serviceMonitor%2Fopentelemetry%2Fsm-example%2F1/targets"
  },
  "serviceMonitor/opentelemetry/sm-example/2": {
    "_link": "/jobs/serviceMonitor%2Fopentelemetry%2Fsm-example%2F2/targets"
  },
  "otel-collector": {
    "_link": "/jobs/otel-collector/targets"
  },
  "serviceMonitor/opentelemetry/sm-example/0": {
    "_link": "/jobs/serviceMonitor%2Fopentelemetry%2Fsm-example%2F0/targets"
  },
  "podMonitor/opentelemetry/pm-example/0": {
    "_link": "/jobs/podMonitor%2Fopentelemetry%2Fpm-example%2F0/targets"
  }
}
```

Onde `serviceMonitor/opentelemetry/sm-example/0` representa uma das portas do
`Service` que o `ServiceMonitor` capturou:

- `opentelemetry` é o namespace no qual o recurso `ServiceMonitor` reside.
- `sm-example` é o nome do `ServiceMonitor`.
- `0` é um dos endpoints de porta correspondente entre o `ServiceMonitor` e o
  `Service`.

De forma similar, o `PodMonitor` aparece como
`podMonitor/opentelemetry/pm-example/0` na saída do `curl`.

Estas são boas notícias, pois nos dizem que a descoberta de configuração de
scraping está funcionando!

Você também pode estar se perguntando sobre a entrada `otel-collector`. Isso
está acontecendo porque `spec.config.receivers.prometheusReceiver` no recurso
`OpenTelemetryCollector` (chamado `otel-collector`) tem auto-scraping
habilitado:

```yaml
prometheus:
  config:
    scrape_configs:
      - job_name: 'otel-collector'
        scrape_interval: 10s
        static_configs:
          - targets: ['0.0.0.0:8888']
```

Podemos dar uma olhada mais profunda em
`serviceMonitor/opentelemetry/sm-example/0`, para ver quais targets de scraping
estão sendo capturados executando `curl` contra o valor da saída `_link` acima:

```shell
curl localhost:8080/jobs/serviceMonitor%2Fopentelemetry%2Fsm-example%2F0/targets | jq
```

Saída de exemplo:

```json
{
  "otelcol-collector-0": {
    "_link": "/jobs/serviceMonitor%2Fopentelemetry%2Fsm-example%2F0/targets?collector_id=otelcol-collector-0",
    "targets": [
      {
        "targets": ["10.244.0.11:8080"],
        "labels": {
          "__meta_kubernetes_endpointslice_port_name": "prom",
          "__meta_kubernetes_pod_labelpresent_app_kubernetes_io_name": "true",
          "__meta_kubernetes_endpointslice_port_protocol": "TCP",
          "__meta_kubernetes_endpointslice_address_target_name": "py-prometheus-app-575cfdd46-nfttj",
          "__meta_kubernetes_endpointslice_annotation_endpoints_kubernetes_io_last_change_trigger_time": "2024-06-21T20:01:37Z",
          "__meta_kubernetes_endpointslice_labelpresent_app_kubernetes_io_name": "true",
          "__meta_kubernetes_pod_name": "py-prometheus-app-575cfdd46-nfttj",
          "__meta_kubernetes_pod_controller_name": "py-prometheus-app-575cfdd46",
          "__meta_kubernetes_pod_label_app_kubernetes_io_name": "py-prometheus-app",
          "__meta_kubernetes_endpointslice_address_target_kind": "Pod",
          "__meta_kubernetes_pod_node_name": "otel-target-allocator-talk-control-plane",
          "__meta_kubernetes_pod_labelpresent_pod_template_hash": "true",
          "__meta_kubernetes_endpointslice_label_kubernetes_io_service_name": "py-prometheus-app",
          "__meta_kubernetes_endpointslice_annotationpresent_endpoints_kubernetes_io_last_change_trigger_time": "true",
          "__meta_kubernetes_service_name": "py-prometheus-app",
          "__meta_kubernetes_pod_ready": "true",
          "__meta_kubernetes_pod_labelpresent_app": "true",
          "__meta_kubernetes_pod_controller_kind": "ReplicaSet",
          "__meta_kubernetes_endpointslice_labelpresent_app": "true",
          "__meta_kubernetes_pod_container_image": "otel-target-allocator-talk:0.1.0-py-prometheus-app",
          "__address__": "10.244.0.11:8080",
          "__meta_kubernetes_service_label_app_kubernetes_io_name": "py-prometheus-app",
          "__meta_kubernetes_pod_uid": "495d47ee-9a0e-49df-9b41-fe9e6f70090b",
          "__meta_kubernetes_endpointslice_port": "8080",
          "__meta_kubernetes_endpointslice_label_endpointslice_kubernetes_io_managed_by": "endpointslice-controller.k8s.io",
          "__meta_kubernetes_endpointslice_label_app": "my-app",
          "__meta_kubernetes_service_labelpresent_app_kubernetes_io_name": "true",
          "__meta_kubernetes_pod_host_ip": "172.24.0.2",
          "__meta_kubernetes_namespace": "opentelemetry",
          "__meta_kubernetes_endpointslice_endpoint_conditions_serving": "true",
          "__meta_kubernetes_endpointslice_labelpresent_kubernetes_io_service_name": "true",
          "__meta_kubernetes_endpointslice_endpoint_conditions_ready": "true",
          "__meta_kubernetes_service_annotation_kubectl_kubernetes_io_last_applied_configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Service\",\"metadata\":{\"annotations\":{},\"labels\":{\"app\":\"my-app\",\"app.kubernetes.io/name\":\"py-prometheus-app\"},\"name\":\"py-prometheus-app\",\"namespace\":\"opentelemetry\"},\"spec\":{\"ports\":[{\"name\":\"prom\",\"port\":8080}],\"selector\":{\"app\":\"my-app\",\"app.kubernetes.io/name\":\"py-prometheus-app\"}}}\n",
          "__meta_kubernetes_endpointslice_endpoint_conditions_terminating": "false",
          "__meta_kubernetes_pod_container_port_protocol": "TCP",
          "__meta_kubernetes_pod_phase": "Running",
          "__meta_kubernetes_pod_container_name": "my-app",
          "__meta_kubernetes_pod_container_port_name": "prom",
          "__meta_kubernetes_pod_ip": "10.244.0.11",
          "__meta_kubernetes_service_annotationpresent_kubectl_kubernetes_io_last_applied_configuration": "true",
          "__meta_kubernetes_service_labelpresent_app": "true",
          "__meta_kubernetes_endpointslice_address_type": "IPv4",
          "__meta_kubernetes_service_label_app": "my-app",
          "__meta_kubernetes_pod_label_app": "my-app",
          "__meta_kubernetes_pod_container_port_number": "8080",
          "__meta_kubernetes_endpointslice_name": "py-prometheus-app-bwbvn",
          "__meta_kubernetes_pod_label_pod_template_hash": "575cfdd46",
          "__meta_kubernetes_endpointslice_endpoint_node_name": "otel-target-allocator-talk-control-plane",
          "__meta_kubernetes_endpointslice_labelpresent_endpointslice_kubernetes_io_managed_by": "true",
          "__meta_kubernetes_endpointslice_label_app_kubernetes_io_name": "py-prometheus-app"
        }
      }
    ]
  }
}
```

O parâmetro de consulta `collector_id` no campo `_link` da saída acima indica
que esses são os targets pertencentes ao `otelcol-collector-0` (o nome do
`StatefulSet` criado para o recurso `OpenTelemetryCollector`).

> [!NOTE]
>
> Consulte o
> [readme do Target Allocator](https://github.com/open-telemetry/opentelemetry-operator/blob/main/cmd/otel-allocator/README.md?plain=1#L128-L134)
> para mais informações sobre o endpoint `/jobs`.

### O Target Allocator está habilitado? A descoberta de serviço Prometheus está habilitada?

Se os comandos `curl` acima não mostrarem uma lista de `ServiceMonitor`s e
`PodMonitor`s esperados, você precisa verificar se os recursos que populam esses
valores estão habilitados.

Uma coisa a lembrar é que só porque você inclui a seção `targetAllocator` no CR
`OpenTelemetryCollector` não significa que ele está habilitado. Você precisa
habilitá-lo explicitamente. Além disso, se você deseja usar a
[descoberta de serviço Prometheus](https://github.com/open-telemetry/opentelemetry-operator/blob/main/cmd/otel-allocator/README.md#discovery-of-prometheus-custom-resources),
você deve habilitá-la explicitamente:

- Defina `spec.targetAllocator.enabled` como `true`
- Defina `spec.targetAllocator.prometheusCR.enabled` como `true`

Para que o seu recurso `OpenTelemetryCollector` fique assim:

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otelcol
  namespace: opentelemetry
spec:
  mode: statefulset
  targetAllocator:
    enabled: true
    serviceAccount: opentelemetry-targetallocator-sa
    prometheusCR:
      enabled: true
```

Consulte a definição completa do recurso `OpenTelemetryCollector`
[em "Você sabe se as métricas estão sendo coletadas?"](#do-you-know-if-metrics-are-actually-being-scraped).

### Você configurou um seletor ServiceMonitor (ou PodMonitor)?

Se você configurou um seletor de
[`ServiceMonitor`](https://observability.thomasriley.co.uk/prometheus/configuring-prometheus/using-service-monitors/),
isso significa que o Target Allocator busca apenas `ServiceMonitor`s que possuem
um `metadata.label` correspondente ao valor em
[`serviceMonitorSelector`](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api/targetallocators.md#targetallocatorspecprometheuscrservicemonitorselector).

Suponha que você configurou um
[`serviceMonitorSelector`](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api/targetallocators.md#targetallocatorspecprometheuscrservicemonitorselector)
para o seu Target Allocator, como no exemplo a seguir:

```yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otelcol
  namespace: opentelemetry
spec:
  mode: statefulset
  targetAllocator:
    enabled: true
    serviceAccount: opentelemetry-targetallocator-sa
    prometheusCR:
      enabled: true
      serviceMonitorSelector:
        matchLabels:
          app: my-app
```

Ao definir o valor de
`spec.targetAllocator.prometheusCR.serviceMonitorSelector.matchLabels` como
`app: my-app`, isso significa que o seu recurso `ServiceMonitor` deve ter esse
mesmo valor em `metadata.labels`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sm-example
  labels:
    app: my-app
    release: prometheus
spec:
```

Consulte a definição completa do `ServiceMonitor`
[em "Você sabe se as métricas estão sendo coletadas?"](#do-you-know-if-metrics-are-actually-being-scraped).

Neste caso, o `prometheusCR.serviceMonitorSelector.matchLabels` do recurso
`OpenTelemetryCollector` está buscando apenas `ServiceMonitor`s com a label
`app: my-app`, como vemos no exemplo anterior.

Se o seu recurso `ServiceMonitor` não tiver essa label, o Target Allocator
falhará em descobrir targets de scraping desse `ServiceMonitor`.

> [!TIP]
>
> O mesmo se aplica se você estiver usando um [PodMonitor][]. Nesse caso, você
> usaria um [`podMonitorSelector`][] em vez de um `serviceMonitorSelector`.

### Você omitiu completamente a configuração serviceMonitorSelector e/ou podMonitorSelector?

Como mencionado em
["Você configurou um seletor ServiceMonitor ou PodMonitor"](#did-you-configure-a-servicemonitor-or-podmonitor-selector),
definir valores incompatíveis para `serviceMonitorSelector` e
`podMonitorSelector` resulta no Target Allocator falhando em descobrir targets
de scraping dos seus `ServiceMonitor`s e `PodMonitor`s, respectivamente.

Da mesma forma, em
[`v1beta1`](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api/opentelemetrycollectors.md#opentelemetryiov1beta1)
do CR `OpenTelemetryCollector`, omitir completamente essa configuração também
resulta no Target Allocator falhando em descobrir targets de scraping dos seus
`ServiceMonitor`s e `PodMonitor`s.

A partir do `v1beta1` do `OpenTelemetryOperator`, um `serviceMonitorSelector` e
`podMonitorSelector` devem ser incluídos, mesmo que você não pretenda usá-los,
como aqui:

```yaml
prometheusCR:
  enabled: true
  podMonitorSelector: {}
  serviceMonitorSelector: {}
```

Esta configuração significa que ele corresponderá a todos os recursos
`PodMonitor` e `ServiceMonitor`. Consulte a
[definição completa do OpenTelemetryCollector em "Você sabe se as métricas estão sendo coletadas?"](#do-you-know-if-metrics-are-actually-being-scraped).

### Suas labels, namespaces e portas correspondem para seu ServiceMonitor e seu Service (ou PodMonitor e seu Pod)?

O `ServiceMonitor` está configurado para capturar Kubernetes
[Services](https://kubernetes.io/docs/concepts/services-networking/service/) que
correspondem em:

- Labels
- Namespaces (opcional)
- Portas (endpoints)

Suponha que você tenha este `ServiceMonitor`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sm-example
  labels:
    app: my-app
    release: prometheus
spec:
  selector:
    matchLabels:
      app: my-app
  namespaceSelector:
    matchNames:
      - opentelemetry
  endpoints:
    - port: prom
      path: /metrics
    - port: py-client-port
      interval: 15s
    - port: py-server-port
```

O `ServiceMonitor` anterior está procurando quaisquer serviços que tenham:

- a label `app: my-app`
- residam em um namespace chamado `opentelemetry`
- uma porta chamada `prom`, `py-client-port`, _ou_ `py-server-port`

Por exemplo, o seguinte recurso `Service` seria capturado pelo `ServiceMonitor`,
porque corresponde aos critérios anteriores:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: py-prometheus-app
  namespace: opentelemetry
  labels:
    app: my-app
    app.kubernetes.io/name: py-prometheus-app
spec:
  selector:
    app: my-app
    app.kubernetes.io/name: py-prometheus-app
  ports:
    - name: prom
      port: 8080
```

O seguinte recurso `Service` não seria capturado, porque o `ServiceMonitor` está
procurando portas chamadas `prom`, `py-client-port`, _ou_ `py-server-port`, e a
porta deste serviço se chama `bleh`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: py-prometheus-app
  namespace: opentelemetry
  labels:
    app: my-app
    app.kubernetes.io/name: py-prometheus-app
spec:
  selector:
    app: my-app
    app.kubernetes.io/name: py-prometheus-app
  ports:
    - name: bleh
      port: 8080
```

> [!TIP]
>
> Se você estiver usando `PodMonitor`, o mesmo se aplica, exceto que ele captura
> pods do Kubernetes que correspondem em labels, namespaces e portas nomeadas.

[`podMonitorSelector`]:
  https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api/targetallocators.md#targetallocatorspecprometheuscr
[PodMonitor]:
  https://prometheus-operator.dev/docs/developer/getting-started/#using-podmonitors
