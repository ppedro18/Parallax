# ADR-0015: Autorización de clientes Wayland por capacidad de socket
- Estado: Propuesto · Fecha: 2026-11-19 · Dueño: ingeniero-seguridad · Implementa: ingeniero-compositor
## Contexto
La propuesta inicial identificaba clientes privilegiados por ruta del ejecutable y security-context-v1. Identificar no es autorizar: la ruta y el PID tienen carreras y se pueden falsificar, `app_id` lo elige el cliente, y security-context-v1 sirve para restringir, no para conceder.
## Opciones
1. Lista blanca por ruta del ejecutable.
2. Capacidad por socket: el compositor entrega sockets privilegiados por fd a procesos que él lanza.
3. Consentimiento del usuario por cliente en tiempo de conexión.
## Decisión
Opción 2 para los componentes del sistema y xdg-desktop-portal (consentimiento del usuario) para las apps. Detalles en `.claude/agents/ingeniero-seguridad.md` y la matriz de protocolos en `ingeniero-compositor.md`.
## Consecuencias
+ La autorización no depende de datos que controla el cliente.
− Un componente privilegiado que muere debe relanzarlo el compositor (no systemd directamente), o hacerse mediante un socket que systemd pase con `OpenFile=` controlado por el compositor: se define en Fase 2.
− No hay aislamiento real entre apps no sandbox del mismo usuario (límite documentado).
## Revisión
Si Wayland estandariza un mecanismo de autorización de clientes ampliamente adoptado.

> **Nota de Tarea 010 (propuesta pendiente, no acepta este ADR).** Las revisiones 6–10 concluyen que `OpenFile=` abre rutas y el fd store solo acepta fds de la propia unidad, por lo que la alternativa de Consecuencias no es viable; se propone que el compositor relance los roles (D-18). Reglas de higiene de descriptores propuestas (D-22): orígenes duplicados con `F_DUPFD_CLOEXEC` a ≥ 10; fds 0–4 definidos explícitamente (0=`/dev/null`, 1–2=journal, 3=socket de rol, 4=sync); `posix_spawn_file_actions_addclosefrom_np(5)` como última acción (fallo ⇒ no hay `exec`); `envp` por lista blanca, nunca el `environ` del compositor; lista explícita de fds permitidos en intermediario y rol; `PR_SET_DUMPABLE=0` en el rol tras el `exec`. **No constituye aislamiento entre procesos del mismo UID.** Ver [consolidación §4.1](../project-management/consolidacion-fase0.md).
