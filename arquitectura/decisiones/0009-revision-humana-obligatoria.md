# 0009 · Ninguna entrega generada por IA llega a `develop` sin revisión humana

- **Estado:** Propuesta · en uso
- **Fecha:** 2026-11-20
- **Responsables:** Transformación Digital e IA (por definir quién aprueba en Arquitectura)

## Contexto

- Los agentes de IA migran y portan código de forma autónoma. Las puertas automáticas y el agente revisor
  detectan mucho, pero no garantizan que el cambio haga lo que el área quería.
- Las apps operan con datos y procesos reales de las áreas.

## Decisión

- Toda entrega del orquestador (plan, esqueleto, módulo o sync) llega como **rama + PR** hacia `develop`.
- **Una persona revisa e integra** cada PR. Nada se escribe directo en `develop`.
- El PR declara su estado: `ok` (puertas en verde y revisión sin bloqueantes) o `[necesita revisión]` (no se
  resolvió en 2 rondas), e incluye el informe del agente y la revisión automática.

## Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| Integrar automáticamente cuando las puertas están en verde | Las puertas verifican forma y pruebas, no la intención del negocio |
| Revisar solo una muestra de los PRs | Un error de lógica de negocio pasaría sin que nadie lo vea |

## Consecuencias

- La revisión humana es el límite de velocidad del flujo. Se mitiga con PRs chicos (un módulo o un sync a la
  vez) y con el informe que acompaña cada PR.
- Qué mirar en un PR de sync: la intención del cambio original, que la lógica de negocio quede en el backend,
  que las pruebas se porten y pasen, y la revisión automática.

## Referencias

- [`actualizacion-automatica/GUIA-HORIZON-SYNC.md`](../../actualizacion-automatica/GUIA-HORIZON-SYNC.md) §3 y §8
