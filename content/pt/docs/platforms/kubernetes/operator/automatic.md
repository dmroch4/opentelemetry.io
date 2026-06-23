---
title: Injetando Auto-instrumentação
linkTitle: Auto-instrumentação
weight: 11
description:
  Uma implementação de auto-instrumentação usando o OpenTelemetry Operator.
# prettier-ignore
cSpell:ignore: Dockerfiles GRPCNETCLIENT k8sattributesprocessor otelinst otlpreceiver REDISCALA replicaset statefulset
default_lang_commit: e6937ea76745c40b06b9e04693b1ee396b4848e5
---

O OpenTelemetry Operator suporta injeção e configuração de bibliotecas de
auto-instrumentação para serviços .NET, Java, Node.js, Python e Go.

## Instalação

Primeiro, instale o
[OpenTelemetry Operator](https://github.com/open-telemetry/opentelemetry-operator)
no seu cluster.

Você pode fazer isso com o
[manifesto de release do Operator](https://github.com/open-telemetry/opentelemetry-operator#getting-started),
o
[helm chart do Operator](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/main/charts/opentelemetry-operator#opentelemetry-operator-helm-chart),
ou com o [Operator Hub](https://operatorhub.io/operator/opentelemetry-operator).

Na maioria dos casos, você precisará instalar o
[cert-manager](https://cert-manager.io/docs/installation/). Se você usar o helm
chart, existe uma opção para gerar um certificado auto-assinado.

> Se você deseja usar auto-instrumentação com Go, precisa habilitar o feature
> gate. Consulte
> [Controlando Capacidades de Instrumentação](#controlling-instrumentation-capabilities)
> para detalhes.

## Criar um OpenTelemetry Collector (Opcional)

É uma boa prática enviar telemetria de containers para um
[OpenTelemetry Collector](/docs/platforms/kubernetes/collector/) em vez de
diretamente para um backend. O Collector ajuda a simplificar o gerenciamento de
segredos, desacopla problemas de exportação de dados (como a necessidade de
fazer retentativas) das suas aplicações, e permite adicionar dados adicionais à
sua telemetria, como com o componente
[k8sattributesprocessor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/k8sattributesprocessor).
Se você optar por não usar um Collector, pode pular para a próxima seção.

O Operator fornece uma
[Custom Resource Definition (CRD) para o OpenTelemetry Collector](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api/opentelemetrycollectors.md)
que é usada para criar uma instância do Collector que o Operator gerencia. O
exemplo a seguir implanta o Collector como deployment (o padrão), mas existem
outros
[modos de implantação](https://github.com/open-telemetry/opentelemetry-operator#deployment-modes)
que podem ser usados.

Ao usar o modo `Deployment`, o operator também criará um Service que pode ser
usado para interagir com o Collector. O nome do service é o nome do recurso
`OpenTelemetryCollector` acrescido de `-collector`. Para o nosso exemplo, será
`demo-collector`.

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: demo
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
      debug:
        verbosity: basic

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter]
          exporters: [debug]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter]
          exporters: [debug]
        logs:
          receivers: [otlp]
          processors: [memory_limiter]
          exporters: [debug]
EOF
```

O comando acima resulta em uma implantação do Collector que você pode usar como
um endpoint para auto-instrumentação nos seus pods.

## Configurar Instrumentação Automática

Para poder gerenciar a instrumentação automática, o Operator precisa ser
configurado para saber quais pods instrumentar e qual instrumentação automática
usar para esses pods. Isso é feito por meio do
[Instrumentation CRD](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/api/instrumentations.md).

Criar o recurso Instrumentation corretamente é fundamental para que a
auto-instrumentação funcione. Garantir que todos os endpoints e variáveis de
ambiente estejam corretos é necessário para que a auto-instrumentação funcione
adequadamente.

### .NET

O seguinte comando criará um recurso Instrumentation básico que é configurado
especificamente para instrumentar serviços .NET.

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
EOF
```

Por padrão, o recurso Instrumentation que auto-instrumenta serviços .NET usa
`otlp` com o protocolo `http/protobuf`. Isso significa que o endpoint
configurado deve ser capaz de receber OTLP via `http/protobuf`. Portanto, o
exemplo usa `http://demo-collector:4318`, que se conectará à porta `http` do
`otlpreceiver` do Collector criado na etapa anterior.

#### Excluindo auto-instrumentação {#dotnet-excluding-auto-instrumentation}

Por padrão, a auto-instrumentação .NET inclui
[muitas bibliotecas de instrumentação](https://github.com/open-telemetry/opentelemetry-dotnet-instrumentation/blob/main/docs/config.md#instrumentations).
Isso facilita a instrumentação, mas pode resultar em dados em excesso ou
indesejados. Se houver bibliotecas que você não deseja usar, você pode definir
`OTEL_DOTNET_AUTO_[SIGNAL]_[NAME]_INSTRUMENTATION_ENABLED=false` onde `[SIGNAL]`
é o tipo de sinal e `[NAME]` é o nome sensível a maiúsculas e minúsculas da
biblioteca.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: '1'
  dotnet:
    env:
      - name: OTEL_DOTNET_AUTO_TRACES_GRPCNETCLIENT_INSTRUMENTATION_ENABLED
        value: false
      - name: OTEL_DOTNET_AUTO_METRICS_PROCESS_INSTRUMENTATION_ENABLED
        value: false
```

A auto-instrumentação .NET também suporta uma anotação de runtime para definir o
.NET
[Runtime Identifier (RID)](https://learn.microsoft.com/en-us/dotnet/core/rid-catalog).
Atualmente `linux-x64` (padrão) e `linux-musl-x64` são suportados:

```bash
instrumentation.opentelemetry.io/inject-dotnet: "true"
instrumentation.opentelemetry.io/otel-dotnet-auto-runtime: "linux-x64"   # default, can be omitted
instrumentation.opentelemetry.io/otel-dotnet-auto-runtime: "linux-musl-x64"  # for musl-based images
```

> **Nota:** Por padrão, o operator define
> `OTEL_DOTNET_AUTO_TRACES_ENABLED_INSTRUMENTATIONS` para todas as
> instrumentações disponíveis suportadas pelo release
> `opentelemetry-dotnet-instrumentation` consumido (ex.
> `AspNet,HttpClient,SqlClient`). Este valor pode ser substituído configurando a
> variável de ambiente explicitamente.

#### Saiba mais {#dotnet-learn-more}

Para mais detalhes, consulte
[Documentação de Auto-Instrumentação .NET](/docs/zero-code/dotnet/).

### Deno

O seguinte comando cria um recurso Instrumentation básico que é configurado para
instrumentar serviços [Deno](https://deno.com).

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  env:
    - name: OTEL_DENO
      value: 'true'
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: '1'
EOF
```

Os processos Deno exportam automaticamente dados de telemetria para o endpoint
configurado quando são iniciados com a variável de ambiente `OTEL_DENO=true`.
Portanto, o exemplo especifica essa variável de ambiente no campo `env` do
recurso Instrumentation, para que seja definida para todos os serviços que
recebem variáveis de ambiente injetadas com este recurso Instrumentation.

Por padrão, o recurso Instrumentation que auto-instrumenta serviços Deno usa
`otlp` com o protocolo `http/proto`. Isso significa que o endpoint configurado
deve ser capaz de receber OTLP via `http/proto`. Portanto, o exemplo usa
`http://demo-collector:4318`, que se conecta à porta `http/proto` do
`otlpreceiver` do Collector criado na etapa anterior.

> [!NOTE]
>
> [A integração OpenTelemetry do Deno][deno-docs] ainda não é estável. Como
> resultado, todas as cargas de trabalho que desejam ser instrumentadas com Deno
> devem ter a flag `--unstable-otel` definida ao iniciar o processo Deno.
>
> [deno-docs]: https://docs.deno.com/runtime/fundamentals/open_telemetry/

#### Opções de configuração {#deno-configuration-options}

Por padrão, a integração OpenTelemetry do Deno exporta a saída `console.log()`
como [logs](/docs/concepts/signals/logs/), enquanto ainda imprime os logs em
stdout / stderr. Você pode configurar esses comportamentos alternativos:

- `OTEL_DENO_CONSOLE=replace`: exportar apenas a saída `console.log()` como
  logs; não imprimir em stdout / stderr.
- `OTEL_DENO_CONSOLE=ignore`: não exportar a saída `console.log()` como logs;
  imprimir em stdout / stderr.

#### Saiba mais {#deno-learn-more}

Para mais detalhes, consulte a documentação da [integração
OpenTelemetry][deno-otel-docs] do Deno.

[deno-otel-docs]: https://docs.deno.com/runtime/fundamentals/open_telemetry/

### Go

O seguinte comando cria um recurso Instrumentation básico que é configurado
especificamente para instrumentar serviços Go.

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
EOF
```

Por padrão, o recurso Instrumentation que auto-instrumenta serviços Go usa
`otlp` com o protocolo `http/protobuf`. Isso significa que o endpoint
configurado deve ser capaz de receber OTLP via `http/protobuf`. Portanto, o
exemplo usa `http://demo-collector:4318`, que se conecta à porta `http/protobuf`
do `otlpreceiver` do Collector criado na etapa anterior.

A auto-instrumentação Go não suporta desabilitar nenhuma instrumentação.
[Consulte o repositório Go Auto-Instrumentation para mais detalhes.](https://github.com/open-telemetry/opentelemetry-go-instrumentation)

### Java

O seguinte comando cria um recurso Instrumentation básico que é configurado para
instrumentar serviços Java.

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
EOF
```

Por padrão, o recurso Instrumentation que auto-instrumenta serviços Java usa
`otlp` com o protocolo `http/protobuf`. Isso significa que o endpoint
configurado deve ser capaz de receber OTLP via `http` com payloads `protobuf`.
Portanto, o exemplo usa `http://demo-collector:4318`, que se conecta à porta
`http` do otlpreceiver do Collector criado na etapa anterior.

#### Excluindo auto-instrumentação {#java-excluding-auto-instrumentation}

Por padrão, a auto-instrumentação Java inclui
[muitas bibliotecas de instrumentação](/docs/zero-code/java/agent/getting-started/#supported-libraries-frameworks-application-services-and-jvms).
Isso facilita a instrumentação, mas pode resultar em dados em excesso ou
indesejados. Se houver bibliotecas que você não deseja usar, você pode definir
`OTEL_INSTRUMENTATION_[NAME]_ENABLED=false` onde `[NAME]` é o nome da
biblioteca. Se você sabe exatamente quais bibliotecas deseja usar, pode
desabilitar as bibliotecas padrão definindo
`OTEL_INSTRUMENTATION_COMMON_DEFAULT_ENABLED=false` e então usar
`OTEL_INSTRUMENTATION_[NAME]_ENABLED=true` onde `[NAME]` é o nome da biblioteca.
Para mais detalhes, consulte
[Suprimindo instrumentação específica](/docs/zero-code/java/agent/disable/).

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: '1'
  java:
    env:
      - name: OTEL_INSTRUMENTATION_KAFKA_ENABLED
        value: false
      - name: OTEL_INSTRUMENTATION_REDISCALA_ENABLED
        value: false
```

#### Saiba mais {#java-learn-more}

Para mais detalhes, consulte
[Configuração do agente Java](/docs/zero-code/java/agent/configuration/).

### Node.js

O seguinte comando cria um recurso Instrumentation básico que é configurado para
instrumentar serviços Node.js.

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4317
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
EOF
```

Por padrão, o recurso Instrumentation que auto-instrumenta serviços Node.js usa
`otlp` com o protocolo `grpc`. Isso significa que o endpoint configurado deve
ser capaz de receber OTLP via `grpc`. Portanto, o exemplo usa
`http://demo-collector:4317`, que se conecta à porta `grpc` do `otlpreceiver` do
Collector criado na etapa anterior.

#### Excluindo bibliotecas de instrumentação {#js-excluding-instrumentation-libraries}

Por padrão, a instrumentação zero-code do Node.js tem todas as bibliotecas de
instrumentação habilitadas.

Para habilitar apenas bibliotecas de instrumentação específicas, você pode usar
a variável de ambiente `OTEL_NODE_ENABLED_INSTRUMENTATIONS` conforme documentado
na
[documentação de instrumentação zero-code do Node.js](/docs/zero-code/js/configuration/#excluding-instrumentation-libraries).

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
# ... other fields skipped from this example
spec:
  # ... other fields skipped from this example
  nodejs:
    env:
      - name: OTEL_NODE_ENABLED_INSTRUMENTATIONS
        value: http,nestjs-core # comma-separated list of the instrumentation package names without the `@opentelemetry/instrumentation-` prefix.
```

Para manter todas as bibliotecas padrão e desabilitar apenas bibliotecas de
instrumentação específicas, você pode usar a variável de ambiente
`OTEL_NODE_DISABLED_INSTRUMENTATIONS`. Para detalhes, consulte
[Excluindo bibliotecas de instrumentação](/docs/zero-code/js/configuration/#excluding-instrumentation-libraries).

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
# ... other fields skipped from this example
spec:
  # ... other fields skipped from this example
  nodejs:
    env:
      - name: OTEL_NODE_DISABLED_INSTRUMENTATIONS
        value: fs,grpc # comma-separated list of the instrumentation package names without the `@opentelemetry/instrumentation-` prefix.
```

> [!NOTE]
>
> Se ambas as variáveis de ambiente estiverem definidas,
> `OTEL_NODE_ENABLED_INSTRUMENTATIONS` é aplicada primeiro e, em seguida,
> `OTEL_NODE_DISABLED_INSTRUMENTATIONS` é aplicada a essa lista. Portanto, se a
> mesma instrumentação estiver incluída em ambas as listas, essa instrumentação
> será desabilitada.

#### Saiba mais {#js-learn-more}

Para mais detalhes, consulte
[Auto-instrumentação Node.js](/docs/languages/js/libraries/#registration).

### Python

O seguinte comando criará um recurso Instrumentation básico que é configurado
especificamente para instrumentar serviços Python.

```bash
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: "1"
EOF
```

Por padrão, o recurso `Instrumentation` que auto-instrumenta serviços Python usa
`otlp` com o protocolo `http/protobuf` (gRPC não é suportado no momento). Isso
significa que o endpoint configurado deve ser capaz de receber OTLP via
`http/protobuf`. Portanto, o exemplo usa `http://demo-collector:4318`, que se
conectará à porta `http` do `otlpreceiver` do Collector criado na etapa
anterior.

> A partir do operator v0.108.0, o recurso Instrumentation define
> automaticamente `OTEL_EXPORTER_OTLP_PROTOCOL` como `http/protobuf` para
> serviços Python. Se você usar uma versão mais antiga do Operator, você
> **DEVE** definir esta variável de ambiente como `http/protobuf`, caso
> contrário a auto-instrumentação Python não funcionará.

#### Auto-instrumentando logs Python

Por padrão, a auto-instrumentação de logs Python está desabilitada. Se você
deseja habilitar este recurso, deve definir a variável de ambiente
`OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED` da seguinte forma:

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: python-instrumentation
  namespace: application
spec:
  exporter:
    endpoint: http://demo-collector:4318
  env:
  propagators:
    - tracecontext
    - baggage
  python:
    env:
      - name: OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED
        value: 'true'
```

> A partir do operator v0.111.0, não é mais necessário definir
> `OTEL_LOGS_EXPORTER` como `otlp`.

#### Excluindo auto-instrumentação {#python-excluding-auto-instrumentation}

Por padrão, a auto-instrumentação Python inclui
[muitas bibliotecas de instrumentação](https://github.com/open-telemetry/opentelemetry-operator/blob/main/autoinstrumentation/python/requirements.txt).
Isso facilita a instrumentação, mas pode resultar em dados em excesso ou
indesejados. Se houver pacotes que você não deseja instrumentar, você pode
definir a variável de ambiente `OTEL_PYTHON_DISABLED_INSTRUMENTATIONS`.

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: demo-instrumentation
spec:
  exporter:
    endpoint: http://demo-collector:4318
  propagators:
    - tracecontext
    - baggage
  sampler:
    type: parentbased_traceidratio
    argument: '1'
  python:
    env:
      - name: OTEL_PYTHON_DISABLED_INSTRUMENTATIONS
        value:
          <comma-separated list of package names to exclude from
          instrumentation>
```

Consulte a
[documentação de configuração do agente Python](/docs/zero-code/python/configuration/#disabling-specific-instrumentations)
para mais detalhes.

#### Saiba mais {#python-learn-more}

Para peculiaridades específicas do Python, consulte a
[documentação do OpenTelemetry Operator para Python](/docs/zero-code/python/operator/#python-specific-topics)
e a
[documentação de configuração do agente Python](/docs/zero-code/python/configuration/).

---

Agora que seu objeto Instrumentation está criado, seu cluster tem a capacidade
de auto-instrumentar serviços e enviar dados para um endpoint. No entanto, a
auto-instrumentação com o OpenTelemetry Operator segue um modelo de opt-in. Para
ativar a instrumentação automática, você precisará adicionar uma anotação ao seu
deployment.

## Adicionar anotações a deployments existentes

O passo final é ativar seus serviços para instrumentação automática. Isso é
feito atualizando `spec.template.metadata.annotations` do seu serviço para
incluir uma anotação específica de linguagem:

- .NET: `instrumentation.opentelemetry.io/inject-dotnet: "true"`
- Deno: `instrumentation.opentelemetry.io/inject-sdk: "true"`
- Go: `instrumentation.opentelemetry.io/inject-go: "true"`
- Java: `instrumentation.opentelemetry.io/inject-java: "true"`
- Node.js: `instrumentation.opentelemetry.io/inject-nodejs: "true"`
- Python: `instrumentation.opentelemetry.io/inject-python: "true"`

Os valores possíveis para a anotação podem ser

- `"true"` - para injetar o recurso `Instrumentation` com o nome padrão do
  namespace atual.
- `"my-instrumentation"` - para injetar a instância do CR `Instrumentation` com
  o nome `"my-instrumentation"` no namespace atual.
- `"my-other-namespace/my-instrumentation"` - para injetar a instância do CR
  `Instrumentation` com o nome `"my-instrumentation"` de outro namespace
  `"my-other-namespace"`.
- `"false"` - não injetar

Alternativamente, a anotação pode ser adicionada a um namespace, o que resultará
em todos os serviços nesse namespace optando pela instrumentação automática.
Consulte a
[documentação de auto-instrumentação do Operator](https://github.com/open-telemetry/opentelemetry-operator/blob/main/README.md#opentelemetry-auto-instrumentation-injection)
para mais detalhes.

### Ativar um serviço Go

Ao contrário da auto-instrumentação de outras linguagens, Go funciona via um
agente eBPF executado como sidecar. Quando ativado, o Operator injetará este
sidecar no seu pod. Além da anotação
`instrumentation.opentelemetry.io/inject-go` mencionada acima, você também deve
fornecer um valor para a
[variável de ambiente `OTEL_GO_AUTO_TARGET_EXE`](https://github.com/open-telemetry/opentelemetry-go-instrumentation/blob/main/docs/how-it-works.md).

Você pode definir esta variável de ambiente via a anotação
`instrumentation.opentelemetry.io/otel-go-auto-target-exe`.

```yaml
instrumentation.opentelemetry.io/inject-go: 'true'
instrumentation.opentelemetry.io/otel-go-auto-target-exe: '/path/to/container/executable'
```

Esta variável de ambiente também pode ser definida via o recurso
Instrumentation, com a anotação tendo precedência. Como a auto-instrumentação Go
requer que `OTEL_GO_AUTO_TARGET_EXE` seja definida, você deve fornecer um
caminho de executável válido via a anotação ou o recurso Instrumentation. Falhar
em definir este valor faz com que a injeção de instrumentação seja abortada,
deixando o pod original inalterado.

Como a auto-instrumentação Go usa eBPF, ela também requer permissões elevadas.
Quando você ativar, o sidecar que o Operator injeta exigirá as seguintes
permissões:

```yaml
securityContext:
  privileged: true
  runAsUser: 0
```

### Auto-instrumentando um container Python baseado em musl {#annotations-python-musl}

Desde o operator v0.113.0, a auto-instrumentação Python também reconhece uma
anotação que permite executá-la em imagens com uma biblioteca C diferente da
glibc.

```sh
# for Linux glibc based images, this is the default value and can be omitted
instrumentation.opentelemetry.io/otel-python-platform: "glibc"
# for Linux musl based images
instrumentation.opentelemetry.io/otel-python-platform: "musl"
```

## Pods com múltiplos containers

### Instrumentação única

Se nada mais for especificado, a instrumentação é realizada no primeiro
container disponível na especificação do pod (de `.spec.containers`, não de init
containers). Em alguns casos — por exemplo quando um sidecar Istio é injetado —
torna-se necessário especificar em qual(is) container(s) a injeção deve ser
realizada.

Use a anotação `instrumentation.opentelemetry.io/container-names` para indicar
um ou mais nomes de containers (de `.spec.containers.name` ou
`.spec.initContainers.name`) nos quais a injeção deve ser feita:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment-with-multiple-containers
spec:
  selector:
    matchLabels:
      app: my-pod-with-multiple-containers
  replicas: 1
  template:
    metadata:
      labels:
        app: my-pod-with-multiple-containers
      annotations:
        instrumentation.opentelemetry.io/inject-java: 'true'
        instrumentation.opentelemetry.io/container-names: 'myapp,myapp2'
    spec:
      containers:
        - name: myapp
          image: myImage1
        - name: myapp2
          image: myImage2
        - name: myapp3
          image: myImage3
```

No caso acima, os containers `myapp` e `myapp2` serão instrumentados; `myapp3`
não será.

> **NOTA**: A auto-instrumentação Go **não** suporta pods com múltiplos
> containers. Ao injetar auto-instrumentação Go, o primeiro container deve ser o
> único container que você deseja instrumentar.

### Instrumentando init containers

Os init containers podem ser instrumentados incluindo seus nomes na anotação
`container-names`. Quando um init container é alvo de instrumentação, o operator
insere automaticamente o init container de instrumentação **antes** do init
container alvo na sequência de init containers do pod. Isso garante que os
arquivos do agente de instrumentação estejam disponíveis quando o init container
alvo for executado.

Instrumentações suportadas para init containers: Java, Python, Node.js, .NET e
injeção somente de SDK.

Não suportados para init containers: Go (não suporta pods com múltiplos
containers), Apache HTTPD e NGINX.

> **Nota**: O Kubernetes garante que os nomes de containers sejam únicos tanto
> nas listas `initContainers` quanto `containers` dentro de uma especificação de
> pod, permitindo que o operator identifique inequivocamente se um nome de
> container refere-se a um init container ou a um container regular.

Exemplo com instrumentação de init container e container regular:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment-with-init-container
spec:
  selector:
    matchLabels:
      app: my-app
  replicas: 1
  template:
    metadata:
      labels:
        app: my-app
      annotations:
        instrumentation.opentelemetry.io/inject-python: 'true'
        instrumentation.opentelemetry.io/container-names: 'my-init-job,myapp'
    spec:
      initContainers:
        - name: my-init-job
          image: my-python-init-image
      containers:
        - name: myapp
          image: my-python-app-image
```

Neste exemplo, tanto `my-init-job` (um init container) quanto `myapp` (um
container regular) serão instrumentados com auto-instrumentação Python.

### Múltiplas instrumentações

A multi-instrumentação funciona apenas quando o feature flag
`enable-multi-instrumentation` está definido como `true`. Quando habilitado, use
anotações de nomes de containers específicas por linguagem para especificar
quais containers devem receber qual instrumentação.

Se os nomes de containers específicos por linguagem não forem especificados, a
instrumentação é realizada no primeiro container regular disponível na
especificação do pod (apenas se a injeção de instrumentação única estiver
configurada).

Em alguns casos, containers no mesmo pod usam tecnologias diferentes. Use as
anotações de nomes de containers específicas por linguagem para indicar um ou
mais nomes de containers (de `.spec.containers.name` ou
`.spec.initContainers.name`) nos quais a injeção deve ser feita:

| Linguagem    | Anotação                                                        |
| ------------ | --------------------------------------------------------------- |
| Java         | `instrumentation.opentelemetry.io/java-container-names`         |
| Node.js      | `instrumentation.opentelemetry.io/nodejs-container-names`       |
| Python       | `instrumentation.opentelemetry.io/python-container-names`       |
| .NET         | `instrumentation.opentelemetry.io/dotnet-container-names`       |
| Go           | `instrumentation.opentelemetry.io/go-container-names`           |
| Apache HTTPD | `instrumentation.opentelemetry.io/apache-httpd-container-names` |
| NGINX        | `instrumentation.opentelemetry.io/nginx-container-names`        |
| Somente SDK  | `instrumentation.opentelemetry.io/sdk-container-names`          |

Exemplo com Java e Python executando em containers diferentes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment-with-multi-containers-multi-instrumentations
spec:
  selector:
    matchLabels:
      app: my-pod-with-multi-containers-multi-instrumentations
  replicas: 1
  template:
    metadata:
      labels:
        app: my-pod-with-multi-containers-multi-instrumentations
      annotations:
        instrumentation.opentelemetry.io/inject-java: 'true'
        instrumentation.opentelemetry.io/java-container-names: 'myapp,myapp2'
        instrumentation.opentelemetry.io/inject-python: 'true'
        instrumentation.opentelemetry.io/python-container-names: 'myapp3'
    spec:
      containers:
        - name: myapp
          image: myImage1
        - name: myapp2
          image: myImage2
        - name: myapp3
          image: myImage3
```

No caso acima, `myapp` e `myapp2` serão instrumentados com Java e `myapp3` com
instrumentação Python.

> **NOTA**: A auto-instrumentação Go **não** suporta pods com múltiplos
> containers. **NOTA**: Um único container não pode ser instrumentado com
> múltiplas instrumentações de linguagem. **NOTA**: A anotação
> `instrumentation.opentelemetry.io/container-names` não é usada para este
> recurso.

## Usando instrumentação personalizada ou de fornecedor

Por padrão, o operator usa bibliotecas de auto-instrumentação upstream. Imagens
de auto-instrumentação personalizadas podem ser configuradas substituindo os
campos `image` no CR `Instrumentation`:

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-instrumentation
spec:
  java:
    image: your-customized-auto-instrumentation-image:java
  nodejs:
    image: your-customized-auto-instrumentation-image:nodejs
  python:
    image: your-customized-auto-instrumentation-image:python
  dotnet:
    image: your-customized-auto-instrumentation-image:dotnet
  go:
    image: your-customized-auto-instrumentation-image:go
  apacheHttpd:
    image: your-customized-auto-instrumentation-image:apache-httpd
  nginx:
    image: your-customized-auto-instrumentation-image:nginx
```

Os Dockerfiles para auto-instrumentação podem ser encontrados no
[diretório autoinstrumentation](https://github.com/open-telemetry/opentelemetry-operator/tree/main/autoinstrumentation).
Siga as instruções nos Dockerfiles sobre como construir uma imagem de container
personalizada.

## Usando auto-instrumentação Apache HTTPD

Para auto-instrumentação Apache HTTPD, o operator assume a versão 2.4 do HTTPD e
o diretório de configuração `/usr/local/apache2/conf` por padrão (como usado na
imagem oficial `httpd`). Se você precisar da versão 2.2, de um diretório de
configuração diferente ou de atributos de agente personalizados, use o seguinte
exemplo:

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-instrumentation
spec:
  apacheHttpd:
    image: your-customized-auto-instrumentation-image:apache-httpd
    version: '2.2'
    configPath: /your-custom-config-path
    attrs:
      - name: ApacheModuleOtelMaxQueueSize
        value: '4096'
```

Uma lista completa de atributos disponíveis pode ser encontrada em
[otel-webserver-module](https://github.com/open-telemetry/opentelemetry-cpp-contrib/tree/main/instrumentation/otel-webserver-module).

## Usando auto-instrumentação NGINX

Para auto-instrumentação NGINX, as versões 1.22.0, 1.23.0 e 1.23.1 do NGINX são
suportadas. O arquivo de configuração do NGINX deve estar em
`/etc/nginx/nginx.conf` por padrão. A instrumentação também espera um diretório
`conf.d` no mesmo diretório que o arquivo de configuração, com uma diretiva
`include <config-file-dir-path>/conf.d/*.conf;` na seção `http { ... }`. Você
também pode ajustar os atributos do OpenTelemetry SDK:

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-instrumentation
spec:
  nginx:
    image: your-customized-auto-instrumentation-image:nginx
    configFile: /my/custom-dir/custom-nginx.conf
    attrs:
      - name: NginxModuleOtelMaxQueueSize
        value: '4096'
```

Uma lista completa de atributos disponíveis pode ser encontrada em
[otel-webserver-module](https://github.com/open-telemetry/opentelemetry-cpp-contrib/tree/main/instrumentation/otel-webserver-module).

## Injetar apenas variáveis de ambiente do OpenTelemetry SDK

Você pode configurar o OpenTelemetry SDK para aplicações que atualmente não
podem ser auto-instrumentadas usando `inject-sdk` no lugar de `inject-python` ou
`inject-java`. Isso injetará variáveis de ambiente como
`OTEL_RESOURCE_ATTRIBUTES`, `OTEL_TRACES_SAMPLER` e
`OTEL_EXPORTER_OTLP_ENDPOINT` que você configura no recurso `Instrumentation`,
mas não injetará o próprio SDK.

```bash
instrumentation.opentelemetry.io/inject-sdk: "true"
```

## Controlando Capacidades de Instrumentação

O operator permite especificar, via feature flags, quais linguagens o recurso
`Instrumentation` pode instrumentar. As linguagens habilitadas por padrão só
precisam ter seu gate fornecido ao desabilitar. O suporte a linguagens pode ser
desabilitado passando a flag com um valor `false`.

| Linguagem    | Gate                                  | Valor Padrão |
| ------------ | ------------------------------------- | ------------ |
| Java         | `enable-java-instrumentation`         | `true`       |
| Node.js      | `enable-nodejs-instrumentation`       | `true`       |
| Python       | `enable-python-instrumentation`       | `true`       |
| .NET         | `enable-dotnet-instrumentation`       | `true`       |
| Apache HTTPD | `enable-apache-httpd-instrumentation` | `true`       |
| Go           | `enable-go-instrumentation`           | `false`      |
| NGINX        | `enable-nginx-instrumentation`        | `false`      |

A multi-instrumentação (múltiplas linguagens no mesmo pod) pode ser habilitada
com a flag `enable-multi-instrumentation`, que assume como padrão `false`. Para
mais informações sobre as capacidades do recurso de multi-instrumentação,
consulte
[Pods com múltiplos containers e múltiplas instrumentações](#multiple-instrumentations).

## Configurar atributos de recurso

O OpenTelemetry Operator pode definir automaticamente atributos de recurso
conforme definido nas
[Convenções Semânticas do OpenTelemetry](/docs/specs/semconv/non-normative/k8s-attributes/).

### Configurar atributos de recurso com anotações

Use o prefixo de anotação `resource.opentelemetry.io/` para adicionar atributos
de recurso aos dados produzidos pela instrumentação do OpenTelemetry:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  annotations:
    resource.opentelemetry.io/service.name: 'my-service'
    resource.opentelemetry.io/service.version: '1.0.0'
    resource.opentelemetry.io/deployment.environment.name: 'production'
spec:
  containers:
    - name: main-container
      image: your-image:tag
```

### Configurar atributos de recurso com labels

Você também pode usar labels comuns do Kubernetes para definir atributos de
recurso (o primeiro encontrado vence). As seguintes labels são suportadas:

- `app.kubernetes.io/instance` → `service.name`
- `app.kubernetes.io/name` → `service.name`
- `app.kubernetes.io/version` → `service.version`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  labels:
    app.kubernetes.io/name: 'my-service'
    app.kubernetes.io/version: '1.0.0'
    app.kubernetes.io/part-of: 'shop'
spec:
  containers:
    - name: main-container
      image: your-image:tag
```

Isso requer opt-in explícito via CR `Instrumentation`:

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-instrumentation
spec:
  defaults:
    useLabelsForResourceAttributes: true
```

### Prioridade para definição de atributos de recurso

A prioridade para definição de atributos de recurso é a seguinte (o primeiro
encontrado vence):

1. Variáveis de ambiente `OTEL_RESOURCE_ATTRIBUTES` e `OTEL_SERVICE_NAME`
2. Anotações com o prefixo `resource.opentelemetry.io/`
3. Labels (ex. `app.kubernetes.io/name`) quando
   `defaults.useLabelsForResourceAttributes=true`
4. Atributos de recurso calculados a partir dos metadados do pod (ex.
   `k8s.pod.name`)
5. Atributos de recurso definidos no CR `Instrumentation` em
   `spec.resource.resourceAttributes`

Esta prioridade é aplicada por atributo individualmente, portanto é possível
definir alguns atributos via anotações e outros via labels.

### Como os atributos de recurso são calculados a partir dos metadados do pod

#### Como `service.name` é calculado

O primeiro valor encontrado nesta ordem é usado:

1. `pod.annotation[resource.opentelemetry.io/service.name]`
2. `pod.label[app.kubernetes.io/name]` (se
   `useLabelsForResourceAttributes=true`)
3. `k8s.deployment.name`
4. `k8s.replicaset.name`
5. `k8s.statefulset.name`
6. `k8s.daemonset.name`
7. `k8s.cronjob.name`
8. `k8s.job.name`
9. `k8s.pod.name`
10. `k8s.container.name`

#### Como `service.version` é calculado

O primeiro valor encontrado nesta ordem é usado:

1. `pod.annotation[resource.opentelemetry.io/service.version]`
2. `pod.label[app.kubernetes.io/version]` (se
   `useLabelsForResourceAttributes=true`)
3. Tag da imagem Docker do container (apenas se a tag não contiver `/`)

#### Como `service.instance.id` é calculado

O primeiro valor encontrado nesta ordem é usado:

1. `pod.annotation[resource.opentelemetry.io/service.instance.id]`
2. Concatenação de `k8s.namespace.name`, `k8s.pod.name` e `k8s.container.name`
   unidos por `.`

#### Como `service.namespace` é calculado

O primeiro valor encontrado nesta ordem é usado:

1. `pod.annotation[resource.opentelemetry.io/service.namespace]`
2. `k8s.namespace.name`

## Solução de problemas

Se você encontrar problemas ao tentar auto-instrumentar seu código, aqui estão
algumas coisas que você pode tentar.

### O recurso Instrumentation foi instalado?

Após instalar o recurso `Instrumentation`, verifique se ele foi instalado
corretamente executando este comando, onde `<namespace>` é o namespace no qual o
recurso `Instrumentation` está implantado:

```sh
kubectl describe otelinst -n <namespace>
```

Saída de exemplo:

```yaml
Name:         python-instrumentation
Namespace:    application
Labels:       app.kubernetes.io/managed-by=opentelemetry-operator
Annotations:  instrumentation.opentelemetry.io/default-auto-instrumentation-apache-httpd-image:
               ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-apache-httpd:1.0.3
             instrumentation.opentelemetry.io/default-auto-instrumentation-dotnet-image:
               ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-dotnet:0.7.0
             instrumentation.opentelemetry.io/default-auto-instrumentation-go-image:
               ghcr.io/open-telemetry/opentelemetry-go-instrumentation/autoinstrumentation-go:v0.2.1-alpha
             instrumentation.opentelemetry.io/default-auto-instrumentation-java-image:
               ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:1.26.0
             instrumentation.opentelemetry.io/default-auto-instrumentation-nodejs-image:
               ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:0.40.0
             instrumentation.opentelemetry.io/default-auto-instrumentation-python-image:
               ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.39b0
API Version:  opentelemetry.io/v1alpha1
Kind:         Instrumentation
Metadata:
 Creation Timestamp:  2023-07-28T03:42:12Z
 Generation:          1
 Resource Version:    3385
 UID:                 646661d5-a8fc-4b64-80b7-8587c9865f53
Spec:
...
 Exporter:
   Endpoint:  http://demo-collector.opentelemetry.svc.cluster.local:4318
...
 Propagators:
   tracecontext
   baggage
 Python:
   Image:  ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:0.39b0
   Resource Requirements:
     Limits:
       Cpu:     500m
       Memory:  32Mi
     Requests:
       Cpu:     50m
       Memory:  32Mi
 Resource:
 Sampler:
Events:  <none>
```

### Os logs do OTel Operator mostram erros de auto-instrumentação?

Verifique os logs do OTel Operator para quaisquer erros relacionados à
auto-instrumentação executando este comando:

```sh
kubectl logs -l app.kubernetes.io/name=opentelemetry-operator --container manager -n opentelemetry-operator-system --follow
```

### Os recursos foram implantados na ordem correta?

A ordem importa! O recurso `Instrumentation` precisa ser implantado antes de
implantar a aplicação, caso contrário a auto-instrumentação não funcionará.

Lembre-se da anotação de auto-instrumentação:

```yaml
annotations:
  instrumentation.opentelemetry.io/inject-python: 'true'
```

Quando o pod inicia, a anotação acima diz ao OTel Operator para procurar um
objeto `Instrumentation` no namespace do pod. Ela também diz ao Operator para
injetar auto-instrumentação Python no pod.

Ele adiciona um
[init-container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)
ao pod da aplicação, chamado `opentelemetry-auto-instrumentation`, que é então
usado para injetar a auto-instrumentação no container da aplicação.

Se o recurso `Instrumentation` não estiver presente no momento em que a
aplicação for implantada, no entanto, o init-container não poderá ser criado.
Portanto, se a aplicação for implantada _antes_ de implantar o recurso
`Instrumentation`, a auto-instrumentação falhará.

Para garantir que o init-container `opentelemetry-auto-instrumentation` tenha
iniciado corretamente (ou tenha iniciado de forma alguma), execute o seguinte
comando:

```sh
kubectl get events -n <your_app_namespace>
```

Que deve produzir uma saída semelhante a esta:

```text
53s         Normal   Created             pod/py-otel-server-7f54bf4cbc-p8wmj    Created container opentelemetry-auto-instrumentation
53s         Normal   Started             pod/py-otel-server-7f54bf4cbc-p8wmj    Started container opentelemetry-auto-instrumentation
```

Se a saída estiver faltando entradas `Created` e/ou `Started` para
`opentelemetry-auto-instrumentation`, isso significa que há um problema com sua
auto-instrumentação. Isso pode ser resultado de qualquer um dos seguintes:

- O recurso `Instrumentation` não foi instalado (ou não foi instalado
  corretamente).
- O recurso `Instrumentation` foi instalado _depois_ que a aplicação foi
  implantada.
- Há um erro na anotação de auto-instrumentação, ou a anotação está no lugar
  errado — veja o item #4 abaixo.

Certifique-se de verificar a saída de `kubectl get events` para quaisquer erros,
pois esses podem ajudar a apontar para o problema.

### A anotação de auto-instrumentação está correta?

Às vezes a auto-instrumentação pode falhar devido a erros na anotação de
auto-instrumentação.

Aqui estão algumas coisas a verificar:

- **A auto-instrumentação é para a linguagem correta?**
  - Por exemplo, ao instrumentar uma aplicação Python, certifique-se de que a
    anotação não diz incorretamente
    `instrumentation.opentelemetry.io/inject-java: "true"` no lugar.
  - Para **Deno**, certifique-se de que está usando a anotação
    `instrumentation.opentelemetry.io/inject-sdk: "true"`, em vez de uma
    anotação contendo a string `deno`.
- **A anotação de auto-instrumentação está no local correto?** Ao definir um
  `Deployment`, as anotações podem ser adicionadas em um de dois locais:
  `spec.metadata.annotations` e `spec.template.metadata.annotations`. A anotação
  de auto-instrumentação precisa ser adicionada em
  `spec.template.metadata.annotations`, caso contrário não funcionará.

### O endpoint de auto-instrumentação foi configurado corretamente?

O atributo `spec.exporter.endpoint` do recurso `Instrumentation` define para
onde enviar os dados. Isso pode ser um [OTel Collector](/docs/collector/), ou
qualquer endpoint OTLP. Se este atributo for omitido, o padrão será
`http://localhost:4317`, que provavelmente não enviará dados de telemetria para
lugar nenhum.

Ao enviar telemetria para um OTel Collector localizado no mesmo cluster
Kubernetes, `spec.exporter.endpoint` deve referenciar o nome do
[`Service`](https://kubernetes.io/docs/concepts/services-networking/service/) do
OTel Collector.

Por exemplo:

```yaml
spec:
  exporter:
    endpoint: http://demo-collector.opentelemetry.svc.cluster.local:4317
```

Aqui, o endpoint do Collector está definido como
`http://demo-collector.opentelemetry.svc.cluster.local:4317`, onde
`demo-collector` é o nome do `Service` Kubernetes do OTel Collector. No exemplo
acima, o Collector está em execução em um namespace diferente da aplicação, o
que significa que `opentelemetry.svc.cluster.local` deve ser adicionado ao nome
do serviço do Collector, onde `opentelemetry` é o namespace no qual o Collector
reside.
