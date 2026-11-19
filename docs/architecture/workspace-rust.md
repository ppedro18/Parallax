# Workspace Rust — PARALLAX

## Objetivo

Definir un workspace Rust unificado, reproducible, seguro y optimizado para PARALLAX.

La arquitectura debe favorecer:

- Rust estable y Edition 2024.
- Dependencias controladas y auditables.
- Compilaciones reproducibles.
- Desarrollo y compilación offline.
- Bajo consumo de memoria y CPU.
- Separación estricta entre componentes y procesos.
- Mantenimiento a largo plazo.
- Compatibilidad con Arch Linux y su ecosistema.

**No introducir dependencias, abstracciones o herramientas por conveniencia. Cada decisión debe tener una justificación técnica.**

## 1. Workspace Cargo

- Un único workspace Cargo para los componentes propios de PARALLAX.
- Centralizar las versiones y características de dependencias compartidas mediante `[workspace.dependencies]`.
- Utilizar `resolver = "3"`.
- Utilizar `edition = "2024"`.
- Fijar una versión exacta y estable del toolchain mediante `rust-toolchain.toml`.
- Verificar que la versión elegida esté publicada y sea compatible con todas las dependencias seleccionadas.
- Fijar los componentes necesarios: `rustfmt`, `clippy` y los componentes adicionales que realmente se utilicen.
- Mantener un `Cargo.lock` versionado para las aplicaciones y binarios del proyecto.
- Evitar versiones flotantes y actualizaciones automáticas no revisadas.

Antes de establecer el toolchain, verificar la compatibilidad de Rust, Cargo, Edition 2024 y las dependencias críticas.

## 2. Compilación offline y vendor

El proyecto debe poder compilarse sin conexión a Internet una vez preparado su entorno de dependencias.

Configurar `.cargo/config.toml`:

```toml
[source.crates-io]
replace-with = "vendored-sources"

[source.vendored-sources]
directory = "vendor"

[net]
offline = true
```

Requisitos:

- Generar el directorio `vendor/` mediante `cargo vendor`.
- Versionar el `Cargo.lock`.
- Documentar el procedimiento de actualización de dependencias.
- Verificar que las fuentes vendorizadas coincidan con el lockfile.
- Evitar dependencias Git sin revisión y fijación de commit.
- No descargar dependencias durante una compilación offline.
- Validar con `cargo build --workspace --offline --locked`.
- Documentar las dependencias nativas del sistema necesarias para compilar.

La compilación offline no debe confundirse con una auditoría de seguridad actualizada.

Las bases de vulnerabilidades y auditorías deben actualizarse mediante un proceso independiente, revisado y reproducible.

## 3. Perfiles de compilación

Perfil general:

```toml
[profile.release]
lto = "thin"
codegen-units = 1
debug = "line-tables-only"
```

Definir perfiles adicionales únicamente cuando exista una necesidad técnica demostrada.

Ejemplo:

```toml
[profile.release-services]
inherits = "release"
panic = "abort"

[profile.release-compositor]
inherits = "release"
panic = "unwind"
```

Estos perfiles deben seleccionarse explícitamente durante la compilación de los artefactos correspondientes.

### Política de pánicos

No utilizar `catch_unwind` como mecanismo general de aislamiento de clientes o recuperación del compositor.

Separar:

1. Errores de protocolo: validar entradas, rechazar solicitudes inválidas y aislar al cliente afectado.
2. Errores operativos recuperables: gestionar mediante `Result` y estados explícitos.
3. Pánicos internos: registrar el fallo y aplicar la estrategia de recuperación del proceso.
4. Fallos nativos, abortos y comportamiento indefinido: no asumir que pueden recuperarse mediante `catch_unwind`.

No continuar ejecutando el compositor después de un pánico interno si no puede demostrarse que su estado sigue siendo consistente.

La recuperación de la sesión debe estar diseñada fuera del mecanismo de desenrollado de pánicos y coordinada con el gestor de sesión.

## 4. Depuración y símbolos

- Mantener símbolos de depuración adecuados para diagnosticar fallos.
- Separar los símbolos de depuración de los paquetes de producción cuando corresponda.
- Documentar el procedimiento para asociar símbolos con sus binarios exactos.
- Evitar eliminar información necesaria para generar backtraces y analizar volcados.
- Evaluar `split-debuginfo` según el toolchain, linker y formato de empaquetado utilizado.

Los paquetes de producción no deben incluir innecesariamente todos los símbolos de depuración.

## 5. Asignador de memoria

- Utilizar inicialmente el asignador del sistema.
- No incorporar `mimalloc` de manera global sin evidencia.
- Evaluar asignadores alternativos mediante benchmarks reproducibles.
- Comparar PSS, RSS, memoria máxima, fragmentación y latencia de asignación.
- Evaluar resultados por componente: compositor, Shell y servicios.
- Priorizar el consumo real de memoria del sistema sobre microbenchmarks aislados.

