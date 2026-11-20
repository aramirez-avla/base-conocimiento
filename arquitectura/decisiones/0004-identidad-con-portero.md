# 0004 · La identidad la resuelve el portero (oauth2-proxy); las apps no implementan login

- **Estado:** Propuesta · en uso
- **Fecha:** 2026-11-20
- **Responsables:** Transformación Digital e IA (por definir quién aprueba en Arquitectura)

## Contexto

- Horizon pone delante de cada app un **portero**: un sidecar `oauth2-proxy` con login Google `@avla.com`
  que reenvía a la app el correo del usuario en el header `X-Forwarded-Email`.
- Las apps hechas con IA tienden a traer login propio (contraseñas, Supabase Auth, JWT), que habría que
  asegurar y mantener en cada una.

## Decisión

- Las apps **no implementan login**: ni contraseñas, ni OAuth, ni JWT propio, ni cookies de sesión.
- Leen el usuario de `X-Forwarded-Email`. Todo endpoint de datos exige usuario, salvo `/healthz` y `/readyz`.
- El rol `admin` se asigna por la variable `ADMIN_EMAILS`; roles más finos van en la base de la app.
- Rol y propiedad se verifican en el servidor en cada endpoint.
- El modo desarrollo sin portero solo se acepta en ambientes `dev`, `local` o `test`; en cualquier otro, la
  app aborta al arrancar.

## Alternativas descartadas

| Alternativa | Por qué no |
|---|---|
| Login propio en cada app | Superficie de ataque y mantenimiento repetidos en cada app |
| JWT del SSO corporativo (como en el core) | Es la norma del mundo core; Horizon ya entrega la identidad a través del portero |

## Consecuencias

- Un solo login sirve para todas las apps del laboratorio; cerrar sesión cierra todas.
- **El portero se activa al otorgar el primer acceso**: antes de eso la URL está abierta y el header es
  falsificable, así que el primer acceso se concede inmediatamente después del deploy.
- Las llamadas de backend a backend entre apps chocan con el portero. Solo hay dos excepciones soportadas:
  el gateway IA (Bearer) y el hub (cookie).
- Al migrar desde Lovable, el login del origen (propio, Google OAuth, Supabase Auth o JWT) se reemplaza por el
  portero (`horizon-sync/reglas/metodologia-migracion.md`).

## Referencias

- [`horizon/GUIA-AGENTES-HORIZON.md`](../../horizon/GUIA-AGENTES-HORIZON.md) §2.2 y §4.4
