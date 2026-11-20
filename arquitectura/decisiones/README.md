# Decisiones de arquitectura (ADRs)

Cada archivo registra **una** decisión: el contexto, lo que decidimos, las alternativas que descartamos y sus
consecuencias. Sirven para que quien llegue después entienda **por qué** las cosas son como son.

Estas son decisiones **transversales** (plataforma, ecosistema y forma de trabajo). Las decisiones propias de
una app viven en el `docs/adr/` de su repo.

## Índice

| # | Decisión | Estado |
|---|---|---|
| [0001](0001-lovable-construye-horizon-opera.md) | Las áreas construyen en Lovable y las apps del laboratorio de IA operan en Horizon | Propuesta · en uso |
| [0002](0002-perfiles-de-stack-aprobados.md) | Usar tres perfiles de stack aprobados en vez de stack libre por app | Propuesta · en uso |
| [0003](0003-contrato-de-app-horizon.md) | Toda app cumple un contrato común para desplegar y operar en Horizon | Propuesta · en uso |
| [0004](0004-identidad-con-portero.md) | La identidad la resuelve el portero (oauth2-proxy); las apps no implementan login | Propuesta · en uso |
| [0005](0005-orquestador-central-de-migracion-y-sync.md) | Migrar y sincronizar desde Lovable con un orquestador central y agentes de IA | Propuesta · en uso |
| [0006](0006-workflows-reutilizables-github-actions.md) | Implementar el orquestador con workflows reutilizables de GitHub Actions | Propuesta · en uso |
| [0007](0007-estado-del-sync-en-el-repo-destino.md) | Guardar el estado del sync en `.upstream-sync.yml` del repo destino | Propuesta · en uso |
| [0008](0008-acceso-minimo-del-orquestador.md) | El orquestador opera con acceso mínimo: lectura del origen y agentes sin commit | Propuesta · en uso |
| [0009](0009-revision-humana-obligatoria.md) | Ninguna entrega generada por IA llega a `develop` sin revisión humana | Propuesta · en uso |
| [0010](0010-documentacion-central-unica.md) | La documentación central vive en `base-conocimiento`, con hechos fechados | Propuesta |

## Estados

- **Propuesta**: escrita, falta la aprobación de Arquitectura. *En uso* indica que ya se aplica en la práctica.
- **Aceptada**: aprobada. Desde aquí el archivo no se edita.
- **Reemplazada**: un ADR posterior la sustituye (se indica cuál).

## Cómo agregar un ADR

1. Copiar [`plantilla.md`](plantilla.md) como `NNNN-titulo-corto.md` con el número siguiente.
2. Completar todas las secciones; las alternativas descartadas son obligatorias.
3. Agregar la fila al índice y abrir un PR.
