# 0007 · Guardar el estado del sync en `.upstream-sync.yml` del repo destino

- **Estado:** Propuesta · en uso
- **Fecha:** 2026-11-20
- **Responsables:** Transformación Digital e IA (por definir quién aprueba en Arquitectura)

## Contexto

Para traer solo lo nuevo, el orquestador necesita saber, por cada app: de qué repo de Lovable viene, qué rama
sigue, con qué perfil de stack se trabaja y **hasta qué commit ya se sincronizó**. Ese dato tiene que avanzar
solo cuando el cambio realmente entró, no cuando se intentó.

## Decisión

El estado vive en `.upstream-sync.yml`, en la raíz del repo destino y versionado junto al código:
`source_repo`, `source_branch`, `stack`, `plan_sha`, `baseline_sha`, `modulos_portados` e `ignore_paths`.

- El PR del sync trae el nuevo `baseline_sha`: **avanza solo al integrarse**.
- Si existe una rama `sync/*` sin integrar, el siguiente sync se detiene.
- Migrar y sincronizar comparten un grupo de concurrencia: nunca corren dos a la vez en el mismo repo.

## Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| Base de datos o archivo central del orquestador | Infraestructura extra y un estado que puede divergir del código real |
| Variables o etiquetas del repo | No quedan en el historial ni en el PR; difíciles de revisar |
| Recalcular comparando código | Lovable y Horizon divergen a propósito; no hay forma confiable de inferirlo |

## Consecuencias

- El repo destino es la **única fuente de verdad**; el orquestador no guarda estado propio.
- Cada avance queda auditado en git, con su PR y su revisión.
- Si un PR se cierra sin integrar, el siguiente intento parte del mismo punto (hay que borrar la rama).
- Para probar como incremental un cambio que ya estaba en el origen, se migra desde un commit anterior y luego
  se ajusta `source_branch`.

## Referencias

- [`actualizacion-automatica/GUIA-HORIZON-SYNC.md`](../../actualizacion-automatica/GUIA-HORIZON-SYNC.md) §5.2
