# 0003 · Toda app cumple un contrato común para desplegar y operar en Horizon

- **Estado:** Propuesta · en uso
- **Fecha:** 2026-11-20
- **Responsables:** Transformación Digital e IA (por definir quién aprueba en Arquitectura)

## Contexto

Horizon impone condiciones que, si la app no cumple, hacen que no despliegue o falle en ejecución:

- Construye la imagen solo desde el `Dockerfile` de la raíz y **no pasa `--build-arg`**.
- El portero reenvía a la app en el **puerto 8080**; con otro puerto la respuesta es `502`.
- La base de datos se crea **después** de levantar el pod y sus credenciales se cargan a mano.
- Cambiar una variable reinicia el pod sin reconstruir la imagen, y el formulario de variables tiene límites
  (256 caracteres, sin espacios, sin `;`, sin valores vacíos).
- El escáner de imágenes (Trivy) bloquea herramientas innecesarias en el runtime.

## Decisión

Toda app cumple el contrato de Horizon. Lo esencial:

1. **Imagen**: un `Dockerfile` en la raíz, multi-stage, `ENV PORT=8080` y `EXPOSE 8080`, usuario
   `10001:0`, `HEALTHCHECK` sin `curl` y ningún `ARG` sin valor por defecto.
2. **Salud**: `GET /healthz` responde 200 siempre que el proceso viva; `GET /readyz` dice si está lista y,
   si no, por qué.
3. **Arranque en dos tiempos**: la app levanta HTTP de inmediato y en segundo plano espera la base, migra con
   candado y queda `lista`, sin reiniciar el pod.
4. **Configuración**: un único módulo lee el entorno, todo con defecto seguro, y **la imagen arranca sin
   ninguna variable** cargada.
5. **Identidad** según [0004](0004-identidad-con-portero.md).

## Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| Que cada app resuelva el despliegue a su manera | Cada despliegue se vuelve una investigación; se repiten los mismos errores |
| Fallar al arrancar si falta la base (fail-fast) | En Horizon la base llega después del pod: termina en CrashLoop |

## Consecuencias

- Las puertas automáticas verifican buena parte del contrato antes de integrar
  (`horizon-sync/scripts/chequeo-horizon.sh` y `humo-docker.sh`).
- El contrato es la fuente de las reglas de los agentes: si cambia aquí, se actualiza
  `horizon-sync/reglas/horizon-general.md`.
- Pendiente: llevar el contrato completo a `horizon/CONTRATO-APP.md` (hoy está en la guía, §4).

## Referencias

- [`horizon/GUIA-AGENTES-HORIZON.md`](../../horizon/GUIA-AGENTES-HORIZON.md) §2.6 y §4
