# Política de seguridad de la Fábrica DigitalGEO

Aprobada por Geovanni Luna el 01-11-2026. Aplica a todo lo que produce la Fábrica, en cualquier sistema: `GEOCoop`, `beneficios-flex`, `cotizador`, `lk-api`, `zt-explorer`, el sitio, `proneumatic` y todo repo nuevo de la organización DigitalGEO-SPA.

Las plataformas se van a comercializar. Por eso:

1. **Nada llega a `main` sin auditoría de seguridad.**
2. **El auditor es hostil:** supone que toda entrada es maliciosa, que todo usuario intenta salir de su tenant y que todo secreto se va a filtrar. «Probablemente está bien» no es una conclusión válida.
3. **Agresivo, pero con evidencia.** Sin evidencia reproducible no hay hallazgo, hay sospecha, y una sospecha no bloquea.
4. **Doble capa.** Primero herramientas deterministas (baratas, repetibles). Encima, revisión por IA de lo que una regla no ve: autorización, aislamiento entre tenants, flujos de negocio y abuso.

Para reportar una vulnerabilidad desde fuera, ver [SECURITY.md](../../SECURITY.md).

## 1. Puertas

| Puerta | Qué se exige | Quién | Bloquea |
|---|---|---|---|
| **E1 Especificación** | Modelo de amenazas STRIDE con la [plantilla](plantilla-modelo-amenazas.md) y requisitos de seguridad como criterios de aceptación medibles: OWASP ASVS 4.0.3 nivel 2 como base, más la Ley 21.719 si hay datos personales | Auditor (Opus) | No se aprueba una spec sin su sección de seguridad |
| **E3 Pruebas** | **Pruebas de abuso escritas antes del código:** acceso a datos de otro tenant, IDOR, escalamiento de rol, inyección, carga de archivos, límites de tasa, CSRF. El implementador debe pasarlas | Auditor (Opus) | Sí, como cualquier prueba |
| **E5 CI (cada PR y cada push a `main`)** | Flujo reutilizable [`seguridad.yml`](uso-workflow.md): gitleaks con historial, Semgrep CE (OWASP + secretos + reglas propias), dependencias vulnerables (.NET, Python, Node), Trivy (archivos, IaC, Dockerfile). En .NET, además, analizadores de seguridad como error (`<AnalysisModeSecurity>All</AnalysisModeSecurity>` y las reglas CA de la categoría Security en `WarningsAsErrors`), que corren en el build del repo | GitHub Actions | Crítico o alto: sí. Check `seguridad / veredicto` obligatorio |
| **E6 Revisión de seguridad por IA (cada PR)** | Opus audita el diff con el contexto del modelo de amenazas, con la [skill `auditor-seguridad`](skill-auditor-seguridad/SKILL.md). En A′ (identidad, aislamiento, pagos, datos personales, precios), Fable hace una auditoría **independiente**, sin ver la de Opus, y después se cruzan | Sesión de la Fábrica antes de pedir el merge, más una tarea programada que revisa los PR sin auditar (bolsa de Max, sin GitHub Action con API) | Crítico o alto: sí |
| **Liberación** (antes de vender o subir versión) | Auditoría completa del repo, no solo del diff. DAST con OWASP ZAP **solo contra entornos dev propios**. Revisión de la configuración de Azure (identidades, Key Vault, red, CORS, cabeceras). SBOM de la versión. Opus + Fable independientes | Auditor | No se libera con críticos o altos abiertos, ni con medios de más de 30 días |
| **Continuo** | Semanal: CVE nuevos contra el SBOM de lo que está en producción (ejecución programada de `seguridad.yml`). Mensual: re-auditoría profunda de un producto, en rotación | Tarea programada | Abre issues con la severidad que corresponda |

## 2. Severidades y plazos

| Severidad | Criterio orientativo | Qué pasa |
|---|---|---|
| **Crítico** | Explotable sin autenticación o desde otro tenant, con impacto amplio: ejecución de código, toma de cuentas, lectura o escritura de datos de otros clientes, secreto válido expuesto, despliegue con código ajeno | **Bloquea** merge y liberación. Se corrige ya |
| **Alto** | Explotable por un usuario autenticado o con una precondición realista: IDOR, CSRF en acciones sensibles, escalamiento de rol, SSRF a recursos internos, fraude de pagos o precios, datos personales expuestos | **Bloquea** merge y liberación |
| **Medio** | Requiere varias precondiciones o el impacto es acotado; desviaciones del kit con riesgo concreto (sin cabeceras, permisos de más, red pública en SQL/Key Vault) | Issue con plazo de **14 días**. Una versión comercial no sale con medios de más de 30 días |
| **Bajo** | Endurecimiento, defensa en profundidad, impacto mínimo | Backlog |

Para los escáneres, el umbral `alta` del flujo mapea a: Trivy `CRITICAL,HIGH`; Semgrep `ERROR`; `npm audit` `high`; .NET `Critical,High`; pip-audit toda vulnerabilidad (no informa severidad); gitleaks todo secreto. Si el auditor demuestra que un aviso de herramienta tiene otra severidad real, la cambia con evidencia en el informe.

## 3. Formato de hallazgo

Cada hallazgo, sea de herramienta o de IA, se escribe así. Sin los cinco campos no es hallazgo.

