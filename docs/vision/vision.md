# Visión y alcance de PARALLAX OS

## Problema
Las distribuciones Linux de escritorio ofrecen experiencias ensambladas a partir de piezas de distintos proyectos (GNOME, KDE, wlroots + barras de terceros). El resultado es incoherente en diseño, comportamiento y consumo, y el usuario percibe "Linux con un tema", no un producto.

## Propuesta
PARALLAX es un sistema operativo de escritorio con una experiencia propia de extremo a extremo —compositor, shell, servicios y aplicaciones— escrita en Rust sobre el kernel Linux y la infraestructura madura del ecosistema (systemd, Mesa, PipeWire, NetworkManager, BlueZ).

**Regla de oro:** construimos lo que el usuario ve y lo que decide cómo se comporta el sistema; reutilizamos lo que ya funciona bien debajo.

## Usuarios objetivo (v1)
1. Desarrolladores y usuarios técnicos con portátiles/sobremesas x86_64 modernos (Intel/AMD, NVIDIA como objetivo secundario).
2. Usuarios que valoran estética y eficiencia y hoy usan macOS o un tiling WM configurado a mano.

**Fuera de v1:** tablets/táctil como modo primario, ARM64 (planificado v2), servidores, móviles, juegos como caso optimizado.

## Alcance v1.0
| Dentro | Fuera |
|---|---|
| Compositor Wayland + WM floating/tiling | Kernel propio o fork |
| Shell: topbar, dock, launcher, widgets, notificaciones, centro de control, bloqueo | Suite ofimática, navegador propio |
| Files, Terminal, Settings, Task Manager, Observatory, Capture | Tienda de apps con pagos |
| Servicios: hardware, recursos, actualizaciones, recuperación, diagnóstico | Sincronización en la nube |
| Instalador, imagen ISO, repositorio firmado | Soporte de X11 como sesión |
| Xwayland para apps heredadas | |

## Principios (orden de prioridad ante conflicto)
1. **Seguridad desde el diseño** — sin root en la UI, mínimo privilegio, firmas.
2. **Estabilidad y recuperación** — todo componente puede caer y volver sin perder trabajo del usuario.
3. **Coherencia** — un sistema de diseño, un toolkit, un modelo de configuración.
4. **Eficiencia medida** — presupuestos por componente, verificados en CI.
5. **Rust nativo** en todo componente propio.
6. **Offline-first** — builds y funcionamiento sin Internet.
7. **Personalización profunda sin romper coherencia** — se personalizan tokens y comportamientos, no se inyecta código arbitrario en el shell.

## Métricas de éxito v1.0
- Memoria del sistema en reposo < 1 GB en la configuración de referencia (ver `benchmarks/metodologia.md`).
- Latencia input→foto p99 < 16 ms a 60 Hz en hardware de referencia.
- Cero procesos gráficos con UID 0.
- Reinicio del shell sin pérdida de ventanas < 1 s.
- 100 % de funcionalidades v1 con criterios de aceptación verificados.

## Anti-objetivos
- No es "Arch + rice". No es un tema. No es un WM con una barra.
- No priorizamos efectos sobre latencia.
- No reescribimos infraestructura madura por purismo de Rust.
