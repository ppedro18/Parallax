# Sistema de diseño PARALLAX

Fuente única de verdad visual y de interacción para compositor, shell y aplicaciones nativas. Sustituye a `docs/design/sistema-diseno.md`.

Referencia de lenguaje: Caffyne Shell (oscuro, paneles flotantes, translucidez medida, esquinas redondeadas). PARALLAX toma el *lenguaje*, nunca assets, iconos ni layouts literales.

| Documento | Contenido | Consumidor principal |
|---|---|---|
| [DESIGN_PRINCIPLES.md](DESIGN_PRINCIPLES.md) | Principios y reglas de decisión | todos |
| [DESIGN_TOKENS.md](DESIGN_TOKENS.md) | Tokens: color, espaciado, forma, opacidad | `px-ui`, compositor |
| [DEPTH_SYSTEM.md](DEPTH_SYSTEM.md) | Capas, elevación, sombras, vidrio, parallax | compositor, shell |
| [TYPOGRAPHY.md](TYPOGRAPHY.md) | Familias, escala, render de texto | `px-ui` |
| [ICONOGRAPHY.md](ICONOGRAPHY.md) | Rejilla, trazo, iconos de app, fallback | diseño, apps |
| [COMPONENTS.md](COMPONENTS.md) | Catálogo de componentes con estados | `px-ui`, apps |
| [MOTION.md](MOTION.md) | Duraciones, curvas, coreografía | compositor, `px-ui` |
| [ACCESSIBILITY.md](ACCESSIBILITY.md) | Contraste, teclado, lector, movimiento | todos |
| [THEMING.md](THEMING.md) | Temas, personalización, distribución de tokens | settingsd, `px-ui` |
| [WINDOW_STATES.md](WINDOW_STATES.md) | Apariencia de ventanas por estado | compositor |
| [RENDERING_PERFORMANCE.md](RENDERING_PERFORMANCE.md) | Coste y presupuesto de cada efecto | rendimiento, compositor |
| [ACCEPTANCE_CRITERIA.md](ACCEPTANCE_CRITERIA.md) | Cómo se verifica todo lo anterior | QA |

## Formato de los tokens
Los tokens se definen en `design-system/tokens/*.toml` (Fase 1) y se generan a código Rust (`px-ui::tokens`) en build. Este documento es la especificación; el TOML es la implementación. Un token no documentado aquí no existe.

## Propiedad
Dueño: `disenador-producto`. Revisan: `ingeniero-framework-ui` (viabilidad), `ingeniero-rendimiento` (coste, con veto), `ingeniero-compositor` (efectos de ventana).
