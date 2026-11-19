# ADR-0001: Rust como lenguaje de todos los componentes propios

- **Estado:** Propuesto
- **Fecha:** 2026-11-19
- **Decisores:** PARALLAX Architecture Team
- **Ámbito:** Todos los componentes desarrollados y mantenidos por el proyecto PARALLAX.

## 1. Contexto

PARALLAX establece Rust como lenguaje principal y obligatorio para el desarrollo de sus componentes propios.

El entorno tendrá procesos de larga duración y alta responsabilidad operativa, incluyendo:

- Compositor gráfico.
- Gestor de ventanas.
- Shell y entorno de escritorio.
- Servicios del sistema.
- Gestión de sesiones.
- Sistema de configuración.
- Gestión de dispositivos.
- Servicios de integración.
- Herramientas de diagnóstico y mantenimiento.

Estos componentes deben priorizar la seguridad de memoria, estabilidad, rendimiento predecible y mantenibilidad.

Rust permite reducir significativamente las vulnerabilidades relacionadas con accesos inválidos a memoria y condiciones de concurrencia, sin renunciar al rendimiento nativo.

## 2. Decisión

Todo componente propio de PARALLAX deberá implementarse en **Rust estable (Stable Rust)**.

Se utilizarán las versiones estables del compilador y las herramientas oficiales del ecosistema.

No se permitirá introducir nuevos componentes propios en C, C++, Python, JavaScript, TypeScript u otros lenguajes sin una excepción arquitectónica aprobada.

### 2.1 Excepciones permitidas

Se permiten las siguientes excepciones:

| Categoría | Política |
|---|---|
| Kernel Linux | Consumir el kernel existente |
| Drivers | Utilizar drivers existentes |
| Firmware | Utilizar firmware del fabricante |
| Mesa | Consumir infraestructura gráfica existente |
| systemd | Integración mediante interfaces disponibles |
| PipeWire | Integración mediante interfaces disponibles |
| NetworkManager | Integración mediante interfaces disponibles |
| BlueZ | Integración mediante interfaces disponibles |
| libinput | FFI cuando sea necesario |
| libseat | FFI cuando sea necesario |
| PDFium / MuPDF | Evaluación individual según necesidad |

Estas excepciones no autorizan a desarrollar nuevos componentes propios en dichos lenguajes.

### 2.2 Interoperabilidad FFI

Cuando PARALLAX necesite interoperar con bibliotecas externas escritas en C o C++, deberá priorizar:

1. APIs oficiales y mantenidas.
2. Bindings Rust existentes y mantenidos.
3. Bindings generados mediante herramientas oficiales.
4. FFI propio cuando no exista una alternativa madura.

Toda integración FFI deberá:

- Mantener una frontera claramente definida.
- Encapsular código inseguro (`unsafe`) en módulos específicos.
- Validar punteros, referencias y ciclos de vida.
- Gestionar correctamente la propiedad de memoria.
- Evitar exponer interfaces inseguras al resto del sistema.
- Documentar las garantías de seguridad.
- Incluir pruebas de integración.
- Registrar la dependencia y su licencia.

Toda nueva dependencia FFI requiere un ADR específico.

## 3. Principios de implementación

### 3.1 Rust seguro por defecto

- Priorizar Rust Safe.
- Utilizar `unsafe` únicamente cuando sea técnicamente necesario.
- Documentar cada bloque `unsafe` y sus invariantes de seguridad.
- Evitar abstracciones innecesariamente complejas.
- Aplicar ownership y borrowing correctamente.
- Evitar ciclos de referencias y fugas de memoria.

### 3.2 Arquitectura modular

Cada componente debe tener responsabilidades delimitadas.

Se favorecerá:

- Separación entre lógica de negocio e infraestructura.
- Interfaces explícitas entre servicios.
- Comunicación mediante IPC cuando corresponda.
- Gestión estructurada de errores.
- Configuración tipada.
- Logging y observabilidad centralizados.
- Pruebas unitarias, integración y rendimiento.

### 3.3 Concurrencia

Se priorizarán las primitivas de concurrencia seguras de Rust.

Las tareas asíncronas deberán tener ciclos de vida definidos, cancelación controlada y límites de recursos cuando corresponda.

### 3.4 Dependencias

Toda dependencia deberá evaluarse según:

- Mantenimiento activo.
- Seguridad.
- Licencia.
- Compatibilidad.
- Rendimiento.
- Dependencias transitivas.
- Tamaño del binario.
- Impacto en compilación.

Se evitarán dependencias innecesarias y duplicaciones funcionales.

## 4. Toolchain oficial

El proyecto utilizará:

- Rust Stable.
- Cargo como gestor de paquetes y compilación.
- rustfmt para formato.
- Clippy para análisis estático.
- rust-analyzer para desarrollo.
- cargo-deny para auditoría de dependencias y licencias.
- cargo-audit para vulnerabilidades conocidas.
- Miri cuando sea aplicable a código inseguro.
- Criterion para benchmarks.

Las versiones del toolchain deberán quedar fijadas y documentadas mediante `rust-toolchain.toml`.

## 5. Consecuencias

### Positivas

- Reducción de vulnerabilidades de seguridad de memoria.
- Mayor uniformidad tecnológica.
- Mejor control de concurrencia.
- Menor dependencia de runtimes externos.
- Mayor facilidad para establecer estándares de calidad.
- Posibilidad de compartir bibliotecas internas.
- Mejor integración entre servicios y componentes.

### Negativas

- Tiempos de compilación potencialmente elevados.
- Ecosistema gráfico menos maduro que C/C++.
- Necesidad de desarrollar abstracciones propias en determinadas áreas.
- Curva de aprendizaje y mantenimiento.
- Posibles limitaciones de bibliotecas especializadas.

## 6. Restricciones

La decisión de utilizar Rust no implica:

- Reescribir el kernel Linux.
- Reimplementar drivers existentes.
- Reemplazar systemd sin una justificación técnica.
- Reescribir Mesa.
- Crear alternativas propias a servicios que ya cumplen los requisitos.
- Introducir FFI indiscriminadamente.
- Priorizar pureza tecnológica sobre estabilidad y funcionalidad.

La interoperabilidad con el ecosistema Linux es una característica arquitectónica, no una excepción indeseada.

## 7. Validación

Antes de adoptar una tecnología crítica que no tenga una alternativa Rust suficientemente madura, deberá realizarse un spike técnico.

El spike evaluará:

- Viabilidad funcional.
- Rendimiento.
- Consumo de memoria.
- Estabilidad.
- Madurez del ecosistema.
- Integración con Linux y Wayland.
- Mantenibilidad a largo plazo.

Si no existe una alternativa viable, se deberá presentar un ADR que documente la excepción.

## 8. Revisión

Este ADR podrá revisarse cuando:

- Un componente crítico carezca de una alternativa Rust viable.
- Una dependencia deje de mantenerse.
- Aparezca una vulnerabilidad arquitectónica relevante.
- Los requisitos de rendimiento no puedan cumplirse.
- Cambien sustancialmente las necesidades del proyecto.

## 9. Principio rector

**PARALLAX será desarrollado en Rust de forma nativa, priorizando seguridad de memoria, control de recursos, rendimiento y estabilidad.**

La tecnología debe servir a la arquitectura del sistema. No se reescribirán componentes maduros del ecosistema Linux únicamente para cumplir una preferencia tecnológica.