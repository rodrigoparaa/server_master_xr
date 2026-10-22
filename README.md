# server_master_xr

Plan maestro y seguimiento del **servidor de archivos privado sobre el iPhone XR** de [Aegis Hardware Enclave](https://github.com/rodrigoparaa/aegis-hardware-enclave).

- **Plan completo:** [PLAN_MAESTRO.md](PLAN_MAESTRO.md): estado, principios, backlog, diseño del servidor (opción A: la llave la tiene el Secure Enclave del XR), API, formato en disco y fases F0–F5 con criterios de terminado.
- **Código:** se implementa en el repositorio `aegis-hardware-enclave` (módulo `Sources/FileVault`, rutas `/v1/files/*`, clientes en `CompanionTools/` y la futura app del iPhone 17 Pro).

## Próximo paso

**F0:** lectura del cuerpo de las peticiones en streaming en `LocalHTTPServer` (hoy se lee entera en memoria), requisito para subir archivos grandes sin agotar la RAM del XR.
