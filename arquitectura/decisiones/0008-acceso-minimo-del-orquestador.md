# 0008 · El orquestador opera con acceso mínimo: lectura del origen y agentes sin commit

- **Estado:** Propuesta · en uso
- **Fecha:** 2026-11-20
- **Responsables:** Transformación Digital e IA (por definir quién aprueba en Arquitectura)

## Contexto

- El orquestador lee repos de Lovable que están fuera de `avla-tech` y escribe en los repos de Horizon.
- Los agentes de IA ejecutan comandos en el runner; un agente con permisos amplios podría modificar el flujo,
  hacer push o filtrar credenciales.

## Decisión

| Actor | Acceso | Cómo |
|---|---|---|
| Repo de Lovable | Solo lectura | GitHub App de solo lectura (`avla-lovable-lector`): token por ejecución, limitado a ese repo; el clon no guarda credenciales |
| Job del repo destino | Escritura solo en su repo | `GITHUB_TOKEN` con `contents: write` y `pull-requests: write` |
| Agente que escribe | Edita archivos y corre validaciones | Lista cerrada de herramientas; **sin commit ni push**: versiona el script |
| Agente revisor | Solo lectura | Sin herramientas de edición |
| Credenciales | Secretos de GitHub | Nunca en código, chats ni logs; `gitleaks` en cada entrega |

## Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| Token personal (PAT) para leer Lovable | Depende de una persona, no vence por ejecución y suele tener más alcance del necesario |
| Dar al agente permisos de git completos | Podría saltarse las puertas o modificar el propio flujo |

## Consecuencias

- Una falla del agente queda contenida en una rama y un PR que alguien revisa.
- **Pendiente:** la organización bloquea los PRs creados por `GITHUB_TOKEN`; hoy el PR se abre a mano desde un
  link. La propuesta es una GitHub App con permiso de escritura acotado a `avla-tech`.
- **Pendiente:** los secretos se cargan en cada repo destino; conviene moverlos a nivel organización.

## Referencias

- [`actualizacion-automatica/GUIA-HORIZON-SYNC.md`](../../actualizacion-automatica/GUIA-HORIZON-SYNC.md) §9
