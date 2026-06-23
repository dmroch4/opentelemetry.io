---
title: Instrumentação Manual Lambda
weight: 11
description: Instrumente manualmente suas Lambdas com OpenTelemetry
default_lang_commit: f49ec57e5a0ec766b07c7c8e8974c83531620af3
---

Para linguagens não cobertas no documento de auto-instrumentação Lambda, a
comunidade não possui uma layer de instrumentação standalone.

Os usuários precisarão seguir o guia de instrumentação genérico para a linguagem
escolhida e adicionar a Lambda layer do Collector para enviar seus dados.

## Adicionar o ARN da Lambda layer do OTel Collector

Consulte o [guia da Lambda layer do Collector](../lambda-collector/) para
adicionar a layer à sua aplicação e configurar o Collector. Recomendamos
adicionar isso primeiro.

## Instrumentar a Lambda com OTel

Consulte o [guia de instrumentação por linguagem](/docs/languages/) sobre como
instrumentar manualmente sua aplicação.

## Publicar sua Lambda

Publique uma nova versão da sua Lambda para implantar as novas alterações e
instrumentação.
