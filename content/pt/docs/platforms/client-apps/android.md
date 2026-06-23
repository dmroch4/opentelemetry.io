---
title: Android
description: >-
  Use o OpenTelemetry em aplicações executadas na plataforma Android
weight: 10
vers:
  ot-android: 1.4.0
cSpell:ignore: inactivity
default_lang_commit: 68c29178b21e7ace970d27c5817a4edcff3ea9fb
---

O OpenTelemetry Android fornece observabilidade para aplicações Android nativas.
Construído sobre o ecossistema [OpenTelemetry Java](/docs/languages/java/), ele
oferece instrumentação automática, monitoramento de usuário real (RUM) e
capacidades de instrumentação manual adaptadas para ambientes móveis.

## Funcionalidades

O OpenTelemetry Android inclui estas capacidades principais:

- **Instrumentação Automática**: módulos integrados para padrões comuns do
  Android:
  - Ciclo de vida de Activity
  - Ciclo de vida de Fragment
  - Detecção de ANR (Application Not Responding)
  - Relatório de falhas (crash reporting)
  - Detecção de mudança de rede
  - Detecção de renderização lenta/congelada de frames
  - Tempo de inicialização
  - Orientação de tela
  - Eventos de clique em View
- **Gerenciamento de Sessão**: rastreie sessões de usuário com timeouts de
  inatividade e tempos máximos de sessão configuráveis.
- **Buffering Offline**: persistência em disco para armazenar dados de
  telemetria quando o dispositivo está offline, garantindo que não haja perda de
  dados durante interrupções de rede.
- **Redação de Atributos**: capacidade de redigir ou modificar atributos de span
  antes da exportação para conformidade com privacidade.

## Primeiros passos

### Pré-requisitos

- Android SDK 21 (Lollipop) ou superior
- Projeto Gradle usando Kotlin (Java pode ser possível)

### Configuração do Gradle

Adicione a dependência do OpenTelemetry Android Agent ao seu arquivo
`build.gradle.kts` do módulo app. Use o Bill of Materials (BOM) para gerenciar
versões:

```kotlin
dependencies {
    implementation(platform("io.opentelemetry.android:opentelemetry-android-bom:{{% param vers.ot-android %}}"))
    implementation("io.opentelemetry.android:android-agent")
}
```

