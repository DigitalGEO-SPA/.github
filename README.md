# .github — DigitalGEO SPA

Repositorio especial de la organización. Contiene:

- `profile/README.md` — perfil público de la organización en GitHub.
- `.github/workflows/seguridad.yml` — flujo reutilizable de seguridad (gitleaks, Semgrep, dependencias, Trivy) que llama cada repo; `seguridad-sarif.yml` sube los resultados a Code Scanning. Uso en [docs/seguridad/uso-workflow.md](docs/seguridad/uso-workflow.md).
- `docs/seguridad/` — política de seguridad de la Fábrica, plantilla de modelo de amenazas y skill `auditor-seguridad`.
- Archivos de comunidad por defecto (`SECURITY.md`, `CONTRIBUTING.md`, plantillas de issues y pull requests). GitHub los aplica a todo repo de la organización que no tenga los suyos.

Los cambios aquí afectan a todos los repositorios: van por pull request y con revisión de dirección.
