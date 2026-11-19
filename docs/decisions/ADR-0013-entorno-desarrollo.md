# ADR-0013: Entorno de desarrollo aislado del anfitrión
- Estado: Propuesto · Fecha: 2026-11-19
## Contexto
El anfitrión inspeccionado es **macOS 27 ARM64** (Apple Silicon) con Rust 1.99.1 de Homebrew, no Arch Linux. No se puede ejecutar ni probar un compositor DRM/KMS ni servicios systemd en macOS. Además PARALLAX v1 es x86_64.
## Opciones
VM Linux aarch64 en el Mac (UTM/QEMU con virtio-gpu) para desarrollo + hardware x86_64 dedicado para pruebas reales · VM x86_64 emulada (lenta) · Desarrollo directo en máquina Arch x86_64.
## Decisión
1. Desarrollo y CI en VM Arch Linux ARM / Asahi-compatible aarch64 (compositor con backend winit anidado y DRM sobre virtio-gpu); el código debe ser portable (sin asunciones x86).
2. Validación en hardware x86_64 de referencia (HW-A/HW-B) obligatoria por hito.
3. Nada se instala en el macOS anfitrión; la VM se crea solo con aprobación del propietario.
## Consecuencias
+ Anfitrión intacto. − Rendimiento en VM no representativo → presupuestos solo se validan en HW-A/HW-B. − ARM64 pasa a ser plataforma de desarrollo de facto; adelanta parte del trabajo de v2.
