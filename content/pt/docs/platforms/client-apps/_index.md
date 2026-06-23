---
title: Aplicações client-side
description: >-
  Use o OpenTelemetry em aplicações controladas pelo usuário final, executadas
  em dispositivos como celulares, computadores desktop e quiosques de varejo.
aliases: [android, client-apps/overview]
cSpell:ignore: contentful
default_lang_commit: 6cebc46de450dd44481a8a6f17c9b3d6f04aa0f2
---

As aplicações client-side apresentam desafios únicos de observabilidade em
comparação com cargas de trabalho server-side. Essas aplicações rodam em
dispositivos que você não controla, com condições de rede, capacidades de
hardware e comportamentos de usuário variados.

## Por que a observabilidade client-side é importante

O monitoramento tradicional server-side fornece visibilidade dos seus sistemas
backend, mas perde o panorama completo da experiência do usuário. A
observabilidade client-side ajuda você a:

- **Entender a experiência real do usuário**: veja tempos de carregamento reais,
  taxas de quadros e responsividade conforme os usuários os experimentam.
- **Depurar problemas em contexto**: correlacione erros com características do
  dispositivo, condições de rede e ações do usuário.
- **Rastrear transações de ponta a ponta**: conecte operações client-side com
  traces do backend para distributed tracing completo.
- **Monitorar a saúde da aplicação em escala**: agregue telemetria em toda a sua
  base de usuários para identificar padrões e tendências.

## Principais diferenças em relação à instrumentação server-side

Ao instrumentar aplicações client, considere estes fatores:

- **Restrições de recursos**: dispositivos móveis e navegadores possuem CPU,
  memória e bateria limitados. A coleta de telemetria deve ser eficiente e
  evitar impactar o desempenho da aplicação.
- **Variabilidade de rede**: usuários podem ter conectividade lenta,
  intermitente ou inexistente. Implemente buffering offline e exportações em
  lote para lidar com a instabilidade da rede.
- **Gerenciamento de sessão**: rastreie sessões de usuário para agrupar
  telemetria relacionada e entender as jornadas do usuário em múltiplas
  execuções da aplicação.
- **Privacidade e consentimento**: aplicações client frequentemente coletam
  dados sujeitos a regulamentações de privacidade. Planeje para minimização de
  dados, gerenciamento de consentimento e redação de atributos.
- **Volume de dados**: com potencialmente milhões de usuários, estratégias de
  amostragem tornam-se essenciais para gerenciar custos mantendo telemetria
  representativa.

## Real User Monitoring (RUM)

O OpenTelemetry suporta padrões de Real User Monitoring (RUM) que capturam como
usuários reais experimentam sua aplicação:

- **Desempenho de carregamento de página/tela**: tempo até o primeiro byte,
  primeira renderização com conteúdo e conclusão total do carregamento.
- **Interações do usuário**: eventos de clique, padrões de navegação e envios de
  formulários.
- **Erros e falhas**: exceções não tratadas, eventos ANR e taxas de erro.
- **Carregamento de recursos**: tempo de requisição de rede, taxas de acerto de
  cache e tamanhos de recursos.

## Conectando ao seu backend

A telemetria do cliente torna-se mais valiosa quando conectada aos seus traces
do backend. Propagate o contexto de trace através das suas requisições HTTP para
manter visibilidade de ponta a ponta:

```text
Client App → API Gateway → Backend Services → Database
    │              │              │              │
    └──────────────┴──────────────┴──────────────┘
                 Correlated Traces
```

Configure seu SDK cliente para injetar headers de trace (`traceparent`,
`tracestate`) e certifique-se de que seus serviços backend propagam esse
contexto através de suas operações.
