# Especificación: PARALLAX Terminal

**Aplicación:** `parallax-terminal`  
**Estado:** Propuesto · Pendiente de aprobación  
**Capa:** C6 — Native Apps  
**Dependencias:** `px-ui`, PTY, `rustix`, systemd user scopes y motor de terminal.

## 1. Objetivo

PARALLAX Terminal es un emulador de terminal nativo, integrado con el sistema de diseño y las capacidades de sesión de PARALLAX.

Debe proporcionar compatibilidad sólida con aplicaciones de consola existentes, bajo consumo de recursos y una experiencia configurable.

No es un shell propio: ejecuta el shell configurado por el usuario, como Bash, Zsh o Fish.

## 2. Arquitectura

Separar:

1. Emulación del terminal.
2. Gestión PTY.
3. Renderizado.
4. Gestión de sesiones.
5. Interfaz.
6. Configuración.

```text
parallax-terminal
│
├── px-ui
│
├── Terminal View
│   ├── Glyph Atlas
│   ├── Selection
│   └── Scrollback
│
├── Terminal Engine
│   ├── Parser
│   ├── Screen State
│   ├── Cursor
│   └── Modes
│
├── PTY Manager
│
└── Session Manager
    └── systemd user scope
```

## 3. Motor de emulación

Evaluar:

- `alacritty_terminal`.
- `vte`.

No tratarlos como dependencias intercambiables: sus responsabilidades, APIs y modelos de emulación son diferentes.

La elección debe basarse en:

- Compatibilidad.
- Mantenimiento.
- Licencia.
- Rendimiento.
- Integración.
- Capacidad de pruebas.

No implementar un parser ANSI propio en v1.

## 4. PTY

Utilizar `rustix::pty` cuando cubra las necesidades requeridas.

Cada pestaña o panel puede disponer de su propia PTY.

El proceso inicial será `$SHELL` del usuario.

Si la variable no está definida o es inválida, utilizar el shell configurado en la cuenta del usuario.

No ejecutar comandos arbitrarios por concatenación de strings.

### 4.1 Ciclo de vida

- Crear PTY.
- Configurar terminal.
- Iniciar proceso.
- Enviar input.
- Recibir output.
- Redimensionar.
- Finalizar.
- Limpiar recursos.

### 4.2 systemd scopes

Cada sesión de terminal se asocia a un scope de usuario cuando sea viable.

Debe definirse:

- Identificador de scope.
- Propiedad de la PTY.
- Variables de entorno.
- CWD.
- Gestión de cierre.
- Recuperación.

La caída de la UI no debe provocar la pérdida accidental de procesos si la política de sesión permite mantenerlos.

La persistencia de procesos tras cerrar Terminal debe ser configurable y explícita.

## 5. Renderizado

El texto se renderiza mediante `px-ui` o una integración de bajo nivel con su backend.

Características:

- Atlas de glifos.
- Cache de fuentes.
- Renderizado GPU.
- Scrollback.
- Selección.
- Cursor.
- Colores.
- Ligaduras opcionales.

No duplicar el motor tipográfico de `px-ui`.

La integración debe respetar shaping, fallback y escalado.

## 6. Funcionalidades

### 6.1 Pestañas

- Crear.
- Cerrar.
- Reordenar.
- Cambiar.
- Restaurar configuración de sesión.

Atajos:

- `Ctrl+Shift+T`: nueva pestaña.
- `Ctrl+Shift+W`: cerrar pestaña.

### 6.2 Paneles

División horizontal y vertical.

Cada panel dispone de:

- PTY independiente.
- Tamaño.
- Foco.
- Estado.
- CWD.

El redimensionamiento debe actualizar el tamaño del terminal mediante `TIOCSWINSZ` o el mecanismo apropiado.

### 6.3 Perfiles

Cada perfil puede definir:

- Shell.
- Directorio inicial.
- Colores.
- Fuente.
- Tamaño.
- Variables de entorno.
- Comportamiento del cursor.
- Scrollback.

Las variables sensibles no deben mostrarse en exportaciones de perfiles.

### 6.4 Scrollback

- 100 000 líneas por defecto.
- Límite configurable.
- Memoria controlada.
- Compresión opcional.
- Búsqueda.
- Selección.

El límite debe definirse también en bytes, no únicamente en cantidad de líneas, porque una línea puede contener cantidades muy distintas de datos.

### 6.5 Búsqueda

- Texto.
- Regex.
- Siguiente.
- Anterior.
- Resaltado.

La búsqueda no debe bloquear el renderizado.

### 6.6 Integraciones

- OSC 8: hipervínculos.
- OSC 133: marcas de prompt y comandos.
- Estado de salida cuando exista integración.
- Abrir carpeta en Files.
- Notificación al terminar comando largo en pestaña no visible.

No inferir de forma insegura el estado de un comando cuando el shell no proporciona integración válida.

## 7. Restauración

Al reabrir Terminal puede restaurar:

- Pestañas.
- Paneles.
- Perfiles.
- Directorios de trabajo.
- Tamaños.

No restaurar procesos de forma automática.

La restauración de una sesión debe crear nuevas PTY y nuevos shells.

Si una aplicación necesita persistir su propio estado, debe utilizar su mecanismo de recuperación.

## 8. Seguridad

- Ejecutar como usuario.
- No utilizar root.
- Respetar permisos del filesystem.
- Validar variables de entorno.
- Evitar inyección en comandos.
- Limitar el acceso de plugins o integraciones.
- No registrar credenciales ni input sensible.
- Gestionar cierre de procesos.

El terminal es una superficie de ejecución de código del usuario; su aislamiento depende de las políticas del sistema y del usuario, no de ocultar el prompt.

## 9. Rendimiento

- Renderizado incremental.
- Damage tracking.
- Cache de glifos.
- Scrollback limitado.
- E/S asíncrona.
- No bloquear UI.
- No reconstruir toda la pantalla por cada carácter.
- Medir throughput y latencia.

## 10. Pruebas

- `vttest` básico.
- Colores.
- Cursor.
- Alternate screen.
- Scrollback.
- Resize.
- Selección.
- Clipboard.
- Unicode.
- Bidi.
- CJK.
- Emoji.
- OSC 8.
- OSC 133.
- Procesos largos.
- Desconexión de PTY.
- Caída de UI.
- Cierre de pestañas.
- Restauración.

## 11. Aceptación

- [ ] Pasa el conjunto definido de pruebas `vttest`.
- [ ] Throughput igual o superior a foot en el benchmark reproducible de `cat` de 1 GB.
- [ ] Latencia de tecla inferior a 10 ms en HW-A para el escenario definido.
- [ ] No hay bloqueo de UI durante salida intensiva.
- [ ] Scrollback respeta límites.
- [ ] Pestañas y paneles funcionan.
- [ ] Integración systemd funciona.
- [ ] Unicode e IME funcionan.
- [ ] Restauración no revive procesos arbitrariamente.

Comparar versiones, hardware, fuentes, resolución y condiciones del benchmark.

## 12. Decisiones pendientes

1. Seleccionar motor de terminal.
2. Definir persistencia de sesiones.
3. Definir política de scopes.
4. Definir tamaño máximo de scrollback.
5. Definir soporte de ligaduras.
6. Definir integración OSC.
7. Definir API de perfiles.

**Estado: Propuesto. Pendiente de aprobación.**