# Uso del flujo reutilizable de seguridad (puerta E5)

El flujo `seguridad.yml` de este repositorio es la puerta E5 de la [política de seguridad](politica.md). Todo repo de DigitalGEO SPA lo llama en cada pull request hacia `main`, en cada push a `main` y una vez por semana.

| Escáner | Qué revisa | Bloquea |
|---|---|---|
| gitleaks 8.30.1 | Secretos en **todo** el historial de git, no solo en el diff | Cualquier secreto, sin importar el umbral |
| Semgrep CE 1.181.0 | `p/owasp-top-ten` + `p/secrets`, sin login y con `--metrics=off` | Reglas de severidad `ERROR` (umbral `alta`) |
| `dotnet list package --vulnerable --include-transitive` | Paquetes NuGet directos y transitivos | `Critical`/`High` |
| pip-audit 2.10.1 | `requirements*.txt` y `pyproject.toml` | Toda vulnerabilidad conocida (pip-audit no informa severidad) |
| `npm audit --omit=dev` | Dependencias de producción de cada `package.json` | `high` o más. Sin `package-lock.json` también bloquea |
| Trivy 0.75.0 `fs` | Vulnerabilidades del sistema de archivos y configuración de Dockerfile, Bicep (compilado a ARM), Terraform, Kubernetes y workflows | `CRITICAL`/`HIGH` |

Los escáneres de dependencias se activan solos según los archivos versionados que tenga el repo (`*.sln`/`*.csproj`, `requirements*.txt`/`pyproject.toml`, `package.json`).

## 1. Archivo que va en cada repo

Crear `.github/workflows/seguridad.yml` en el repo:

```yaml
name: seguridad

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  schedule:
    - cron: "17 9 * * 1"   # lunes 06:17 Chile (UTC-3): CVE nuevos contra lo que está en main
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: seguridad-${{ github.ref }}
  cancel-in-progress: true

jobs:
  seguridad:
    uses: DigitalGEO-SPA/.github/.github/workflows/seguridad.yml@<sha> # <fecha o versión del kit>
    with:
      umbral: alta
```

Reemplazar `<sha>` por el SHA de commit **completo** (40 caracteres) de `main` en `DigitalGEO-SPA/.github` después de mergear el kit:

```bash
git ls-remote https://github.com/DigitalGEO-SPA/.github refs/heads/main
```

Nunca se usa `@main` ni un tag: la referencia móvil permitiría que un cambio en este repo cambie la puerta de todos los repos sin revisión.

Reglas para el archivo del repo:

- **Sin `secrets: inherit`** y sin secretos: el flujo no los necesita.
- **Sin `pull_request_target`.** Los PR de forks corren con `pull_request`, con token de solo lectura y sin secretos.
- `permissions: contents: read` en el nivel del archivo. El flujo reutilizable no puede pedir más que eso.

## 2. Check obligatorio en la protección de `main`

En *Settings → Branches → main* (o en el *ruleset* de la organización), marcar como requerido el check:

```
seguridad / veredicto
```

`veredicto` consolida todos los jobs y falla cerrado: un job fallido, cancelado o con error de herramienta (por ejemplo, si Trivy no puede bajar su base de CVE) bloquea. Un escáner desactivado o un lenguaje ausente aparece como `skipped` y pasa.

## 3. Entradas (`with:`)

| Entrada | Valor por defecto | Uso |
|---|---|---|
| `umbral` | `alta` | `critica` · `alta` · `media` · `baja`. Por política no se usa uno más permisivo que `alta`. `critica` deja pasar altos y exige un riesgo aceptado aprobado por Geovanni |
| `gitleaks` | `true` | `false` desactiva el escáner (exige riesgo aceptado) |
| `semgrep` | `true` | Ídem |
| `dependencias` | `true` | Ídem (.NET, Python y Node juntos) |
| `trivy` | `true` | Ídem |
| `semgrep-configs` | `p/owasp-top-ten p/secrets` | Agregar reglas propias del repo, p. ej. `p/owasp-top-ten p/secrets seguridad/semgrep/` |
| `pip-audit-ignorar` | vacío | IDs separados por espacio (`PYSEC-…`, `GHSA-…`). Cada ID con su ADR de riesgo aceptado |
| `dotnet-version` | `8.0.x` | Solo si el repo no tiene `global.json` |
| `python-version` | `3.12` | Python con que corren Semgrep y pip-audit |
| `node-version` | `22` | Node para `npm audit` |

