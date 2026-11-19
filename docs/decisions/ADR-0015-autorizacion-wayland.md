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
