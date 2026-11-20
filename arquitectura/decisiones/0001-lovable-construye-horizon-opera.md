# 0001 · Las áreas construyen en Lovable y las apps del laboratorio de IA operan en Horizon

- **Estado:** Propuesta · en uso
- **Fecha:** 2026-11-20
- **Responsables:** Transformación Digital e IA (por definir quién aprueba en Arquitectura)

## Contexto

- Las áreas necesitan herramientas digitales y no cuentan con equipos de desarrollo propios. Con IA
  (Lovable, Claude Code) pueden construir y pilotear soluciones en días.
- Lo que se publica directamente desde Lovable queda fuera del control de la compañía: sin identidad
  corporativa, en infraestructura de terceros (Cloudflare) y con conectores propios de Lovable.
- En AVLA conviven dos mundos normativos: los **sistemas core** (pólizas, endosos; Java/Spring, Bitbucket,
  Jenkins, SSO corporativo) y las **apps internas del laboratorio de IA**. Exigirle a una app de laboratorio
  el proceso del core la vuelve lenta y cara; dejarla sin reglas la vuelve un riesgo.

## Decisión

- Las áreas **construyen y pilotean** sus apps en **Lovable**.
- Las apps que prueban valor **operan en Horizon**, el *Internal Developer Portal* del laboratorio de IA
  (`lab-qa`): repo privado en `avla-tech`, imagen Docker construida por Horizon, EKS, un subdominio por app y
  login Google `@avla.com` a través del portero.
- Los sistemas core siguen en la plataforma core de TI y sus normas. Las buenas prácticas generales del core
  (pruebas, dinero en decimal, protección de datos) también aplican en Horizon; su infraestructura no.

## Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| Usar lo publicado por Lovable ("Publish") | Sin identidad corporativa ni control sobre la infraestructura; no se audita |
| Que TI desarrolle cada solución | No escala al ritmo de las áreas; la necesidad se pierde en la traducción |
| Llevar las apps de laboratorio a la plataforma core | Proceso y stack del core desproporcionados para apps internas en piloto |

## Consecuencias

- Hay que **migrar** cada app de Lovable a Horizon y **mantenerla al día** mientras el área sigue
  trabajando en Lovable → [0005](0005-orquestador-central-de-migracion-y-sync.md).
- Horizon necesita un contrato común y stacks acotados para operar muchas apps con un equipo chico →
  [0002](0002-perfiles-de-stack-aprobados.md), [0003](0003-contrato-de-app-horizon.md).
- Límites de la plataforma que toda app hereda: timeout de ~30 s en CloudFront, WAF compartido, una réplica,
  disco efímero y red aislada de las APIs internas (ver `horizon/GUIA-AGENTES-HORIZON.md` §2).

## Referencias

- [`horizon/GUIA-AGENTES-HORIZON.md`](../../horizon/GUIA-AGENTES-HORIZON.md) §1 y §2
