# 0002 · Usar tres perfiles de stack aprobados en vez de stack libre por app

- **Estado:** Propuesta · en uso
- **Fecha:** 2026-11-20
- **Responsables:** Transformación Digital e IA (por definir quién aprueba en Arquitectura)

## Contexto

- Cada app nace con el stack que eligió quien la construyó (o la herramienta de IA). Con más apps, cada stack
  distinto multiplica lo que hay que saber para desplegar, revisar y mantener.
- Los agentes de IA que migran y revisan código trabajan mejor con reglas concretas por stack que con reglas
  genéricas.

## Decisión

Toda app en Horizon usa **uno** de estos perfiles:

| Perfil | Para qué | Referencia |
|---|---|---|
| `node-tanstack-lovable` | Apps nacidas en Lovable: TanStack Start, Node 22, TypeScript estricto, zod | `lab-qa-ia-lab-06` |
| `node-monorepo` | Apps más grandes: npm workspaces, NestJS/Fastify + React/Vite, PostgreSQL con Drizzle, pg-boss | `lab-qa-ia-lab-08`, `lab-qa-ia-lab-01` |
| `python-angular` | FastAPI + Angular servida por el backend, SQLAlchemy + Alembic | `carpeta-tributaria`, `avla-miniapps-hub` |

Un stack fuera de estos perfiles requiere un ADR en el repo de la app y aprobación de Arquitectura.

## Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| Stack libre por app | Cada app es un caso especial para desplegar, revisar y mantener |
| Un único stack para todo | Las apps de Lovable ya vienen en TanStack y hay equipos y apps existentes en Python; forzar uno solo obliga a reescribir |

## Consecuencias

- Las reglas para agentes y las puertas automáticas se escriben **por perfil**
  (`horizon-sync/reglas/perfil-<stack>.md`), y el perfil se detecta solo o se declara en `.upstream-sync.yml`.
- `python-angular` sigue como borrador vivo; hay que cerrarlo.
- Agregar un perfil es una decisión nueva (ADR), no una excepción en un repo.

## Referencias

- [`horizon/GUIA-AGENTES-HORIZON.md`](../../horizon/GUIA-AGENTES-HORIZON.md) §3
- `avla-tech/horizon-sync`: `reglas/perfil-*.md`, `scripts/detectar-stack.sh`
