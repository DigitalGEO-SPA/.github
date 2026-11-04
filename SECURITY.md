# Política de seguridad

## Reportar una vulnerabilidad

No abras un issue público. Escribe a **ventas@digitalgeo.cl** con el asunto `SEGURIDAD — <repositorio>`, indicando:

- repositorio y archivo o componente afectado;
- descripción y pasos para reproducir;
- impacto estimado.

Respondemos dentro de 5 días hábiles.

## Reglas internas

- Ningún secreto (claves, tokens, contraseñas, cadenas de conexión, archivos `.env`, `.pfx`, `.PublishSettings`) entra a git. Los secretos viven en Azure Key Vault o en GitHub Actions secrets.
- Ningún dato real de clientes (configuraciones, evaluaciones de tenant, planos, credenciales de equipos) entra a un repositorio público.
- Si un secreto llega a un commit, se considera filtrado: se rota primero y después se limpia el historial.
- Cuentas de la organización con 2FA obligatorio.

## Fábrica DigitalGEO: auditoría obligatoria

Nada llega a `main` sin auditoría de seguridad. Puertas, severidades, plazos, formato de hallazgo y kit de código seguro: [docs/seguridad/politica.md](docs/seguridad/politica.md).

- Crítico y alto bloquean el merge y la liberación. Medio: 14 días. Excepciones solo con aprobación escrita de Geovanni (riesgo aceptado con ADR y vencimiento).
- CI: cada repo llama al flujo reutilizable [`seguridad.yml`](.github/workflows/seguridad.yml) ([cómo usarlo](docs/seguridad/uso-workflow.md)) y exige el check `seguridad / veredicto`.
- Cada repo mantiene una carpeta `seguridad/` con su [modelo de amenazas](docs/seguridad/plantilla-modelo-amenazas.md), informes, SBOM y riesgos aceptados.
