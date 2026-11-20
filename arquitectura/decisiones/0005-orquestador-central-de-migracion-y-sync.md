# 0005 · Migrar y sincronizar desde Lovable con un orquestador central y agentes de IA

- **Estado:** Propuesta · en uso
- **Fecha:** 2026-11-20
- **Responsables:** Transformación Digital e IA (por definir quién aprueba en Arquitectura)

## Contexto

- Por [0001](0001-lovable-construye-horizon-opera.md), cada app se construye en Lovable y opera en Horizon.
- Las áreas **siguen mejorando** sus apps en Lovable después de la migración. Cada funcionalidad nueva había
  que portarla a mano al código de Horizon.
- Con más áreas y más proyectos, el volumen de cambios por sincronizar crece más rápido que la capacidad de
  portarlos a mano. Mientras un cambio espera, Lovable y Horizon muestran cosas distintas, y la calidad del
  porte depende de quién lo hace.

## Decisión

Un orquestador central, **Horizon Sync** (`avla-tech/horizon-sync`), hace dos cosas:

1. **Migración inicial**, una vez por app: plan, esqueleto y módulos, cada uno en su PR.
2. **Sincronización incremental**, cada vez que el área lo pida: toma solo los commits nuevos de Lovable
   desde la última sincronización y los porta adaptados a Horizon.

El trabajo lo hacen agentes de IA (Claude Code) en un ciclo fijo: un agente que escribe, puertas automáticas
y un agente revisor, con hasta 2 rondas de corrección.

## Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| Portar a mano cada cambio | No escala con el volumen de proyectos; calidad dispareja |
| Migrar una vez y congelar Lovable | Las áreas dejan de poder iterar, que es justo lo que da valor |
| Que el área trabaje directo en el repo de Horizon | Pierde la herramienta con la que sabe construir |
| Aplicar el diff de Lovable tal cual | El código de Horizon ya es distinto (portero, contrato, stack); el diff no aplica |

## Consecuencias

- Las mismas reglas de Horizon se aplican igual en todas las apps, sin depender de quién porta.
- Lo que existe antes del plan entra por la migración; solo lo posterior entra por el sync.
- Se valida por etapas: en `lab-qa-ia-lab-08` se migraron 9 módulos y se sincronizó un primer cambio
  (noviembre 2026). Falta validar con una regla de negocio (prueba en `tax-tidy-hub`).
- Las decisiones de implementación están en [0006](0006-workflows-reutilizables-github-actions.md) a
  [0009](0009-revision-humana-obligatoria.md).

## Referencias

- [`actualizacion-automatica/GUIA-HORIZON-SYNC.md`](../../actualizacion-automatica/GUIA-HORIZON-SYNC.md)
