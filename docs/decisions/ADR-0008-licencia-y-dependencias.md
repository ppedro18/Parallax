# ADR-0008: Licencia del proyecto y política de dependencias

- **Estado:** Propuesto — decisión redactada, pendiente de aprobación del responsable del proyecto
- **Fecha:** 2026-11-19
- **Ámbito:** Código fuente, bibliotecas, componentes gráficos, herramientas, recursos y distribución de PARALLAX.
- **Documentación complementaria:** `security/dependencias.md`

## 1. Contexto

La licencia de PARALLAX condiciona su distribución, modificación, reutilización, comercialización y contribución por terceros.

También determina qué dependencias pueden incorporarse y bajo qué condiciones.

PARALLAX busca:

- Mantener un entorno de escritorio abierto.
- Permitir contribuciones externas.
- Conservar el control sobre la distribución del sistema.
- Reutilizar bibliotecas propias en otros proyectos.
- Evitar restricciones de licencias incompatibles.
- Mantener abierta la posibilidad de comercialización.
- Proteger la integridad y procedencia del código.

La política de licencias deberá cubrir tanto el código desarrollado por PARALLAX como las dependencias externas y los recursos distribuidos con el sistema.

## 2. Opciones evaluadas

### 2.1 GPL-3.0-or-later

Licencia copyleft fuerte.

**Ventajas:**
- Garantiza libertades de uso, modificación y redistribución.
- Mantiene las obligaciones de copyleft para las obras derivadas cubiertas por la licencia.
- Permite comercializar el software.
- Compatible con un modelo de desarrollo abierto.
- Facilita aprovechar dependencias distribuidas bajo GPL compatible.

**Consideraciones:**
- La distribución de obras cubiertas requiere cumplir las obligaciones de GPL.
- Limita la posibilidad de integrar determinadas dependencias incompatibles.
- Requiere definir claramente los límites entre componentes y bibliotecas.
- No permite tratar el código GPL como una biblioteca propietaria sin cumplir sus condiciones.

### 2.2 MPL-2.0

Licencia copyleft débil a nivel de archivo.

**Ventajas:**
- Permite combinar código MPL con componentes bajo otras licencias.
- Conserva obligaciones de publicación para archivos cubiertos que se modifican y distribuyen.
- Ofrece flexibilidad para arquitecturas con componentes independientes.

**Consideraciones:**
- Requiere controlar qué archivos están cubiertos por MPL.
- Puede complicar la gestión de modificaciones y redistribución.
- No proporciona el mismo nivel de copyleft que GPL.

### 2.3 MIT OR Apache-2.0

Licenciamiento permisivo para bibliotecas reutilizables.

**Ventajas:**
- Facilita reutilizar componentes en proyectos externos.
- Permite distribución comercial y propietaria por parte de consumidores.
- MIT es sencilla y ampliamente adoptada.
- Apache-2.0 incorpora disposiciones explícitas sobre patentes.

**Consideraciones:**
- No obliga a que los proyectos que reutilicen las bibliotecas publiquen sus modificaciones.
- Requiere conservar avisos de copyright y licencias.
- Apache-2.0 incorpora obligaciones adicionales de atribución y notificación.

La expresión `MIT OR Apache-2.0` significa que el usuario puede elegir cualquiera de las dos licencias, sujeto a sus respectivos términos.

## 3. Recomendación de licenciamiento

Se recomienda adoptar un modelo de licencias diferenciadas según la responsabilidad de cada componente.

### 3.1 Composición del proyecto

| Componente | Licencia recomendada |
|---|---|
| PARALLAX Compositor | GPL-3.0-or-later |
| PARALLAX Shell | GPL-3.0-or-later |
| Aplicaciones oficiales | GPL-3.0-or-later |
| Servicios propios del sistema | GPL-3.0-or-later |
| Bibliotecas reutilizables `px-*` | MIT OR Apache-2.0 |
| Herramientas independientes para desarrolladores | MIT OR Apache-2.0 |
| Documentación | CC BY-SA 4.0 |
| Recursos gráficos propios | Licencia documentada individualmente |

Esta separación permite mantener el entorno de escritorio bajo copyleft y ofrecer bibliotecas reutilizables bajo licencias permisivas.

La licencia de cada componente deberá declararse explícitamente.

La separación de crates no constituye por sí sola una garantía de independencia jurídica. Se deberá evaluar la relación real entre componentes, sus dependencias y la forma de distribución.

## 4. Política de licencias por componente

### 4.1 Componentes GPL

Los componentes principales del entorno utilizarán GPL-3.0-or-later.

Se aplicará a:

- Compositor.
- Shell.
- Aplicaciones oficiales.
- Servicios que formen parte del sistema.
- Componentes estrechamente integrados con el entorno.

Cada repositorio o paquete deberá incluir su archivo `LICENSE` y los avisos de copyright correspondientes.

### 4.2 Bibliotecas reutilizables

Los crates de bajo nivel podrán utilizar MIT OR Apache-2.0 cuando tengan una responsabilidad independiente y puedan reutilizarse fuera de PARALLAX.

