# Propuesta: estructura del workspace y nomenclatura

- **Estado:** Propuesta — pendiente de aprobación de arquitecto-sistemas y del propietario. **No normativa.**
- **Fecha:** 2026-11-19
- **Resuelve (si se aprueba):** P-09, P-10, P-11 (base-audit-r3).
- **No crea** `Cargo.toml`, crates ni directorios. Los documentos afectados no se modifican hasta su aprobación; §5 lista los cambios a aplicar.

## 1. Inventario actual

### 1.1 Estructura del repositorio: tres versiones

| Fuente | Estructura |
|---|---|
| `architecture/arquitectura.md §4` | Agrupada por capa: `crates/` (px-*), `core/{compositor,wm,proto}`, `intelligence/resourced`, `services/*`, `shell/{shell,widgets,lock,greeter}`, `applications/*`, `distribution/*`, `foundation/`, `vendor/`, `tests/{integration,vm,recovery,perf}`, `docs/` |
| `roadmap/roadmap.md §1.2` | `crates/` (4 crates), `apps/`, `services/`, `distribution/`, `tests/`, `tools/` y documentación en la raíz (`specifications/`, `architecture/`, `security/`, `benchmarks/`) |
| `specifications/compositor.md §15` | `crates/parallax-compositor/`, `crates/px-wm/`, `px-wm-layouts`, `px-compositor-adapter`, `px-render`, `px-input`, `px-protocols`, `px-types` (todo bajo `crates/`) |

### 1.2 Prefijos: crates frente a binarios
AGENTS.md: crates internos `px-`; binarios `parallax-<nombre>`.

| Nombre en documentos | Dónde | Tipo real | Conflicto |
|---|---|---|---|
| `px-update` | roadmap §6.4, ADR-0009 §5 | Binario/servicio | Debería ser `parallax-updated` (como `arquitectura.md §2`) |
| `px-terminal`, `px-files`, `px-settings`, `px-task-manager` | roadmap Fase 4, ADR-0011 | Aplicaciones (binarios) | Debería ser `parallax-<app>`; `arquitectura.md` y `servicios.md` ya usan `parallax-terminal`, `parallax-settings`, `parallax-task-manager` |
| `px-compositor` | roadmap Fase 2 | Binario | `parallax-compositor` en el resto |
| `px-shell` | ADR-0006 §4.2, ADR-0010 | Binario/crate del Shell | `parallax-shell` en el resto |
| `px-proto` / `px-protocols` | `arquitectura.md §3` / `compositor.md §15` | Crate de bindings de protocolos privados | Mismo concepto con dos nombres |
| `px-dbus` | `arquitectura.md §3` | Crate de interfaces zbus generadas | Sin conflicto; depende de contratos XML (P-06) |
| `px-render`, `px-input`, `px-wm-layouts`, `px-compositor-adapter` | `compositor.md §15` | Crates candidatos | El propio documento dice no dividir sin necesidad real; no figuran en `arquitectura.md §3` |
| `px-theme` | design-system `THEMING.md`, `COMPONENTS.md` | Módulo de tema | No figura en `arquitectura.md §3`; posible módulo de `px-ui` |

Nombres que **no** son conflictos: `parallax-shell-v1`, `parallax-workspace-v1` (protocolos Wayland); `parallax-hw`, `parallax-diag`, `parallax-service` (usuarios de sistema); `parallax-session.target` (unidad); acciones polkit `org.parallax.<area>.<accion>` en minúsculas.

### 1.3 Nombres D-Bus
AGENTS.md: `org.parallax.<Servicio>1`.

| Forma | Dónde |
|---|---|
| Versionada: `Session1`, `Settings1`, `Hardware1`, `Resources1`, `Notifications1`, `Job1` | arquitectura, ipc, servicios, especificaciones |
| Sin versión: `Shell`, `Compositor`, `Settings`, `Power`, `Widgets`, `Notifications` | solo `ADR-0007 §5.3` (ejemplos "conceptuales") |
| Errores `org.parallax.Error.*` | `ipc.md` |

## 2. Propuesta de estructura

Se toma `arquitectura.md §4` como base, porque es la más completa, coincide con la tabla de propiedad de AGENTS.md (`core/`, `shell/`, `intelligence/`, `services/hardware`, `applications/`) y separa capas con distintos privilegios y ciclos de release.

```text
/
├── Cargo.toml  Cargo.lock  rust-toolchain.toml  deny.toml      # Fase 1
├── .cargo/config.toml                                          # Fase 1 (vendor)
├── AGENTS.md  README.md
├── docs/                                                       # documentación (no se mueve a la raíz)
├── crates/            # bibliotecas compartidas px-* sin dueño de capa
├── core/              # C4: compositor/, wm/ (px-wm), proto/ (px-proto)
├── intelligence/      # C3: resourced/
├── services/          # C2: hardwared/, settingsd/, notifyd/, updated/, recoveryd/, diagd/
├── shell/             # C5: shell/, widgets/, lock/, greeter/
├── applications/      # C6: files/, terminal/, settings/, task-manager/, observatory/, capture/
├── foundation/        # C1: unidades systemd, udev, polkit, PAM, sysctl (no Rust)
├── distribution/      # C7: packages/, image/, installer/, repo/, hardware-profiles/
├── tests/             # integration/, vm/, recovery/, perf/
├── tools/             # herramientas de desarrollo (xtask u otros), si aparecen
└── vendor/            # cargo vendor
```