> [!NOTE]
>
> Consulte os
> [releases do OpenTelemetry Android](https://github.com/open-telemetry/opentelemetry-android/releases)
> para obter a versão mais recente.

### Inicializar o agente

Inicialize o OpenTelemetry no método `onCreate()` da sua classe `Application`:

```kotlin
class MyApplication : Application() {
    lateinit var openTelemetryRum: OpenTelemetryRum

    override fun onCreate() {
        super.onCreate()
        openTelemetryRum = initializeOpenTelemetry(this)
    }
}

private fun initializeOpenTelemetry(context: Context): OpenTelemetryRum =
    OpenTelemetryRumInitializer.initialize(
        context = context,
        configuration = {
            httpExport {
                baseUrl = "https://your-collector-endpoint:4318"
                baseHeaders = mapOf("Authorization" to "Bearer <token>")
            }
            instrumentations {
                // Todas as instrumentações estão habilitadas por padrão.
                // Desabilite as específicas conforme necessário:
                slowRendering { enabled(false) }
            }
            session {
                backgroundInactivityTimeout = 15.minutes
                maxLifetime = 4.days
            }
        }
    )
```

## Configuração

O OpenTelemetry Android utiliza um DSL Kotlin para configuração, conforme
mostrado no exemplo de inicialização acima. A tabela a seguir descreve as opções
de configuração disponíveis:

### Opções de configuração

| Bloco                                     | Descrição                                               |
| ----------------------------------------- | ------------------------------------------------------- |
| `httpExport { baseUrl }`                  | URL do endpoint OTLP para exportar telemetria           |
| `httpExport { baseHeaders }`              | Headers personalizados para incluir nas requisições     |
| `globalAttributes`                        | Atributos adicionados a toda a telemetria               |
| `session { backgroundInactivityTimeout }` | Timeout de inatividade antes de iniciar uma nova sessão |
| `session { maxLifetime }`                 | Tempo máximo de vida da sessão                          |
| `instrumentations`                        | Configura módulos individuais de auto-instrumentação    |

## Instrumentação automática

O OpenTelemetry Android fornece módulos de instrumentação automática que você
pode habilitar ou desabilitar. Para informações detalhadas sobre cada
instrumentação, incluindo telemetria emitida e opções de configuração, consulte
a documentação vinculada.

### Ciclo de vida de Activity

Captura automaticamente spans para eventos do ciclo de vida de Activity
(`onCreate`, `onStart`, `onResume`, `onPause`, `onStop`, `onDestroy`). Consulte
[Instrumentação de Activity](https://github.com/open-telemetry/opentelemetry-android/blob/main/instrumentation/activity/README.md).

### Ciclo de vida de Fragment

Captura spans para eventos do ciclo de vida de Fragment, útil para rastrear
navegação em arquiteturas de single-activity. Consulte
[Instrumentação de Fragment](https://github.com/open-telemetry/opentelemetry-android/blob/main/instrumentation/fragment/README.md).

### Detecção de ANR

Detecta condições de Application Not Responding (ANR) e as reporta como spans,
ajudando a identificar problemas de bloqueio da thread de UI. Consulte
[Instrumentação de ANR](https://github.com/open-telemetry/opentelemetry-android/blob/main/instrumentation/anr/README.md).

### Relatório de falhas

Captura exceções não tratadas e as reporta com stack traces, permitindo
correlacionar falhas com sessões de usuário e traces. Consulte
[Instrumentação de Crash](https://github.com/open-telemetry/opentelemetry-android/blob/main/instrumentation/crash/README.md).

### Monitoramento de rede

Detecta mudanças de estado de rede e adiciona informações de conectividade à
telemetria, ajudando a entender as condições de rede durante erros. Consulte
[Instrumentação de Network](https://github.com/open-telemetry/opentelemetry-android/blob/main/instrumentation/network/README.md).

### Frames lentos e congelados

Monitora o desempenho de renderização de frames e reporta renderizações lentas
(>16ms) e frames congelados (>700ms) para ajudar a identificar gargalos de
desempenho na UI. Consulte
[Instrumentação de renderização lenta](https://github.com/open-telemetry/opentelemetry-android/blob/main/instrumentation/slowrendering/README.md).

## Instrumentação manual

Acesse a API OpenTelemetry para instrumentação manual:

```kotlin
val openTelemetry = openTelemetryRum.openTelemetry
val tracer = openTelemetry.getTracer("com.example.myapp")

val span = tracer.spanBuilder("my-operation")
    .startSpan()

try {
    span.makeCurrent().use {
        // Seu código aqui
    }
} finally {
    span.end()
}
```

## Instrumentação de cliente HTTP

Instrumente clientes OkHttp para rastrear requisições de rede:

```kotlin
val okHttpClient = OkHttpTelemetry.builder(openTelemetryRum.openTelemetry)
    .build()
    .newCallFactory(OkHttpClient.Builder().build())
```

## Boas práticas

### Restrições de recursos

Dispositivos móveis possuem recursos limitados. Considere estas boas práticas:

- **Exportações em lote**: o processamento em lote é habilitado por padrão para
  reduzir chamadas de rede e consumo de bateria.
- **Amostragem**: implemente estratégias de amostragem para reduzir o volume de
  dados mantendo telemetria representativa.
- **Buffering offline**: a persistência em disco é habilitada por padrão para
  lidar com conectividade intermitente.

### Considerações de privacidade

- Use redação de atributos para remover dados sensíveis antes da exportação.
- Considere os requisitos de consentimento do usuário para coleta de telemetria.
- Evite capturar informações de identificação pessoal (PII) em nomes de span ou
  atributos.

### Testes

Ao testar com um emulador, use `10.0.2.2` como endereço do host para alcançar o
collector na sua máquina local:

```kotlin
httpExport {
    baseUrl = "http://10.0.2.2:4318"
}
```

## Recursos

- [OpenTelemetry Android no GitHub](https://github.com/open-telemetry/opentelemetry-android)
- [Documentação do OpenTelemetry Java](/docs/languages/java/)
- [Convenções semânticas do Android](/docs/specs/semconv/registry/attributes/android/)
- [Aplicações de exemplo](https://github.com/open-telemetry/opentelemetry-android/tree/main/demo-app)

## Ajuda e feedback

Em caso de dúvidas, entre em contato via
[GitHub Issues](https://github.com/open-telemetry/opentelemetry-android/issues)
ou pelo canal
[#otel-android](https://cloud-native.slack.com/archives/C05J0T9K27Q) no
[CNCF Slack](https://slack.cncf.io/).