Ejemplos:

- Tipos compartidos.
- Utilidades de geometría.
- Estructuras de datos.
- Serialización.
- Abstracciones independientes.
- Utilidades de configuración.
- Componentes matemáticos.
- Bibliotecas sin dependencia directa de la lógica del Shell.

La elección de una licencia permisiva deberá justificarse por la reutilización real del componente.

No se dividirá artificialmente código GPL en crates permisivos para intentar eludir las obligaciones de copyleft.

## 5. Política de dependencias externas

Todas las dependencias deberán pasar por una revisión de licencia antes de incorporarse al proyecto.

### 5.1 Dependencias permitidas

Se priorizarán:

- MIT.
- Apache-2.0.
- BSD-2-Clause.
- BSD-3-Clause.
- ISC.
- Zlib.
- GPL-3.0-or-later cuando sea compatible con el componente consumidor.
- LGPL cuando se hayan evaluado sus condiciones de integración y distribución.
- MPL-2.0 cuando su uso y obligaciones sean compatibles con la arquitectura y distribución del proyecto.

La compatibilidad no se determinará únicamente por el nombre de la licencia.

Se deberá analizar la combinación concreta de componentes.

### 5.2 Dependencias sujetas a revisión

Requerirán aprobación antes de incorporarse:

- AGPL.
- Licencias comerciales.
- Licencias duales.
- Licencias con restricciones de uso.
- Dependencias con cláusulas adicionales.
- Licencias personalizadas.
- Componentes sin licencia identificable.

### 5.3 Dependencias prohibidas

No se incorporarán:

- Bibliotecas sin licencia verificable.
- Componentes cuyo origen no pueda establecerse.
- Código con restricciones incompatibles con los objetivos del proyecto.
- Dependencias que impongan condiciones que PARALLAX no pueda cumplir.
- Recursos de terceros sin autorización de redistribución.

## 6. Herramientas de auditoría

Se establecerá una política automatizada de auditoría de dependencias.

Herramientas recomendadas:

- `cargo-deny`: auditoría de licencias, fuentes y dependencias.
- `cargo-audit`: detección de vulnerabilidades conocidas.
- `cargo-license`: inventario de licencias.
- SBOM en formato SPDX o CycloneDX para las distribuciones.

Las comprobaciones se integrarán en CI.

Una dependencia nueva no podrá incorporarse silenciosamente a una versión de producción.

Las excepciones deberán documentarse y aprobarse.

## 7. Registro de dependencias

Se mantendrá el archivo:

`security/dependencias.md`

Deberá incluir:

- Nombre de la dependencia.
- Versión utilizada.
- Licencia.
- URL del proyecto original.
- Componente de PARALLAX que la consume.
- Tipo de dependencia.
- Obligaciones relevantes.
- Estado de aprobación.
- Excepciones autorizadas.

También se generará un inventario automático de dependencias para cada versión publicada.

Las licencias de recursos gráficos, fuentes, iconos y otros archivos no Rust deberán registrarse por separado.

## 8. Distribución y comercialización

La elección de GPL-3.0-or-later no impedirá comercializar PARALLAX.

El proyecto podrá:

- Distribuir imágenes de instalación.
- Vender soporte técnico.
- Ofrecer servicios de mantenimiento.
- Comercializar dispositivos con PARALLAX preinstalado.
- Ofrecer servicios profesionales.
- Distribuir versiones modificadas cumpliendo las obligaciones aplicables.

Cuando se distribuya software cubierto por GPL, se deberán cumplir las condiciones de redistribución y proporcionar el código fuente correspondiente en los términos exigidos por la licencia.

El uso de software GPL como servicio remoto no genera por sí mismo la obligación de publicar el código fuente bajo GPL, a diferencia de las condiciones específicas de AGPL.

Las bibliotecas MIT OR Apache-2.0 podrán ser reutilizadas por terceros bajo sus respectivas condiciones.

## 9. Contribuciones externas

Antes de aceptar contribuciones se establecerá una política de procedencia y autoría.

Se recomienda:

- Mantener historial de contribuciones.
- Exigir que el contribuyente tenga derecho a aportar el código.
- Conservar avisos de copyright.
- Utilizar DCO o un mecanismo equivalente.
- No aceptar código copiado sin procedencia verificable.
- Documentar las contribuciones de terceros.
- Definir si se requiere CLA antes de habilitar contribuciones externas.

No se exigirá una cesión automática de copyright como condición implícita para contribuir.

Cualquier cambio de titularidad o concesión de derechos deberá ser explícito.

## 10. Recursos no ejecutables

La política también cubrirá:

- Fuentes tipográficas.
- Iconos.
- Ilustraciones.
- Wallpapers.
- Sonidos.
- Animaciones.
- Documentación.
- Capturas de pantalla.
- Recursos de instalación.

Cada recurso deberá tener una licencia compatible con su uso y redistribución.

Se priorizarán recursos originales o con licencias abiertas verificables.