Salida: `resultado` (`aprobado` | `bloqueado`).

Ejemplo de un repo .NET con reglas propias y un riesgo aceptado vigente:

```yaml
jobs:
  seguridad:
    uses: DigitalGEO-SPA/.github/.github/workflows/seguridad.yml@<sha> # <fecha o versión del kit>
    with:
      umbral: alta
      semgrep-configs: "p/owasp-top-ten p/secrets p/csharp seguridad/semgrep/"
      dotnet-version: "10.0.x"
```

## 4. Subir resultados a Code Scanning (opcional)

Los SARIF de gitleaks, Semgrep y Trivy siempre quedan como artefactos de la ejecución (`sarif-*`, 30 días). Para verlos en la pestaña *Security* del repo hay que agregar un segundo job que llama a `seguridad-sarif.yml`. Es el único que pide `security-events: write`, y por eso va aparte:

```yaml
jobs:
  seguridad:
    uses: DigitalGEO-SPA/.github/.github/workflows/seguridad.yml@<sha> # <fecha o versión del kit>
    with:
      umbral: alta

  sarif:
    needs: seguridad
    if: ${{ !cancelled() }}
    permissions:
      contents: read
      security-events: write
    uses: DigitalGEO-SPA/.github/.github/workflows/seguridad-sarif.yml@<sha> # mismo SHA
```

Requisito: Code Scanning disponible. En repos públicos es gratis. En repos privados exige GitHub Advanced Security; sin ella, el job `sarif` falla y no se usa.

## 5. Falsos positivos y excepciones

Ninguna supresión se agrega sin su respaldo. Un falso positivo se documenta en el informe de triage; un riesgo real que no se corrige es un **riesgo aceptado** con ADR, aprobado por escrito por Geovanni y con fecha de vencimiento (ver [política](politica.md#5-excepciones-riesgo-aceptado)).

| Escáner | Mecanismo | Formato exigido |
|---|---|---|
| gitleaks | `.gitleaksignore` (huella del hallazgo) | Comentario previo `# FP: <motivo>` o `# ADR-NNNN vence AAAA-MM-DD` |
| Semgrep | `# nosemgrep: <id-regla>` en la línea | `-- FP: <motivo>` o `-- ADR-NNNN` en el mismo comentario |
| pip-audit | entrada `pip-audit-ignorar` | Comentario en el YAML del repo con el ADR y su vencimiento |
| npm audit | no se suprime: se actualiza o se reemplaza el paquete | — |
| .NET | no se suprime: se actualiza o se fija la versión transitiva | — |
| Trivy | `.trivyignore` en la raíz | `# ADR-NNNN vence AAAA-MM-DD` encima de cada ID. Se puede usar `exp:AAAA-MM-DD` para que expire solo |

Un secreto detectado por gitleaks **nunca** es una excepción: se rota primero y después se limpia el historial ([SECURITY.md](../../SECURITY.md)).

## 6. Mantenimiento del kit

- Las acciones están fijadas por SHA con la versión en comentario. Los binarios (gitleaks, Trivy) se bajan con versión fija y se verifican con un SHA-256 fijado en el flujo. Semgrep y pip-audit se instalan con versión exacta desde PyPI.
- Se eligen versiones con al menos dos semanas de publicadas.
- Al subir una versión: cambiar la variable `*_VERSION` y su `*_SHA256` (desde el archivo `checksums.txt` oficial de la versión) en el mismo PR, y probarlo en un repo piloto antes de mover el SHA en los demás.
- Cada repo actualiza su `@<sha>` por PR. Dependabot (`package-ecosystem: github-actions`) propone las actualizaciones de SHA de las acciones en este repo y en cada repo que llama.

## 7. Si el repo `.github` de la organización fuera privado

Los flujos reutilizables de un repo privado solo se pueden llamar si en *DigitalGEO-SPA/.github → Settings → Actions → General → Access* se elige «Accessible from repositories in the 'DigitalGEO-SPA' organization». Si el repo es público (lo es, porque publica el perfil de la organización), no hace falta nada.