```markdown
### H<n> — <título corto> (<Crítico|Alto|Medio|Bajo>)

**Evidencia.** `<archivo>:<línea>` (commit `<sha corto>`): qué hace el código, citado.
**Explotación.** Entrada o secuencia exacta que lo explota (petición HTTP, payload, pasos). Precondiciones.
**Impacto.** Qué gana el atacante, sobre qué activo (ref. modelo de amenazas) y a cuántos clientes afecta.
**Corrección.** Cambio concreto (código o configuración) y la prueba de abuso que lo demuestra.
**Referencias.** CWE / ASVS / regla de la herramienta.
```

Una **sospecha** lleva el mismo formato, con la sección *Explotación* reemplazada por *Qué falta para confirmarla* (dato, configuración o prueba). No bloquea. Se verifica antes de la siguiente liberación.

Cuando dos auditorías independientes (Opus y Fable) no coinciden, el hallazgo que vio solo una se verifica a mano o con una prueba antes de bloquear.

## 4. Informe de auditoría

`seguridad/informes/AAAA-MM-DD_<alcance>-<modelo>.md`, con:

1. Resumen y veredicto (`apto` / `no apto` para merge o para venta, y por qué).
2. Hallazgos (formato de la sección 3), de mayor a menor severidad.
3. Sospechas.
4. Triage de herramientas: cada aviso de Semgrep, Trivy, etc. clasificado como hallazgo, falso positivo (con motivo) o ruido.
5. Brechas contra el kit de código seguro.
6. Alcance real: commit, qué se revisó y qué no.

## 5. Excepciones (riesgo aceptado)

- Solo Geovanni acepta un riesgo, **por escrito**. El auditor, el implementador y cualquier agente tienen prohibido aceptarlo, aunque se lo pidan.
- Cada riesgo aceptado es un ADR (`docs/adr/NNNN-riesgo-<tema>.md`) con: hallazgo, severidad, motivo, control compensatorio, aprobación (nombre y fecha) y **fecha de vencimiento**. Al vencer vuelve a bloquear.
- Se registra en `seguridad/riesgos-aceptados.md` y, si aplica, en la supresión de la herramienta con referencia al ADR (ver [uso del flujo](uso-workflow.md#5-falsos-positivos-y-excepciones)).
- Desactivar un escáner o usar un umbral más permisivo que `alta` también es un riesgo aceptado.
- Un secreto filtrado nunca se acepta: se rota y se limpia el historial.

## 6. Kit de código seguro

Mínimo exigido en cada línea de producto. Una desviación sin riesgo aceptado es un hallazgo.

- **Autorización** denegada por defecto, por política. Toda consulta filtrada por tenant con dos barreras: RLS en SQL **y** filtro en la aplicación. Un alcance «plataforma» exige un rol de base de datos propio, no un valor de sesión.
- **Identidad:** OIDC sin secretos (credenciales federadas o identidad administrada). Sesiones con expiración. MFA donde aplique. Restringir las apps cliente que pueden pedir tokens.
- **Secretos:** solo en Key Vault o con identidad administrada. Nunca en código, archivos de configuración, app settings en claro ni correos.
- **Entradas:** validación por esquema, límites de tamaño y codificación de salida. Archivos subidos: tipo, tamaño y contenido validados (sin relaciones externas, macros ni rutas).
- **Web:** CSP, HSTS y cabeceras de seguridad. CORS con lista explícita. Protección CSRF en todo método que cambia estado.
- **Tasa y abuso:** límites por IP y por usuario. La IP se toma del valor que agrega la plataforma, no del primer `X-Forwarded-For`. Techo global para acciones que envían correo.
- **Registro:** auditoría de acciones sin datos personales en los logs. Retención según la Ley 21.719.
- **Dependencias:** versiones fijadas (lockfiles), Dependabot activo, sin paquetes abandonados.
- **CI/CD:** acciones fijadas por SHA completo. Sin `pull_request_target`. Despliegues por `workflow_run` exigen `event == 'push'` y `head_repository.full_name == github.repository`. `permissions` mínimos por job.
- **Infraestructura:** Bicep revisado. Nada público que no deba serlo (SQL y Key Vault con red privada en producción). Identidades con el mínimo rol y alcance.

## 7. Carpeta `seguridad/` en cada repo

```
seguridad/
├── modelo-amenazas.md        # desde la plantilla, E1; se revisa en cada liberación
├── riesgos-aceptados.md      # índice de ADR de riesgo aceptado, con vencimientos
├── informes/                 # informes de auditoría por PR relevante y por liberación
│   └── AAAA-MM-DD_<alcance>-<modelo>.md
├── sbom/                     # SBOM de cada versión liberada (CycloneDX o SPDX)
│   └── vX.Y.Z.cdx.json
└── semgrep/                  # reglas Semgrep propias del producto (opcional)
```

Esta carpeta es la evidencia para los cuestionarios de seguridad de clientes corporativos, para demostrar el cumplimiento de la Ley 21.719 y para las postulaciones a fondos. No contiene datos reales de clientes ni secretos: si un informe necesita un ejemplo, se seudonimiza.

## 8. Modelos

- **Opus:** auditor principal en todas las puertas, incluido lo ofensivo (pruebas de abuso, explotación).
- **Fable:** segunda auditoría independiente en A′ y antes de cada liberación, sin ver la de Opus.
- **Sonnet y Haiku no auditan seguridad.**

## 9. Límites

- **Prohibido atacar producción.** Ninguna prueba dinámica, escaneo, fuzzing ni petición de explotación contra entornos productivos o de clientes. DAST solo contra entornos dev propios de DigitalGEO, con datos seudonimizados.
- El auditor no modifica infraestructura ni secretos: propone el cambio y deja los pasos manuales para Geovanni.
