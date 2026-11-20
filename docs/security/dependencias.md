# Política de dependencias

Aplica ADR-0008 §14 (estado: Propuesto). En caso de conflicto prevalece el ADR.

## Reglas
1. Toda dependencia (crate, paquete del sistema, fuente) se registra en la tabla con: nombre, versión exacta, licencia SPDX, procedencia, uso dentro del sistema, tipo de integración, compatibilidad con la licencia del componente consumidor, obligaciones de redistribución, riesgos de mantenimiento y seguridad, revisor y fecha. Los recursos gráficos y multimedia se registran en `design-system/assets/manifest.toml` (ADR-0008 §14.4).
2. Licencias de componentes propios: ejecutables del sistema `GPL-3.0-or-later`; bibliotecas `px-*` independientes `MIT OR Apache-2.0`; documentación `CC-BY-SA-4.0` (ADR-0008 §14.1–14.3).
3. No se presume compatibilidad por ser código abierto. Licencias permisivas (MIT, Apache-2.0, BSD-2/3-Clause, ISC, Zlib, Unicode-3.0, OFL-1.1 para fuentes) se aceptan tras comprobar la compatibilidad con el consumidor. GPL, LGPL, AGPL, MPL, propietarias o con excepciones especiales requieren revisión específica y quedan registradas con su análisis. Una crate `px-*` permisiva no puede depender de código cuyas obligaciones sean incompatibles con su licencia. Verificado por `cargo deny`.
4. `Cargo.lock` versionado; `cargo vendor` en `vendor/`; builds `--offline --locked`.
5. Nueva dependencia = PR que actualiza esta tabla + ADR si es estructural (toolkit, runtime, render).
6. Preferir crates con: mantenimiento activo, `unsafe` mínimo o auditado, sin `build.rs` que descargue.
7. Paquetes del sistema: repositorio espejo controlado (snapshot fechado de Arch + repo `parallax`), firmado; nunca se instala desde AUR en builds.
8. Prohibido: descargas en `build.rs`, crates con telemetría, dependencias git sin commit fijado, dependencias o recursos sin licencia o procedencia verificable.
9. Distribución: paquetes y repositorios oficiales incluyen avisos de licencia, atribuciones y fuentes exigidas; SBOM CycloneDX por release (`cargo cyclonedx` + manifiesto de paquetes).
10. Builds reproducibles: objetivo de verificación bit a bit en dos máquinas por release, cuando sea viable (roadmap §6.1).
11. Pendiente de decisión (no vinculante hasta su ADR): auditoría de crates con `cargo vet` e importación de auditorías de terceros. No figura en el toolchain de ADR-0001 §4.

## Registro (vacío hasta Fase 1)
| Nombre | Versión | Licencia SPDX | Procedencia | Uso | Integración | Compatibilidad | Obligaciones | Riesgos | Revisor | Fecha |
|---|---|---|---|---|---|---|---|---|---|---|
