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