Los recursos propios deberán incluir información de autoría y licencia.

## 11. Gestión de excepciones

Toda excepción a la política deberá documentarse.

El registro incluirá:

- Dependencia o recurso afectado.
- Licencia.
- Motivo de incorporación.
- Componente consumidor.
- Obligaciones.
- Riesgos identificados.
- Responsable de aprobación.
- Fecha de revisión.

Las excepciones no podrán aprobarse únicamente porque una dependencia resulte técnicamente conveniente.

## 12. Consecuencias

### Positivas

- Política de licencias explícita.
- Protección del carácter abierto del entorno.
- Reutilización independiente de bibliotecas propias.
- Posibilidad de comercialización.
- Mayor control sobre las dependencias.
- Mejor trazabilidad de recursos y código.
- Reducción del riesgo de incorporar componentes incompatibles.

### Negativas

- Mayor trabajo de auditoría.
- Necesidad de mantener inventarios.
- Restricciones en determinadas dependencias.
- Obligaciones de distribución bajo GPL.
- Necesidad de revisar licencias en cada actualización importante.

## 13. Revisión

Este ADR deberá revisarse si:

- Cambian los objetivos comerciales.
- Se incorpora una dependencia con licencia incompatible.
- Se decide publicar una parte del entorno bajo otra licencia.
- Se modifica la estrategia de contribuciones.
- Se introduce un modelo de distribución comercial diferente.
- Aparecen obligaciones legales nuevas que afecten al proyecto.

## 14. Decisión

PARALLAX adoptará una política de licencias por categoría de artefacto.

### 14.1 Componentes ejecutables del sistema

Licencia: `GPL-3.0-or-later`.

Se aplica a:

- `parallax-compositor`.
- `parallax-shell`.
- `parallax-lock`.
- Aplicaciones nativas oficiales.
- Servicios propios del sistema.
- Herramientas oficiales distribuidas como parte del producto.

Los componentes derivados o integrados deberán respetar las obligaciones de licencia correspondientes.

### 14.2 Bibliotecas reutilizables

Licencia: `MIT OR Apache-2.0`.

Se aplica a crates `px-*` que sean bibliotecas independientes, reutilizables y no estén sujetos a obligaciones incompatibles de componentes de terceros.

Cada crate declarará expresamente su licencia en `Cargo.toml` y conservará los avisos de copyright correspondientes.

La licencia permisiva no constituye una excepción automática para código derivado de componentes GPL ni permite ignorar obligaciones de dependencias.

### 14.3 Documentación

Licencia: `CC-BY-SA-4.0`.

Se aplica a la documentación original publicada por el proyecto, salvo que un documento indique expresamente otra licencia.

Los fragmentos de terceros conservarán sus atribuciones y licencias originales.

### 14.4 Recursos gráficos y multimedia

Cada recurso tendrá licencia individual documentada.

Se mantendrá un inventario de procedencia y licencia en:

`design-system/assets/manifest.toml`

No se incorporarán assets sin procedencia y autorización de uso verificables.

### 14.5 Dependencias externas

Cada dependencia deberá registrar:

- Nombre y versión.
- Licencia SPDX.
- Procedencia.
- Uso dentro del sistema.
- Tipo de integración.
- Compatibilidad con las licencias del componente consumidor.
- Obligaciones de redistribución.
- Riesgos de mantenimiento y seguridad.

No se presumirá compatibilidad por el simple hecho de que una dependencia sea de código abierto.

Las dependencias GPL, LGPL, AGPL, MPL, propietarias o con excepciones especiales requerirán revisión específica cuando corresponda.

### 14.6 Distribución

Los paquetes y repositorios oficiales deberán incluir los avisos de licencia, atribuciones y fuentes correspondientes a las obligaciones aplicables.

Se mantendrá un SBOM en formato CycloneDX.

### 14.7 Política de cambios

La modificación de la política de licencias requiere un nuevo ADR o una revisión formal de este ADR.

**Estado:** Propuesto, pendiente de aprobación del responsable del proyecto.

## 15. Principio rector

**PARALLAX debe proteger la libertad de su entorno de escritorio sin impedir que sus componentes independientes puedan reutilizarse en otros proyectos.**

La licencia no será una decisión improvisada por dependencia: será una política transversal de arquitectura, desarrollo y distribución.
> **Nota de Tarea 010 (propuesta pendiente, no normativa).** Dependencias candidatas surgidas de las revisiones 5–10, **pendientes de revisión formal de licencia y de entrada en `security/dependencias.md`**: crate `libc` (`MIT OR Apache-2.0` según docs.rs 0.2.191; uso: `posix_spawn`, `addclosefrom_np`, `close_range`, `flock`) o binding `extern "C"` propio a glibc (LGPL-2.1-or-later, enlace dinámico: compatibilidad pendiente de revisión); `libpolkit-agent-1` + GLib (D-16). Ninguna está aprobada. Ver [consolidación §4](../project-management/consolidacion-fase0.md).
