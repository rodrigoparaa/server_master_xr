# server_master_xr

Plan maestro y seguimiento del **servidor de archivos privado sobre el iPhone XR** de [Aegis Hardware Enclave](https://github.com/rodrigoparaa/aegis-hardware-enclave).

- **Plan completo:** [PLAN_MAESTRO.md](PLAN_MAESTRO.md): estado, principios, backlog, diseño del servidor (opción A: la llave la tiene el Secure Enclave del XR), API, formato en disco y fases F0–F5 con criterios de terminado.
- **Guía sencilla:** [README_SENCILLO.html](README_SENCILLO.html): explicación visual e interactiva para quien no sabe de informática (descárgala y ábrela en el navegador).
- **Código:** se implementa en el repositorio `aegis-hardware-enclave` (módulo `Sources/FileVault`, rutas `/v1/files/*`, clientes en `CompanionTools/` app de Mac en `MacApp/` y app del iPhone 17 Pro en `PhoneApp/`).

## Estado

**Servidor de archivos terminado (F0–F5, 22/10/2026).** Subida y descarga cifradas por trozos, CLI, barra de menús, Finder y app Archivos (File Provider), app del iPhone 17 Pro con Face ID y «Compartir», texto de documentos para la IA, destrucción total y avisos. Siguiente trabajo posible: el backlog general (pruebas de calidad de la IA, embeddings, caché).