Cualquier cambio de asignador debe justificarse mediante resultados medibles.

## 6. CI y validaciones

La integración continua debe ejecutar, como mínimo:

```bash
cargo fmt --all -- --check

cargo clippy \
  --workspace \
  --all-targets \
  -- -D warnings

cargo test --workspace --locked

cargo deny check

cargo vet

cargo audit

cargo build --workspace --offline --locked
```

### Política de CI

Separar dos flujos:

**A. CI reproducible offline**

- Toolchain fijado.
- Dependencias vendorizadas.
- Lockfile congelado.
- Compilación sin red.
- Pruebas y análisis estático.
- Políticas de licencia y dependencias.
- Artefactos reproducibles.

**B. Mantenimiento de seguridad**

- Actualización de la base RustSec.
- Actualización de auditorías de dependencias.
- Revisión de vulnerabilidades.
- Evaluación de dependencias abandonadas.
- Revisión de cambios de toolchain.
- Actualización controlada del vendor y lockfile.

Una auditoría offline con información desactualizada no puede considerarse evidencia suficiente de seguridad.

## 7. Reproducibilidad

Utilizar `SOURCE_DATE_EPOCH` y controlar los factores que afectan a la generación de artefactos:

- Versión exacta de Rust y Cargo.
- Linker y herramientas nativas.
- Dependencias y lockfile.
- Variables de entorno.
- Rutas de compilación.
- Timestamps y metadatos.
- Generación de código.
- Recursos incorporados.
- Opciones de compilación y empaquetado.

La reproducibilidad debe comprobarse comparando artefactos generados en entornos independientes.

No considerar suficiente la mera presencia de `SOURCE_DATE_EPOCH`.

## 8. Dependencias candidatas

Las siguientes bibliotecas son candidatas sujetas a evaluación. Su inclusión no está aprobada automáticamente.

### Compositor y Wayland

- `smithay`
- `calloop`
- `drm`
- `gbm`
- `wayland-server`
- `wayland-client`
- `wayland-protocols`

### Entrada y dispositivos

- `input`
- `udev`

### IPC y sistema

- `zbus`
- `rustix`
- `nix`, únicamente cuando `rustix` no cubra adecuadamente la operación requerida.

### Asincronía

- `tokio`, únicamente en procesos que necesiten su modelo de ejecución.

No introducir Tokio globalmente ni en el compositor por defecto.

### Configuración y serialización

- `serde`
- `toml`

### Observabilidad y errores

- `tracing`
- `tracing-journald`
- `thiserror`
- `anyhow`

### Texto y accesibilidad

- `cosmic-text`
- `accesskit`
- Toolkit seleccionado mediante ADR-0005.

### Terminal

- `alacritty_terminal` o `vte`, sujeto a ADR-0011 y a la comparación de responsabilidades, API y mantenimiento.

### Sistema de archivos y multimedia

- `notify`
- `pipewire`
- `image`
- `resvg`

## 9. Política obligatoria de dependencias

Antes de incorporar una dependencia, documentar:

1. Funcionalidad que aporta.
2. Componente que la utilizará.
3. Alternativas consideradas.
4. Madurez y actividad de mantenimiento.
5. Licencia y compatibilidad con ADR-0008.
6. Dependencias transitivas.
7. Vulnerabilidades conocidas.
8. Impacto en memoria y rendimiento.
9. Dependencias nativas requeridas.
10. Compatibilidad con compilación offline.

No añadir bibliotecas redundantes.

No añadir dependencias a todo el workspace cuando solo las necesita un componente.

No sustituir una tecnología madura del ecosistema Linux por una biblioteca Rust propia sin una justificación arquitectónica.

Mantener `security/dependencias.md` actualizado con las decisiones, versiones y justificaciones.

## 10. Restricciones para los agentes

- No actualizar el toolchain sin revisión.
- No añadir dependencias sin justificar y documentar su incorporación.
- No introducir `unsafe` sin encapsulación, invariantes y justificación.
- No introducir Tokio en el compositor por conveniencia.
- No modificar las estrategias de pánicos sin evaluar sus consecuencias.
- No desactivar controles de CI para conseguir una compilación exitosa.
- No introducir dependencias Git sin fijar y revisar su procedencia.
- No declarar un build reproducible sin evidencia comparativa.
- No declarar una dependencia segura únicamente porque `cargo audit` no informa vulnerabilidades.

## 11. Criterios de aceptación

El workspace se considera preparado cuando:

- Compila con el toolchain fijado.
- Compila sin conexión utilizando el vendor.
- Supera fmt, Clippy y tests.
- Supera las políticas de dependencias y licencias.
- Tiene un proceso documentado de auditoría de seguridad.
- Tiene perfiles de compilación justificados.
- Permite generar artefactos reproducibles en entornos independientes.
- Mantiene una separación clara de dependencias por componente.
- No introduce dependencias sin aprobación arquitectónica.

**Estado: Propuesto. Pendiente de revisión y aprobación de arquitectura PARALLAX.**