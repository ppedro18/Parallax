# ADR-0002: Base de paquetes Arch con snapshot congelado

- **Estado:** Propuesto
- **Fecha:** 2026-11-19
- **Ámbito:** Distribución, paquetes, actualizaciones y mantenimiento del sistema operativo.

## 1. Contexto

PARALLAX utilizará Arch Linux como base tecnológica inicial, aprovechando su ecosistema de paquetes, kernel, Mesa, firmware y herramientas de usuario.

Sin embargo, PARALLAX no dependerá directamente del modelo rolling release de Arch Linux.

El objetivo es evolucionar progresivamente hacia una distribución independiente, controlada y mantenible, conservando inicialmente la compatibilidad con el ecosistema Arch.

Para conseguirlo, se necesita:

- Controlar exactamente las versiones instaladas.
- Evitar actualizaciones inesperadas.
- Garantizar que todos los componentes de una versión sean compatibles.
- Mantener un historial verificable de paquetes.
- Permitir instalaciones y actualizaciones offline.
- Controlar los cambios que llegan a producción.
- Incorporar parches críticos de seguridad fuera del calendario normal.

## 2. Opciones evaluadas

| Alternativa | Ventajas | Desventajas |
|---|---|---|
| Arch Rolling directo | Software actualizado y mantenimiento reducido de infraestructura propia | Actualizaciones impredecibles y menor control |
| Snapshot fechado de Arch | Control de versiones y reproducibilidad del entorno | Responsabilidad de mantener los snapshots |
| Debian / Fedora | Ecosistema maduro y ciclos de publicación definidos | Menor alineación con la estrategia tecnológica elegida |
| Construcción desde fuente (LFS-like) | Control completo de la distribución | Coste elevado de mantenimiento y desarrollo |

## 3. Decisión

PARALLAX utilizará un **snapshot fechado de Arch Linux**, obtenido de Arch Linux Archive (ALA), como base inicial de la distribución.

El snapshot será espejado en infraestructura controlada por PARALLAX y utilizado como referencia inmutable para construir cada versión del sistema.

No se utilizará Arch Rolling directamente en instalaciones de producción.

### 3.1 Inmutabilidad del snapshot

Cada snapshot deberá registrar:

- Fecha de referencia.
- Identificador único.
- Versiones exactas de los paquetes.
- Arquitectura de destino.
- Metadatos de repositorios.
- Checksums de los artefactos.
- Procedencia de cada paquete.
- Estado de validación.
- Referencia al snapshot anterior.

Una vez publicado, un snapshot no podrá modificarse.

Las correcciones deberán generar una nueva revisión identificable.

### 3.2 Infraestructura de repositorios

PARALLAX mantendrá repositorios independientes para sus paquetes y los paquetes heredados de Arch.

Estructura conceptual:

```text
distribution/
├── repositories/
│   ├── upstream/
│   │   ├── core/
│   │   ├── extra/
│   │   └── multilib/
│   │
│   ├── parallax/
│   │   ├── core/
│   │   ├── desktop/
│   │   ├── graphics/
│   │   ├── system/
│   │   └── applications/
│   │
│   └── snapshots/
│       └── YYYY-MM-DD/
│
├── packages/
│   ├── PKGBUILDs/
│   ├── patches/
│   └── metadata/
│
├── keys/
├── manifests/
├── channels/
└── release/
```

Los repositorios upstream deberán conservar la procedencia y trazabilidad de los paquetes originales.

Los paquetes desarrollados o modificados por PARALLAX deberán construirse y firmarse mediante la infraestructura de compilación propia.

### 3.3 Sistema de confianza y firmas

PARALLAX tendrá un llavero criptográfico propio para administrar la confianza de sus repositorios.

Se establecerán las siguientes reglas:

- Validar las firmas originales de los paquetes upstream.
- Verificar los checksums de todos los artefactos.
- Firmar los paquetes construidos por PARALLAX.
- Firmar las bases de datos de los repositorios propios.
- Mantener separación entre claves de desarrollo y producción.
- Proteger las claves privadas de publicación.
- Documentar procedimientos de rotación y revocación.
- Rechazar paquetes cuya procedencia o integridad no pueda verificarse.

La firma del repositorio por PARALLAX no debe confundirse con la firma original del desarrollador upstream.

Cuando se reconstruya o modifique un paquete, deberá generarse un nuevo artefacto con su correspondiente firma PARALLAX.

Las claves privadas de producción no deberán almacenarse en los repositorios de código.

### 3.4 Canales de distribución

PARALLAX utilizará tres canales oficiales.

| Canal | Frecuencia objetivo | Propósito |
|---|---|---|
| dev | Semanal | Integración y experimentación |
| beta | Mensual | Validación de estabilidad |
| stable | Trimestral | Distribución de versiones verificadas |

Las frecuencias representan objetivos de publicación, no fechas obligatorias.

Cada canal deberá disponer de un conjunto coherente de paquetes.

Los canales no podrán mezclar versiones arbitrarias de diferentes snapshots.

La promoción entre canales seguirá este flujo:

```text
Arch Linux Archive
        |
        v
Snapshot congelado
        |
        v
Repositorio upstream PARALLAX
        |
        v
Canal DEV
        |
        v
Pruebas y validación
        |
        v
Canal BETA
        |
        v
QA y pruebas de regresión
        |
        v
Canal STABLE
        |
        v
Distribución oficial
```

La promoción deberá conservar la trazabilidad de los artefactos y sus versiones.

### 3.5 Actualizaciones de seguridad

El calendario de publicación no limitará la incorporación de correcciones críticas.

PARALLAX mantendrá un procedimiento de actualización extraordinaria para:

- Vulnerabilidades críticas del kernel.
- Vulnerabilidades de escalada de privilegios.
- Vulnerabilidades del compositor.
- Problemas de seguridad en bibliotecas fundamentales.
- Vulnerabilidades que comprometan el aislamiento de procesos.
- Vulnerabilidades de ejecución remota relevantes.

Las actualizaciones extraordinarias deberán:

1. Identificar el paquete afectado.
2. Evaluar la vulnerabilidad.
3. Aplicar un parche upstream o backport cuando corresponda.
4. Construir el paquete.
5. Ejecutar pruebas de regresión.
6. Firmar y publicar el artefacto.
7. Distribuir la actualización a los canales afectados.

Las correcciones críticas podrán publicarse fuera del calendario habitual.

### 3.6 Paquetes propios

Todos los paquetes propios se mantendrán mediante PKGBUILD.

Cada paquete deberá incluir:

- Nombre y versión.
- Dependencias.
- Fuente de origen.
- Checksums.
- Parches aplicados.
- Opciones de compilación.
- Arquitecturas compatibles.
- Licencia.
- Información de mantenimiento.

Los paquetes propios deberán evitar dependencias innecesarias y configuraciones específicas del entorno de desarrollo.

## 4. Compatibilidad y ABI

Arch Linux no garantiza una ABI estable entre versiones de sus bibliotecas.

El uso de snapshots reduce la incertidumbre, pero no proporciona por sí mismo una ABI estable.

Por tanto:

- Todos los paquetes de una versión deberán validarse conjuntamente.
- Las actualizaciones de bibliotecas fundamentales requerirán reconstruir los componentes afectados.
- No se asumirán compatibilidades ABI entre snapshots diferentes.
- Los binarios propios deberán construirse contra el conjunto de bibliotecas de su versión objetivo.
- Las actualizaciones incompatibles deberán tratarse como transiciones controladas.

PARALLAX no garantizará compatibilidad binaria indefinida con paquetes de Arch Linux.

## 5. Instalación y operación offline

La infraestructura de distribución deberá permitir:

- Instalar el sistema sin conexión a Internet.
- Instalar paquetes desde un repositorio local.
- Conservar snapshots históricos.
- Recuperar versiones anteriores de paquetes.
- Reconstruir instalaciones utilizando manifiestos.
- Verificar la integridad de los paquetes instalados.

Los snapshots deberán conservarse según una política de retención documentada.

La capacidad offline no implica que todos los servicios o aplicaciones externas funcionen sin conexión.

## 6. Recuperación y rollback

PARALLAX deberá proporcionar mecanismos para recuperarse de actualizaciones defectuosas.

Se contemplarán:

- Recuperación de paquetes anteriores.
- Restauración de configuraciones.
- Recuperación de versiones del sistema.
- Verificación de integridad.
- Arranque de una versión anterior cuando la infraestructura lo permita.

El mecanismo de rollback deberá estar integrado con el sistema de archivos y el gestor de arranque elegidos.

No se asumirá que reinstalar paquetes anteriores garantiza por sí solo una recuperación completa.

## 7. Consecuencias

### Positivas

- Control completo sobre las versiones distribuidas.
- Mayor previsibilidad del sistema.
- Reducción de actualizaciones inesperadas.
- Repositorios independientes.
- Posibilidad de instalaciones offline.
- Trazabilidad de paquetes.
- Publicaciones controladas.
- Base para evolucionar hacia una distribución independiente.

### Negativas

- Responsabilidad de mantener infraestructura de repositorios.
- Necesidad de gestionar firmas y claves criptográficas.
- Trabajo adicional de backports de seguridad.
- Necesidad de reconstruir paquetes ante cambios incompatibles.
- Costes de almacenamiento y mantenimiento de snapshots.
- Responsabilidad directa sobre las actualizaciones de producción.

## 8. Evolución hacia una distribución independiente

La utilización de Arch Linux será una decisión de infraestructura inicial, no una dependencia permanente.

PARALLAX podrá evolucionar hacia repositorios, herramientas de construcción y procesos de distribución propios.

Esta transición deberá ser gradual y mantener la compatibilidad operativa de las instalaciones existentes.

La independencia de Arch Linux no será un objetivo que justifique sacrificar estabilidad o seguridad.

## 9. Criterios de aceptación

La implementación se considerará válida cuando:

1. No existan actualizaciones rolling directas en instalaciones stable.
2. Cada versión tenga un snapshot identificable e inmutable.
3. Los paquetes puedan verificarse criptográficamente.
4. Los repositorios propios estén firmados.
5. Exista trazabilidad de procedencia.
6. Los canales dev, beta y stable estén separados.
7. Se puedan publicar parches críticos fuera de ciclo.
8. Exista un procedimiento probado de rollback.
9. Se pueda realizar una instalación completamente offline.
10. Sea posible reconstruir una instalación a partir de sus manifiestos y artefactos conservados.

## 10. Revisión

Este ADR deberá revisarse cuando:

- El mantenimiento de snapshots supere la capacidad operativa del equipo.
- El coste de los backports de seguridad sea insostenible.
- Aparezcan problemas estructurales de compatibilidad.
- PARALLAX disponga de infraestructura suficiente para mantener una base de paquetes independiente.
- Se decida cambiar la estrategia de distribución.

## 11. Principio rector

**PARALLAX controla qué software entra en el sistema, cuándo entra, cómo se verifica y cómo se recupera.**

La distribución no debe depender de actualizaciones impredecibles ni delegar completamente su estabilidad en terceros.

Arch Linux será el punto de partida tecnológico. La infraestructura de paquetes, confianza y publicación será responsabilidad de PARALLAX.