Reglas propuestas:
1. **Directorios creados solo cuando haya un primer crate**; la tabla es un mapa, no un andamiaje (roadmap §1.2: "no se crearán abstracciones sin consumidores").
2. `docs/` permanece en `docs/`; se descarta la variante del roadmap con documentación en la raíz.
3. `apps/` → `applications/` (coincide con AGENTS.md).
4. `tools/` se conserva del roadmap, solo si aparece una herramienta.
5. Fase 1 solo crea `crates/{px-types,px-ipc,px-config,px-telemetry}` (roadmap §1.3).
6. Los crates de `compositor.md §15` (`px-render`, `px-input`, `px-wm-layouts`, `px-compositor-adapter`) son **módulos** de `parallax-compositor`/`px-wm` hasta que se justifique dividirlos (regla que ya dice el propio documento).
7. Nombre del directorio y nombre del crate: el directorio es corto (`core/compositor/`), el crate lleva el nombre completo (`parallax-compositor`).

## 3. Propuesta de nomenclatura

| Elemento | Regla | Ejemplos |
|---|---|---|
| Crate de biblioteca | `px-<nombre>` | `px-types`, `px-ipc`, `px-wm`, `px-ui`, `px-proto`, `px-dbus` |
| Crate binario / ejecutable | `parallax-<nombre>` | `parallax-compositor`, `parallax-shell`, `parallax-lock`, `parallax-greeter` |
| Demonio de servicio | `parallax-<nombre>d` | `parallax-settingsd`, `parallax-updated`, `parallax-recoveryd`, `parallax-resourced` |
| Aplicación | `parallax-<app>` | `parallax-files`, `parallax-terminal`, `parallax-settings`, `parallax-task-manager` |
| Unidad systemd | `parallax-<nombre>.service` / `.target` | `parallax-session.target` |
| Usuario de sistema | `parallax-<rol>` | `parallax-hw`, `parallax-diag` |
| Nombre D-Bus de servicio | `org.parallax.<Servicio><N>` | `org.parallax.Settings1` |
| Interfaz D-Bus | igual que el nombre o subinterfaz `org.parallax.<Servicio><N>.<Sub>` | — |
| Ruta de objeto | `/org/parallax/<Servicio><N>` | `/org/parallax/Settings1` |
| Error D-Bus | `org.parallax.Error.<Nombre>` | ya en `ipc.md` |
| Acción polkit | `org.parallax.<area>.<accion>` (minúsculas) | `org.parallax.update.apply` |
| Protocolo Wayland privado | `parallax-<nombre>-v<N>` | `parallax-shell-v1` |
| ID de aplicación | `org.parallax.<App>` | **Pendiente**: no definido en los documentos |

Decisiones abiertas dentro de esta propuesta:
- `px-proto` (recomendado, más corto, ya en arquitectura) frente a `px-protocols`.
- `px-theme`: módulo de `px-ui` (recomendado hasta ADR-0005) o crate aparte.
- Lista definitiva de servicios D-Bus: depende de los contratos XML (P-06); esta propuesta **no** fija interfaces.

## 4. Dependencias
- ADR-0004 (`px-wm` puro), ADR-0005 (`px-ui`, `px-theme`), ADR-0006 (Shell), ADR-0007 (D-Bus), ADR-0008 §14.2 (licencia de crates `px-*`): la regla `px-` = biblioteca coincide con la asignación de licencia MIT OR Apache-2.0; un binario `parallax-*` es GPL-3.0-or-later. Esto hace que el prefijo sea también señal de licencia, a confirmar al aprobar ADR-0008.

## 5. Cambios a aplicar tras aprobación (no aplicados)

| Archivo | Cambio |
|---|---|
| `roadmap/roadmap.md §1.2` | Sustituir el árbol por el de §2 (o remitir a `arquitectura.md §4`) |
| `roadmap/roadmap.md` Fases 2, 4, 6 | `px-compositor`→`parallax-compositor`; `px-terminal/files/settings/task-manager`→`parallax-*`; `px-update`→`parallax-updated` |
| `decisions/ADR-0009 §5` | `px-update`→`parallax-updated` (requiere revisión del ADR; está `Propuesto`) |
| `decisions/ADR-0011` | `px-terminal`→`parallax-terminal` |
| `decisions/ADR-0006 §4.2`, `ADR-0010` | `px-shell`→`parallax-shell` |
| `decisions/ADR-0007 §5.3` | Ejemplos con sufijo de versión |
| `specifications/compositor.md §15` | Árbol bajo `core/`; crates candidatos como módulos; `px-protocols`→`px-proto` |
| `architecture/arquitectura.md §4` | Añadir `tools/` y la regla 1 |
