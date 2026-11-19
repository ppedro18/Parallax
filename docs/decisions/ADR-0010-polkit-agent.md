# ADR-0010: Agente de autenticación Polkit integrado en PARALLAX Shell

- **Estado:** Propuesto
- **Fecha:** 2026-11-19
- **Ámbito:** Autenticación administrativa, autorización y superficies de seguridad.
- **Relaciones:** ADR-0003 (Smithay), ADR-0005 (Toolkit UI), ADR-0006 (Procesos del Shell), ADR-0007 (IPC).

## 1. Contexto

PARALLAX necesita proporcionar una experiencia de autenticación administrativa coherente con su sistema de diseño.

En Linux, Polkit proporciona un mecanismo de autorización para operaciones privilegiadas solicitadas por aplicaciones y servicios.

Cuando una acción requiere autenticación, Polkit utiliza un agente de autenticación registrado en la sesión del usuario para presentar la solicitud y recoger la respuesta.

PARALLAX implementará su propio agente para evitar depender visualmente de diálogos externos y mantener una experiencia integrada con el Shell.

La implementación deberá respetar completamente el modelo de seguridad de Polkit.

## 2. Decisión

El PARALLAX Shell implementará un agente de autenticación Polkit mediante la interfaz:

`org.freedesktop.PolicyKit1.AuthenticationAgent`

El agente se registrará ante el servicio Polkit para la sesión gráfica correspondiente.

Cuando se solicite autenticación:

1. El agente recibirá la solicitud de autenticación.
2. Identificará la acción y el contexto disponible.
3. Presentará un diálogo de autenticación.
4. Recogerá la respuesta del usuario.
5. Transmitirá la respuesta mediante el protocolo de Polkit.
6. Gestionará el resultado y cerrará el diálogo.

**El Shell no verificará contraseñas, no ejecutará operaciones privilegiadas y no implementará su propio mecanismo de autorización.**

La decisión de autorización y la verificación de credenciales permanecerán bajo el control de Polkit y del mecanismo de autenticación configurado en el sistema, normalmente integrado con PAM.

## 3. Arquitectura

```text
Aplicación solicitante
        |
        v
     Polkit
        |
        v
 Authentication Agent
        |
        v
    px-shell
        |
        v
 Authentication Dialog
        |
        v
 Entrada del usuario
        |
        v
    Polkit
        |
        v
Mecanismo de autenticación
        |
        v
 Resultado de autorización
```

El Shell será responsable exclusivamente de la presentación y comunicación con Polkit.

El agente deberá respetar el ciclo de vida de las solicitudes de autenticación y las interfaces D-Bus correspondientes.

## 4. Registro del agente

El agente se registrará mediante el mecanismo oficial de Polkit.

Deberá:

- Registrarse para la sesión gráfica del usuario.
- Gestionar el identificador de sesión.
- Detectar solicitudes nuevas.
- Gestionar múltiples solicitudes de autenticación.
- Cancelar solicitudes correctamente.
- Liberar recursos al finalizar.
- Volver a registrarse después de reinicios del Shell.

No se implementará un protocolo propio que sustituya a la interfaz de autenticación de Polkit.

## 5. Diálogo de autenticación

El diálogo será una superficie propia del Shell.

Se utilizará layer-shell con nivel `overlay`, sujeto a las capacidades de la versión de Smithay y los protocolos disponibles.

### Información mostrada

El diálogo presentará, cuando esté disponible:

- Descripción de la acción solicitada.
- Identidad del usuario que solicita la operación.
- Nombre del ejecutable solicitante, si puede obtenerse de forma fiable.
- Identidad del usuario que debe autenticarse.
- Mensaje de autenticación proporcionado por Polkit.
- Indicador de autenticación en curso.
- Resultado o error de autenticación.

Los datos del solicitante se tratarán como información no confiable.

No se asumirá que el nombre del ejecutable es suficiente para demostrar la identidad de una aplicación.

El diálogo no deberá mostrar información sensible innecesaria.

### Comportamiento

- Entrada de contraseña protegida visualmente.
- Ocultación de caracteres introducidos.
- Botón para cancelar.
- Estados de carga y error.
- Prevención de envíos repetidos.
- Limpieza de campos al finalizar.
- Navegación por teclado.
- Accesibilidad mediante el sistema de componentes PARALLAX.

El Shell no almacenará credenciales en preferencias, archivos, logs ni telemetría.

## 6. Seguridad de la superficie

El diálogo de autenticación tendrá prioridad visual sobre las superficies ordinarias del Shell.

El compositor deberá aplicar una política específica:

- Mantener visible el diálogo mientras la solicitud esté activa.
- Proporcionar foco de teclado exclusivo durante la autenticación.
- Evitar que ventanas ordinarias intercepten la entrada.
- Evitar que otras superficies suplanten el diálogo.
- Impedir que el usuario interactúe accidentalmente con operaciones subyacentes.
- Restaurar el foco anterior después de cerrar el diálogo.

La prioridad de seguridad deberá implementarse en el compositor, no únicamente mediante propiedades visuales de layer-shell.

La autenticación deberá permanecer vinculada a la solicitud original de Polkit.

Una solicitud cancelada o finalizada no podrá reutilizarse.

## 7. Gestión de credenciales

El Shell no implementará autenticación PAM directamente.

