# Especificación: Pantalla de bloqueo y Greeter — PARALLAX

**Componentes:** `parallax-lock`, `parallax-greeter`  
**Estado:** Propuesto · Pendiente de aprobación  
**Capa:** C5 — Desktop Experience  
**Dependencias:** `parallax-compositor`, PAM, greetd, logind, `px-ui`.

## 1. Objetivo

Definir dos componentes distintos:

- **Lock:** protege una sesión de usuario ya iniciada.
- **Greeter:** permite seleccionar usuario e iniciar una sesión gráfica.

No comparten el mismo ciclo de vida ni el mismo contexto de seguridad.

El Lock es una frontera de seguridad dentro de una sesión activa. El Greeter pertenece al proceso de inicio de sesión gestionado por greetd.

## 2. Pantalla de bloqueo

### 2.1 Arquitectura

`parallax-lock` es un cliente Wayland que utiliza `ext-session-lock-v1`.

El compositor mantiene la autoridad sobre el estado Locked.

El cliente de bloqueo no puede desbloquear por sí mismo la sesión mediante la simple destrucción de su superficie o desconexión.

### 2.2 Activación

Fuentes:

- `Super+L`.
- Tiempo de inactividad configurado.
- Cierre de tapa, según política.
- `Session1.Lock`.
- Antes de suspensión.
- Solicitud explícita de usuario.

El bloqueo debe solicitarse antes de suspender cuando la política de energía así lo requiera.

La coordinación con logind debe respetar los tiempos de inhibitor delay y no bloquear indefinidamente la suspensión.

### 2.3 Estado

Estados conceptuales:

- Inactive.
- LockRequested.
- Locking.
- Locked.
- Authenticating.
- UnlockRequested.
- Unlocked.
- Recovery.

El estado real de bloqueo pertenece al compositor, no al cliente.

### 2.4 Autenticación

Utilizar PAM mediante un servicio específico, por ejemplo `parallax-lock`.

Reglas:

- No almacenar contraseñas.
- No registrar credenciales.
- No implementar autenticación propia.
- No ejecutar PAM desde un hilo gráfico bloqueante.
- Mantener mensajes de error genéricos cuando corresponda.
- Respetar la política PAM del sistema.

Contraseña obligatoria para v1.

Fprintd puede incorporarse en v1.x, sujeto a disponibilidad y validación de la política PAM.

### 2.5 Protección contra fuerza bruta

Utilizar la política de autenticación del sistema, incluyendo `pam_faillock` cuando esté configurado.

No duplicar contadores de intentos en la UI como mecanismo de seguridad.

Los retrasos deben aplicarse en la capa de autenticación, no únicamente mediante animaciones o timers del cliente.

## 3. Fallo del cliente de bloqueo

Si `parallax-lock` muere:

1. El compositor mantiene Locked.
2. Elimina la superficie del cliente fallido.
3. Presenta una superficie segura de emergencia.
4. No muestra contenido de aplicaciones.
5. Registra el incidente.
6. Solicita recuperación del cliente.
7. Exige autenticación válida antes de desbloquear.

El compositor debe tener una ruta de emergencia independiente del proceso Shell.

La pantalla sólida no sustituye el mecanismo de autenticación.

No permitir desbloqueo por:

- Timeout.
- Desconexión Wayland.
- Caída de Shell.
- Reinicio del cliente.
- Fallo de GPU.
- Suspensión.
- Reanudación.
- Cambio de VT.

## 4. Contenido de Lock

Elementos configurables:

- Reloj.
- Fecha.
- Estado de batería.
- Indicador de red, sin información sensible.
- Notificaciones.
- Controles multimedia opcionales.

### 4.1 Notificaciones

Modos:

- `none`
- `count`
- `content`

El modo `content` debe respetar las preferencias de privacidad por aplicación.

No mostrar información sensible por defecto.

### 4.2 Multimedia

Los controles multimedia son opcionales.

No deben permitir ejecutar acciones peligrosas ni exponer datos de sesión.

## 5. Seguridad visual

- No mostrar ventanas ordinarias.
- No mostrar miniaturas de aplicaciones.
- No permitir interacción con superficies de sesión.
- No permitir que overlays de Shell se sitúen sobre el bloqueo.
- No permitir que un cliente ordinario simule la pantalla de bloqueo.
- Mantener el bloqueo al cambiar de VT.
- Mantener el bloqueo durante suspensión y reanudación.

El compositor debe aplicar la prioridad de seguridad definida en la arquitectura de superficies.

## 6. Greeter

### 6.1 Arquitectura

`parallax-greeter` se integra con greetd.

Se ejecuta en el contexto de usuario y sesión configurado por greetd, sin privilegios administrativos innecesarios.

El compositor utilizado para el Greeter puede ser una instancia mínima del compositor PARALLAX:

`parallax-compositor --greeter`

Debe utilizar una configuración reducida y separada de la sesión normal.

No asumir que el Greeter puede reutilizar el proceso compositor de una sesión de usuario.

### 6.2 Funcionalidades

- Selección de usuario.
- Inicio de sesión.
- Selección de sesión.
- Idioma y distribución de teclado.
- Opciones de accesibilidad.
- Apagar.
- Reiniciar.

### 6.3 Sesiones

El Greeter presenta únicamente sesiones instaladas y autorizadas.

No aceptar rutas arbitrarias ni comandos suministrados por el usuario.

La selección de sesión se resuelve mediante la configuración de greetd y el catálogo de sesiones.

### 6.4 Usuarios

La enumeración de usuarios debe seguir la política de privacidad del sistema.

No exponer cuentas de servicio innecesarias.

Permitir el ingreso manual cuando la política lo requiera.

### 6.5 Accesibilidad

- Teclado completo.
- Focus visible.
- Escalado de texto.
- Alto contraste.
- Orca cuando el entorno lo permita.
- Teclado en pantalla si se define su integración.
- Compatibilidad con distintos layouts de teclado.

## 7. Recuperación

El Lock y el Greeter deben tener servicios systemd independientes.

La caída de Greeter no debe afectar una sesión activa.

La caída de Lock no debe desbloquear una sesión.

El reinicio de componentes debe conservar las condiciones de seguridad.

## 8. Pruebas y aceptación

### Lock

- [ ] Bloqueo manual.
- [ ] Bloqueo por inactividad.
- [ ] Bloqueo antes de suspensión.
- [ ] PAM correcto.
- [ ] PAM incorrecto.
- [ ] Retraso por intentos fallidos.
- [ ] Muerte del lock client.
- [ ] Muerte de Shell.
- [ ] Fallo del renderer.
- [ ] Desconexión de monitor.
- [ ] VT switch.
- [ ] Suspensión y reanudación.
- [ ] No existe desbloqueo implícito.
- [ ] Notificaciones respetan privacidad.

### Greeter

- [ ] Inicio mediante greetd.
- [ ] Selección de usuario.
- [ ] Selección de sesión.
- [ ] Layout de teclado.
- [ ] Apagar y reiniciar.
- [ ] Accesibilidad.
- [ ] No permite comandos arbitrarios.
- [ ] No utiliza privilegios innecesarios.

## 9. Decisiones pendientes

1. Definir PAM service file.
2. Definir política de usuarios visibles.
3. Definir recuperación del Lock.
4. Definir soporte biométrico.
5. Definir accesibilidad del Greeter.
6. Definir compositor mínimo y su configuración.
7. Definir tratamiento de notificaciones sensibles.

**Estado: Propuesto. Pendiente de aprobación.**