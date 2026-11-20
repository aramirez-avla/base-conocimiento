# 0006 · Implementar el orquestador con workflows reutilizables de GitHub Actions

- **Estado:** Propuesta · en uso
- **Fecha:** 2026-11-20
- **Responsables:** Transformación Digital e IA (por definir quién aprueba en Arquitectura)

## Contexto

- El orquestador de [0005](0005-orquestador-central-de-migracion-y-sync.md) tiene que correr para muchas
  apps, cada una en su propio repo de `avla-tech`.
- Las reglas, prompts y scripts van a cambiar seguido mientras aprendemos. Cada mejora debería llegar a todas
  las apps sin tocar cada repo.
- Los repos de Horizon ya usan GitHub y GitHub Actions.

## Decisión

- Toda la lógica vive en `avla-tech/horizon-sync` como **workflows reutilizables** (`workflow_call`):
  `migrar-plan`, `migrar-esqueleto`, `migrar-modulo`, `sincronizar` y `validar-pr`, más sus scripts, prompts
  y reglas.
- Cada repo destino solo tiene workflows cortos que llaman a `horizon-sync@main` (copiados de `plantillas/`).
- Para leer scripts, prompts y reglas desde el runner se usa una action compuesta (`.github/actions/central`),
  porque el `GITHUB_TOKEN` del repo que llama no puede clonar horizon-sync, que es privado.

## Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| Copiar los scripts en cada repo | Las copias se desalinean; cada mejora hay que llevarla repo por repo |
| Un servicio propio (servidor que orquesta) | Hay que desplegarlo, monitorearlo y darle credenciales amplias sobre todos los repos |
| Ejecutarlo a mano desde un equipo local | No deja trazabilidad y depende de una persona |

## Consecuencias

- Sin infraestructura propia: corre en los runners de GitHub.
- Una mejora en horizon-sync llega a todas las apps en el siguiente job.
- Cada ejecución queda en el repo de la app: el run, los artefactos y el PR.
- **Riesgo:** un error en `main` de horizon-sync afecta a todas las apps a la vez. Se mitiga con el workflow
  `ci` de horizon-sync y con cambios por PR.
- Requiere que horizon-sync permita su uso desde repos de la organización (*Settings → Actions → Access*).

## Referencias

- [`actualizacion-automatica/GUIA-HORIZON-SYNC.md`](../../actualizacion-automatica/GUIA-HORIZON-SYNC.md) §5.1
