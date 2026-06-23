---
title: Configuração do Collector Lambda
linkTitle: Lambda Collector Config
weight: 11
description: Adicione e configure a Lambda layer do Collector à sua Lambda
cSpell:ignore: ADOT awsxray configmap confmap regionalized
default_lang_commit: f49ec57e5a0ec766b07c7c8e8974c83531620af3
---

A comunidade OpenTelemetry oferece o Collector em uma Lambda layer separada das
layers de instrumentação para dar aos usuários máxima flexibilidade. Isso é
diferente da implementação atual do AWS Distribution of OpenTelemetry (ADOT),
que agrupa instrumentação e Collector juntos.

## Adicionar o ARN da Lambda layer do OTel Collector

Depois de instrumentar sua aplicação, você deve adicionar a Lambda layer do
Collector para coletar e enviar seus dados ao backend escolhido.

Encontre o
[release mais recente da layer do Collector](https://github.com/open-telemetry/opentelemetry-lambda/releases)
e use seu ARN após alterar a tag `<region>` para a região em que sua Lambda
está.

Nota: Lambda layers são um recurso regionalizado, o que significa que só podem
ser usadas na Região em que são publicadas. Certifique-se de usar a layer na
mesma região que suas funções Lambda. A comunidade publica layers em todas as
regiões disponíveis.

## Configurar o OTel Collector

A configuração da Lambda layer do OTel Collector segue o padrão OpenTelemetry.

Por padrão, a Lambda layer do OTel Collector usa o config.yaml.

### Definir a Variável de Ambiente para seu Backend Preferido

Nas configurações de variáveis de ambiente da Lambda, crie uma nova variável que
armazene seu token de autorização.

### Atualizar os Exportadores Padrão

No seu arquivo `config.yaml`, adicione seu(s) exportador(es) preferido(s) se
ainda não estiverem presentes. Configure seu(s) exportador(es) usando as
variáveis de ambiente que você definiu para seus tokens de acesso na etapa
anterior.

**Sem uma variável de ambiente definida para seus exportadores, a configuração
padrão suporta apenas a emissão de dados usando o exportador debug.** Aqui está
a configuração padrão:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: '0.0.0.0:4317'
      http:
        endpoint: '0.0.0.0:4318'

exporters:
  # NOTA: Antes da v0.86.0, use `logging` em vez de `debug`.
  debug:
    verbosity: detailed

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [debug]
    metrics:
      receivers: [otlp]
      exporters: [debug]
  telemetry:
    metrics:
      address: localhost:8888
```

## Publicar sua Lambda

Publique uma nova versão da sua Lambda para habilitar as alterações que você
fez.

## Configuração Avançada do OTel Collector

Encontre aqui a lista de componentes disponíveis suportados para configuração
personalizada. Para habilitar o debugging, você pode usar o arquivo de
configuração para definir o nível de log como debug. Veja o exemplo abaixo.

### Escolher seu Confmap Provider Preferido

As Lambda Layers OTel suportam os seguintes tipos de confmap providers: `file`,
`env`, `yaml`, `http`, `https` e `s3`. Para personalizar a configuração do
coletor OTel usando diferentes Confmap providers, consulte o
[documento de Confmap providers do Amazon Distribution of OpenTelemetry](https://aws-otel.github.io/docs/components/confmap-providers#confmap-providers-supported-by-the-adot-collector)
para mais informações.

### Criar um Arquivo de Configuração Personalizado

Aqui está um exemplo de arquivo de configuração `collector.yaml` no diretório
raiz:

```yaml
#collector.yaml no diretório raiz
#Defina uma variável de ambiente 'OPENTELEMETRY_COLLECTOR_CONFIG_URI' para '/var/task/collector.yaml'

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 'localhost:4317'
      http:
        endpoint: 'localhost:4318'

exporters:
  # NOTA: Antes da v0.86.0, use `logging` em vez de `debug`.
  debug:
  awsxray:

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [awsxray]
    metrics:
      receivers: [otlp]
      exporters: [debug]
  telemetry:
    metrics:
      address: localhost:8888
```

### Mapear seu Arquivo de Configuração Personalizado usando Variáveis de Ambiente

Uma vez que sua configuração do collector esteja definida por meio de um confmap
provider, crie uma variável de ambiente em sua função Lambda
`OPENTELEMETRY_COLLECTOR_CONFIG_URI` e defina o caminho da configuração em
relação ao confmap provider como seu valor. Por exemplo, se você estiver usando
um file configmap provider, defina seu valor como
`/var/task/<caminho>/<para>/<arquivo>`. Isso informará ao extension onde
encontrar a configuração do collector.

#### Configuração Personalizada do Collector Usando o CLI

Você pode definir isso pelo console Lambda ou pelo AWS CLI.

```bash
aws lambda update-function-configuration --function-name Function --environment Variables={OPENTELEMETRY_COLLECTOR_CONFIG_URI=/var/task/collector.yaml}
```

#### Definir Variáveis de Ambiente de Configuração pelo CloudFormation

Você pode configurar variáveis de ambiente também via template
**CloudFormation**:

```yaml
Function:
  Type: AWS::Serverless::Function
  Properties:
    ...
    Environment:
      Variables:
        OPENTELEMETRY_COLLECTOR_CONFIG_URI: /var/task/collector.yaml
```

#### Carregar Configuração de um Objeto S3

Carregar configuração do S3 exigirá que a role IAM associada à sua função inclua
acesso de leitura ao bucket relevante.

```yaml
Function:
  Type: AWS::Serverless::Function
  Properties:
    ...
    Environment:
      Variables:
        OPENTELEMETRY_COLLECTOR_CONFIG_URI: s3://<bucket_name>.s3.<region>.amazonaws.com/collector_config.yaml
```
