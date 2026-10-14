# Cómo trabajamos en los repositorios de DigitalGEO

## Ramas
- `main` es siempre la versión vigente y desplegable.
- Cambios grandes o riesgosos: rama `tema/<descripcion-corta>` y pull request hacia `main`.
- Cambios menores de un solo autor pueden ir directo a `main`.

## Commits
Formato: `AAAA-MM-DD <tema> — <resumen>`

```
2026-10-14 lk-api — reintento automático ante HTTP 401
```

- Un commit por unidad de trabajo, con mensaje que explique el porqué si no es obvio.
- Nunca commitear secretos ni datos de clientes (ver [SECURITY.md](SECURITY.md)).

## Versiones
- Herramientas y productos usan [SemVer](https://semver.org/lang/es/): `vMAYOR.MENOR.PARCHE`, como tag de git sobre `main`.
- Cada versión publicada se describe en `CHANGELOG.md` (fecha, qué cambió, qué se rompe).

## Documentación mínima por repositorio
- `README.md`: qué hace, cómo se ejecuta, de qué depende y dónde corre.
- `CHANGELOG.md` desde la primera versión con tag.
- `.env.example` cuando el código lee variables de entorno.
