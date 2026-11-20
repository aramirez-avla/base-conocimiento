# 0010 · La documentación central vive en `base-conocimiento`, con hechos fechados

- **Estado:** Propuesta
- **Fecha:** 2026-11-20
- **Responsables:** Transformación Digital e IA (por definir el dueño de la documentación)

## Contexto

- El diagnóstico del plan de mitigación de Horizon concluyó que buena parte de las fricciones de despliegue
  vienen de **documentación fragmentada**: el mismo hecho escrito distinto en varios repos, documentos viejos
  conviviendo con los nuevos y hechos de la plataforma sin fecha ni fuente.
- Ese plan dejó pendiente **dónde vive la guía central**. Recomendaba un repo dedicado en vez de una carpeta
  dentro de una app (hub o gateway), porque esas apps mezclarían su despliegue con el de todas.

## Decisión

- La documentación central vive en el repo **`base-conocimiento`**: plataforma Horizon, contrato de app,
  stacks, decisiones de arquitectura, procesos y Horizon Sync.
- Cada repo de app documenta **solo lo propio** y enlaza aquí.
- Se rige por las reglas de mantenimiento del [README](../../README.md): una sola copia, hechos con fecha y
  fuente, decisiones en ADRs, fricciones registradas y documentos reemplazados marcados como obsoletos.

## Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| Carpeta en `avla-miniapps-hub/docs` | El hub es una app más; mezcla su despliegue con el de todas |
| Carpeta en `gateway-ia-avla/docs` | Mismo problema, y no todas las apps usan IA |
| Que cada repo documente todo | Es la situación que generó las fricciones |

## Consecuencias

- Hay que mover a este repo lo que hoy está suelto: separar `GUIA-AGENTES-HORIZON.md` por tema y traer la
  estrategia de despliegue en LAN y el mapa de llamadas entre servicios.
- Las reglas que leen los agentes (`horizon-sync/reglas/`) deben derivarse de estos documentos, no al revés.
- Falta definir el **dueño** de la documentación, que revisa los PRs.

## Referencias

- `PLAN-MITIGACION-HORIZON.md` §1, §2.3 y §4
