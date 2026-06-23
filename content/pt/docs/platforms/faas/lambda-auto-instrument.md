---
title: Auto-Instrumentação Lambda
weight: 11
description: Instrumente automaticamente suas Lambdas com OpenTelemetry
cSpell:ignore: Corretto regionalized
default_lang_commit: bf8cc730d1e6f7015372fb96c9cc627d7421d9b7
---

A comunidade OpenTelemetry fornece Lambda layers de instrumentação standalone
para as seguintes linguagens:

- Java
- JavaScript
- Python
- Ruby

Estas podem ser adicionadas à sua Lambda pelo portal AWS para instrumentar
automaticamente sua aplicação. Essas layers não incluem o Collector, que é uma
adição obrigatória a menos que você configure uma instância externa do Collector
para enviar seus dados.

## Adicionar o ARN da Lambda layer do OTel Collector

Consulte o [guia da Lambda layer do Collector](../lambda-collector/) para
adicionar a layer à sua aplicação e configurar o Collector. Recomendamos
adicionar isso primeiro.

## Requisitos de Linguagem

{{< tabpane text=true >}} {{% tab Java %}}

A Lambda layer suporta os runtimes Lambda Java 8, 11 e 17 (Corretto). Para mais
informações sobre versões Java suportadas, consulte a
[documentação do OpenTelemetry Java](/docs/languages/java/).

**Nota:** O agente de auto-instrumentação Java está na Lambda layer — a
instrumentação automática tem um impacto notável no tempo de inicialização no
AWS Lambda e geralmente você precisará usar isso junto com concorrência
provisionada e requisições de aquecimento para atender requisições de produção
sem causar timeouts nas requisições iniciais durante a inicialização.

Por padrão, o agente Java OTel na Layer tentará auto-instrumentar todo o código
em sua aplicação. Isso pode ter um impacto negativo no tempo de cold start da
Lambda.

Recomendamos habilitar a auto-instrumentação apenas para as
bibliotecas/frameworks usados pela sua aplicação.

Para habilitar apenas instrumentações específicas, você pode usar as seguintes
variáveis de ambiente:

- `OTEL_INSTRUMENTATION_COMMON_DEFAULT_ENABLED`: quando definida como false,
  desabilita a auto-instrumentação na Layer, exigindo que cada instrumentação
  seja habilitada individualmente.
- `OTEL_INSTRUMENTATION_<NAME>_ENABLED`: defina como true para habilitar a
  auto-instrumentação para uma biblioteca ou framework específico. Substitua
  `<NAME>` pela instrumentação que você deseja habilitar. Para a lista de
  instrumentações disponíveis, consulte [Suprimindo instrumentação específica do
  agente][1].

  [1]:
    /docs/zero-code/java/agent/disable/#suppressing-specific-agent-instrumentation

Por exemplo, para habilitar apenas a auto-instrumentação para Lambda e o AWS
SDK, você definiria as seguintes variáveis de ambiente:

```sh
OTEL_INSTRUMENTATION_COMMON_DEFAULT_ENABLED=false
OTEL_INSTRUMENTATION_AWS_LAMBDA_ENABLED=true
OTEL_INSTRUMENTATION_AWS_SDK_ENABLED=true
```

{{% /tab %}} {{% tab JavaScript %}}

A Lambda layer suporta os runtimes Lambda Node.js v18+. Para mais informações
sobre versões JavaScript e Node.js suportadas, consulte a
[documentação do OpenTelemetry JavaScript](https://github.com/open-telemetry/opentelemetry-js).

{{% /tab %}} {{% tab Python %}}

A Lambda layer suporta os runtimes Lambda Python 3.9+. Para mais informações
sobre versões Python suportadas, consulte a
[documentação do OpenTelemetry Python](https://github.com/open-telemetry/opentelemetry-python/blob/main/README.md#supported-runtimes)
e o pacote no [PyPi](https://pypi.org/project/opentelemetry-api/).

{{% /tab %}} {{% tab Ruby %}}

A Lambda layer suporta os runtimes Lambda Ruby 3.2 e 3.3. Para mais informações
sobre versões do SDK e API do OpenTelemetry Ruby suportadas, consulte a
[documentação do OpenTelemetry Ruby](https://github.com/open-telemetry/opentelemetry-ruby/blob/main/README.md#compatibility)
e o pacote no [RubyGem](https://rubygems.org/search?query=opentelemetry).

{{% /tab %}} {{< /tabpane >}}

## Configurar `AWS_LAMBDA_EXEC_WRAPPER`

Altere o ponto de entrada da sua aplicação definindo
`AWS_LAMBDA_EXEC_WRAPPER=/opt/otel-handler` para Node.js, Java, Ruby ou Python.
Este script wrapper invoca sua aplicação Lambda com a instrumentação automática
aplicada.

## Adicionar o ARN da Lambda Layer de Instrumentação

Para habilitar a auto-instrumentação OTel em sua função Lambda, você precisa
adicionar e configurar as layers de instrumentação e do Collector, e então
habilitar o rastreamento.

1. Abra a função Lambda que você pretende instrumentar no console AWS.
2. Na seção Layers no Designer, escolha Add a layer.
3. Em specify an ARN, cole o ARN da layer e então escolha Add.

Encontre o
[release mais recente da layer de instrumentação](https://github.com/open-telemetry/opentelemetry-lambda/releases)
para sua linguagem e use seu ARN após alterar a tag `<region>` para a região em
que sua Lambda está.

Nota: Lambda layers são um recurso regionalizado, o que significa que só podem
ser usadas na Região em que são publicadas. Certifique-se de usar a layer na
mesma região que suas funções Lambda. A comunidade publica layers em todas as
regiões disponíveis.

## Configurar seus exportadores de SDK

Os exportadores padrão usados pelas Lambda layers funcionarão sem nenhuma
alteração se houver um Collector embutido com receptores gRPC/HTTP. As variáveis
de ambiente não precisam ser atualizadas. No entanto, existem diferentes níveis
de suporte a protocolos e valores padrão por linguagem, documentados abaixo.

{{< tabpane text=true >}} {{% tab Java %}}

`OTEL_EXPORTER_OTLP_PROTOCOL=grpc` suporta: `grpc`, `http/protobuf` e
`http/json` `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317`

{{% /tab %}} {{% tab JavaScript %}}

A variável de ambiente `OTEL_EXPORTER_OTLP_PROTOCOL` não é suportada. O
exportador hardcoded usa o protocolo `http/protobuf`
`OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318`

{{% /tab %}} {{% tab Python %}}

`OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf` suporta: `http/protobuf` e
`http/json` `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318`

{{% /tab %}} {{% tab Ruby %}}

`OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf` suporta: `http/protobuf`
`OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318`

{{% /tab %}} {{< /tabpane >}}

## Publicar sua Lambda

Publique uma nova versão da sua Lambda para implantar as novas alterações e
instrumentação.
