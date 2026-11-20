# base-conocimiento

Documentación central de Transformación Digital e IA: cómo es la plataforma **Horizon**, qué decisiones de
arquitectura tomamos, cómo se construyen y operan las apps del laboratorio de IA y cómo funciona la
actualización automática desde Lovable.

Es la **única copia** de esta información. Los repos de cada app enlazan aquí en vez de repetirla.

## Índice

| Carpeta | Contenido | Estado |
|---|---|---|
| [`arquitectura/decisiones/`](arquitectura/decisiones/) | Decisiones de arquitectura (ADRs): una decisión por archivo | 10 ADRs |
| `arquitectura/ecosistema.md` | Mapa de servicios: hub, gateway IA, mini-apps y quién llama a quién | Pendiente |
| [`horizon/`](horizon/) | La plataforma Horizon | Ver abajo |
| [`actualizacion-automatica/`](actualizacion-automatica/) | Horizon Sync: migración y sincronización desde Lovable | Guía lista |
| `procesos/` | Ciclo de vida de una app: de Lovable a operación en Horizon | Pendiente |

### `horizon/`

| Documento | Contenido | Estado |
|---|---|---|
| [`GUIA-AGENTES-HORIZON.md`](horizon/GUIA-AGENTES-HORIZON.md) | Guía completa para agentes: plataforma, stacks, contrato, migración y despliegue | Lista; se separará por tema |
| `PLATAFORMA-HORIZON.md` | Hechos verificados de la plataforma, cada uno con fecha y app donde se verificó | Pendiente (sale de la guía, §1–2) |
| `CONTRATO-APP.md` | Lo que toda app debe cumplir para desplegar y operar | Pendiente (sale de la guía, §4) |
| `STACKS.md` | Perfiles de stack aprobados | Pendiente (sale de la guía, §3) |
| `ALTA-APP.md` | Checklist único para dar de alta una app nueva | Pendiente |
| `FRICCIONES.md` | Registro vivo de problemas de despliegue y dónde quedó documentada la solución | Pendiente |

### `actualizacion-automatica/`

| Documento | Contenido |
|---|---|
| [`GUIA-HORIZON-SYNC.md`](actualizacion-automatica/GUIA-HORIZON-SYNC.md) | Cómo funciona el orquestador, cómo está implementado y qué configurar en origen y destino |

## Reglas de mantenimiento

1. **Una sola copia.** Cada hecho vive en un único documento; los demás lo enlazan.
2. **Hechos con fecha y fuente.** Todo hecho de la plataforma lleva `verificado: <fecha>, <app>`. Lo no
   verificado se marca `SUPUESTO` y no se usa como instrucción.
3. **Las decisiones van en un ADR.** Un ADR no se edita una vez aceptado: se reemplaza con uno nuevo que lo
   indica en `Reemplaza a`.
4. **Cada despliegue con fricción deja rastro** en `horizon/FRICCIONES.md`, con el documento que se corrigió.
5. **Documento reemplazado se marca** con `OBSOLETO → ver <enlace>` al inicio, en vez de dejarlo convivir.
6. **Lo que se ejecuta vive en su repo.** Reglas, prompts y scripts de Horizon Sync están en
   `avla-tech/horizon-sync`; aquí se explican y se enlazan, no se copian.
7. **Sin secretos.** Ni valores de credenciales, ni llaves, ni tokens, tampoco como ejemplo.
8. **Cambios por PR**, con revisión del dueño de la documentación.
