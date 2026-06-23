---
title: Runbooks de Alertas Prometheus
default_lang_commit: fe623719bc24346e9dcd77e9769026cf1c720cc5
---

## Regras do Manager

### ReconcileErrors

|             |                                                                                                                                                |
| ----------: | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| Significado | O OpenTelemetry Operator não consegue concluir a etapa de reconciliação, provavelmente por causa de um OpenTelemetryCollector mal configurado. |
|     Impacto | Nenhum impacto em implantações já em execução ou em novas corretas.                                                                            |
| Diagnóstico | Verifique os logs do manager para entender por que isso pode estar acontecendo.                                                                |
|   Mitigação | Descubra qual OpenTelemetryCollector está causando os erros e corrija a configuração.                                                          |

### WorkqueueDepth

|             |                                                                                                                             |
| ----------: | --------------------------------------------------------------------------------------------------------------------------- |
| Significado | A fila de trabalho do operator é maior que 0.                                                                               |
|     Impacto | Nenhum impacto se a profundidade da fila voltar para 0 rapidamente. Mais investigação é necessária se o problema persistir. |
| Diagnóstico | Verifique os logs do manager para entender por que isso pode estar acontecendo.                                             |
|   Mitigação | Isso pode ser causado por muitos erros. Aja com base no que os logs estão mostrando.                                        |