Las credenciales serán transmitidas mediante el mecanismo previsto por el protocolo de Polkit y procesadas por el servicio y sus helpers autorizados.

Se aplicarán las siguientes reglas:

- No registrar contraseñas.
- No persistir contraseñas.
- No compartir credenciales entre solicitudes.
- Limpiar los campos y buffers de entrada al terminar.
- Cancelar la autenticación cuando el usuario cierre el diálogo.
- No reutilizar respuestas de solicitudes anteriores.
- No modificar el comportamiento de PAM.

La protección de memoria de las credenciales deberá evaluarse dentro de las limitaciones del toolkit y del mecanismo de entrada utilizado.

## 8. Múltiples solicitudes

El agente deberá soportar más de una solicitud de autenticación.

Se implementará una cola o mecanismo de coordinación para evitar diálogos superpuestos.

Cada solicitud mantendrá:

- Identificador de solicitud.
- Identidad de la sesión.
- Contexto del solicitante.
- Estado de autenticación.
- Estado de cancelación.
- Resultado final.

Las solicitudes no podrán compartir accidentalmente credenciales ni resultados.

Se priorizará una presentación clara que permita identificar qué operación está esperando autorización.

## 9. Recuperación

El agente principal estará integrado en `px-shell`.

Si el Shell termina inesperadamente, el usuario no deberá quedarse sin posibilidad de realizar operaciones administrativas.

Se implementará un agente mínimo de recuperación, independiente del Shell principal.

### Agente de recuperación

Se evaluará un proceso ligero de usuario o una sesión de recuperación que pueda registrar el agente Polkit cuando el Shell principal no esté disponible.

Deberá:

- Presentar una interfaz mínima.
- Registrar el agente para la sesión.
- Gestionar solicitudes de autenticación.
- Utilizar el mismo mecanismo oficial de Polkit.
- No ejecutar operaciones privilegiadas directamente.
- Ceder el control al agente principal cuando este vuelva a estar disponible.

No deberán registrarse simultáneamente dos agentes competidores para la misma sesión sin una política explícita de sustitución.

La transición deberá contemplar la cancelación de solicitudes pendientes y el registro correcto del nuevo agente.

El agente mínimo no deberá depender de componentes gráficos que solo existan dentro de `px-shell`.

## 10. Integración con el sistema

La implementación deberá ser compatible con:

- Polkit.
- D-Bus.
- PAM.
- systemd-logind.
- Wayland.
- Smithay.
- El sistema de superficies PARALLAX.

Se deberá comprobar el comportamiento con aplicaciones que utilicen operaciones administrativas mediante Polkit.

La interfaz no dependerá de que las aplicaciones solicitantes utilicen tecnologías específicas.

## 11. Pruebas de seguridad y funcionamiento

Se realizarán pruebas para:

### Funcionamiento normal

- Solicitud de autenticación.
- Autenticación correcta.
- Contraseña incorrecta.
- Cancelación.
- Solicitudes repetidas.
- Mensajes de error.

### Integridad de la solicitud

- Solicitud cancelada.
- Solicitud expirada.
- Solicitudes simultáneas.
- Cambio de foco.
- Desconexión D-Bus.
- Reinicio del servicio Polkit.

### Protección de la superficie

- Intento de abrir una ventana sobre el diálogo.
- Intento de robar el foco.
- Cambio de workspace.
- Cambio de monitor.
- Interacción con superficies subyacentes.
- Activación de accesibilidad.

### Recuperación

- Terminación inesperada del Shell.
- Registro del agente alternativo.
- Solicitud administrativa durante la recuperación.
- Reinicio del Shell.
- Sustitución del agente alternativo.

### Criterios de aceptación

- El agente se registra correctamente en Polkit.
- La autenticación funciona mediante el mecanismo configurado.
- El Shell no valida credenciales directamente.
- El diálogo presenta información verificable de la solicitud.
- Las credenciales no se registran ni persisten.
- Las solicitudes canceladas no pueden reutilizarse.
- El compositor protege el foco y la superficie.
- El agente alternativo funciona sin `px-shell`.
- La recuperación no reinicia innecesariamente la sesión gráfica.

## 12. Consecuencias

### Positivas

- Experiencia visual coherente.
- Integración con el Shell.
- Compatibilidad con el ecosistema Polkit.
- Sin necesidad de implementar autenticación propia.
- Separación entre presentación y autorización.
- Recuperación independiente del proceso principal.

### Negativas

- Dependencia de Polkit y D-Bus.
- Mayor responsabilidad de seguridad en el Shell.
- Necesidad de gestionar solicitudes concurrentes.
- Desarrollo y mantenimiento de un agente alternativo.
- Necesidad de integración específica con las políticas del compositor.

## 13. Revisión

Este ADR deberá revisarse si:

- La integración de Polkit cambia significativamente.
- Smithay no permite implementar las garantías de superficie necesarias.
- El agente alternativo presenta incompatibilidades.
- Aparecen problemas de accesibilidad.
- Se identifican vulnerabilidades en la gestión de credenciales o solicitudes.

## 14. Principio rector

**PARALLAX controla cómo se presenta la autenticación, pero nunca sustituye al mecanismo de seguridad del sistema.**

La interfaz puede ser propia; la confianza y la autorización deben permanecer bajo el control de Polkit y de los mecanismos de autenticación del sistema.