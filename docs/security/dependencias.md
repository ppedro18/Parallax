# Política de dependencias

## Reglas
1. Toda dependencia (crate, paquete del sistema, asset, fuente) se registra en la tabla con: nombre, versión exacta, licencia (SPDX), origen, justificación, revisor, fecha.
2. Licencias permitidas: MIT, Apache-2.0, BSD-2/3, ISC, Zlib, MPL-2.0, OFL-1.1 (fuentes), Unicode-3.0. LGPL solo para enlace dinámico de librerías de sistema. GPL solo para procesos separados. Verificado por `cargo deny`.
3. `Cargo.lock` versionado; `cargo vendor` en `vendor/`; builds `--offline --locked`.
4. `cargo vet`: cada crate auditado o importado de auditorías confiables (Mozilla, Google, Bytecode Alliance, ZcashFdn) vendidas en el repo.
5. Nueva dependencia = PR que actualiza esta tabla + ADR si es estructural (toolkit, runtime, render).
6. Preferir crates con: mantenimiento activo, `unsafe` mínimo o auditado, sin `build.rs` que descargue.
7. Paquetes del sistema: repositorio espejo controlado (snapshot fechado de Arch + repo `parallax`), firmado; nunca se instala desde AUR en builds.
8. Prohibido: descargas en `build.rs`, crates con telemetría, dependencias git sin commit fijado.
9. SBOM CycloneDX generado por release (`cargo cyclonedx` + manifiesto de paquetes).
10. Builds reproducibles: verificación bit a bit en dos máquinas por release.

## Registro (vacío hasta Fase 1)
| Nombre | Versión | Licencia | Origen | Justificación | Revisor | Fecha |
|---|---|---|---|---|---|---|
