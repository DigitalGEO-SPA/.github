# Modelo de amenazas — `<producto>` (plantilla)

> Se copia a `seguridad/modelo-amenazas.md` de cada repo en la etapa **E1 Especificación**. Sin este documento completo no se aprueba la spec (ver [política](politica.md)). Se actualiza cuando cambia un activo, un flujo, un límite de confianza o un supuesto, y se revisa en cada liberación.
>
> Instrucciones: borrar las notas entre `<…>`. Ninguna celda queda vacía: si algo no aplica, se escribe «no aplica» y por qué. «Probablemente está bien» no es una respuesta válida.

| Campo | Valor |
|---|---|
| Producto / repo | `<nombre>` / `<DigitalGEO-SPA/repo>` |
| Versión o commit | `<vX.Y.Z / sha corto>` |
| Fecha | `<AAAA-MM-DD>` |
| Autor del modelo | `<persona o «Claude (opus)»>` |
| Revisión independiente | `<«Claude (fable)», obligatoria si el producto es A′>` |
| ¿Es A′? | `<sí/no — identidad, aislamiento entre tenants, pagos, datos personales o precios>` |
| Aprobó | Geovanni Luna, `<fecha>` |

## 1. Contexto

- **Qué hace el producto:** `<dos o tres líneas>`
- **Quién lo usa:** `<roles: cliente final, administrador del cliente, operador DigitalGEO, sistema externo…>`
- **Dónde corre:** `<Azure: suscripción, grupo de recursos, servicios (Container Apps, Functions, SQL, Key Vault…), región>`
- **Exposición:** `<internet público / solo tenant M365 / red privada>`
- **Multi-tenant:** `<no / sí: cómo se identifica el tenant y dónde se aplica el aislamiento>`

## 2. Diagrama de flujo de datos

```
<diagrama en texto o Mermaid. Marcar cada límite de confianza con ═══ y numerar los flujos (F1, F2…)>

[Navegador] ──F1 HTTPS──▶ ║ Easy Auth ║ ──F2──▶ [API] ──F3 MI──▶ [SQL (RLS)]
                                                   └──F4 MI──▶ [Key Vault]
```

## 3. Activos

| ID | Activo | Tipo de dato | Clasificación | ¿Dato personal (Ley 21.719)? | ¿Dato sensible? | Dónde vive |
|---|---|---|---|---|---|---|
| A1 | `<p. ej. ficha de empleado>` | `<RUT, nombre, correo, saldo de puntos>` | `<público / interno / confidencial / restringido>` | `<sí/no>` | `<sí/no: salud, biométrico, etc.>` | `<tabla, blob, log>` |
| A2 | `<secreto de firma SAML>` | `<certificado>` | restringido | no | no | `<Key Vault kv-…>` |

## 4. Límites de confianza y flujos

| ID | Origen → destino | Cruza límite | Autenticación | Autorización | Datos (activos) | Canal |
|---|---|---|---|---|---|---|
| F1 | `<navegador → Easy Auth>` | internet → Azure | `<Entra ID OIDC + MFA>` | `<rol de app>` | A1 | TLS 1.2+ |
| F3 | `<API → SQL>` | app → datos | `<identidad administrada>` | `<RLS por TenantId + filtro en app>` | A1 | TDS cifrado |

## 5. Supuestos

Cada supuesto es algo que **no** se verifica en este repo. Si un supuesto falla, el modelo se rehace.

| ID | Supuesto | Quién lo garantiza | Cómo se verifica | Si falla… |
|---|---|---|---|---|
| S1 | `<«asignación requerida» activa en la app empresarial de Entra>` | `<Geovanni, en el tenant>` | `<revisión en cada liberación>` | `<cualquier usuario del tenant entra>` |
| S2 | `<la cookie de Easy Auth es SameSite=Lax>` | `<plataforma>` | `<DevTools tras login>` | `<CSRF explotable>` |

## 6. Amenazas STRIDE por activo y flujo

Una fila por amenaza concreta. Se recorre cada flujo y cada activo con las seis letras. Las amenazas de aislamiento entre tenants (salir del propio tenant) se analizan siempre, aunque el producto hoy tenga un solo cliente.

| ID | Activo / flujo | STRIDE | Amenaza (quién, cómo) | Probabilidad | Impacto | Severidad | Controles existentes (archivo:línea) | Control faltante | Prueba de abuso (E3) | Estado |
|---|---|---|---|---|---|---|---|---|---|---|
| T1 | F3 / A1 | **S** Suplantación | `<usuario de tenant B falsifica TenantId en el cuerpo>` | `<alta/media/baja>` | `<alto>` | `<crítico/alto/medio/bajo>` | `<middleware X (api/x.cs:40)>` | `<RLS en SQL>` | `<tests/abuso/tenant_cruzado_test.cs>` | `<mitigada / abierta / riesgo aceptado>` |
| T2 | A1 | **T** Manipulación | | | | | | | | |
| T3 | F1 | **R** Repudio | `<acción de admin sin registro de auditoría>` | | | | | | | |
| T4 | A1 | **I** Divulgación | `<IDOR en GET /empleados/{id}>` | | | | | | | |
| T5 | F1 | **D** Denegación | `<sin límite de tasa en /login>` | | | | | | | |
| T6 | A2 | **E** Elevación | `<usuario se asigna rol admin vía PATCH>` | | | | | | | |

Guía rápida por letra:

- **S — Suplantación:** tokens de otra app cliente, cabeceras de identidad inyectables (`X-MS-CLIENT-PRINCIPAL`, `X-Forwarded-For`), modo demo sin contraseña, ausencia de MFA.
- **T — Manipulación:** CSRF, plantillas o archivos subidos, parámetros de precio o de tenant controlados por el cliente, despliegue (`workflow_run`, forks, acciones sin SHA).
- **R — Repudio:** acciones sin bitácora, bitácora editable por el mismo actor, relojes sin sincronizar.
- **I — Divulgación:** IDOR, datos de otro tenant, datos personales en logs, errores con detalle, SSRF, secretos en app settings.
- **D — Denegación:** sin límite de tasa por IP y por usuario, cargas sin tope de tamaño, ZIP bomba, abuso de cuota de correo.
- **E — Elevación:** roles asignables por el propio usuario, alcance «plataforma» activable sin prueba de identidad, identidad administrada con permisos de más.

## 7. Controles (kit de código seguro)

Marcar cada control del [kit](politica.md#6-kit-de-código-seguro). Una desviación sin riesgo aceptado es un hallazgo.

| Control | Cumple | Evidencia (archivo:línea, recurso Azure, prueba) |
|---|---|---|
| Autorización denegada por defecto, por política | `<sí/no/parcial>` | |
| Filtro por tenant en SQL (RLS) **y** en la app | | |
| OIDC sin secretos; sesiones con expiración; MFA | | |
| Secretos solo en Key Vault o identidad administrada | | |
| Validación por esquema, límites de tamaño, codificación de salida | | |
| CSP, HSTS, cabeceras, CORS con lista explícita | | |
| Límite de tasa por IP y por usuario | | |
| Auditoría de acciones sin datos personales en logs; retención | | |
| Dependencias fijadas, Dependabot activo, sin paquetes abandonados | | |
| Bicep revisado; nada público que no deba serlo | | |
| CI: `seguridad / veredicto` obligatorio en `main` | | |

## 8. Requisitos de seguridad como criterios de aceptación (OWASP ASVS 4.0.3, nivel 2)

Solo los capítulos y requisitos que aplican al producto, cada uno con su prueba. Lo que no aplica se lista con el motivo.

| Requisito ASVS | Descripción breve | Criterio de aceptación medible | Prueba (E3) | Estado |
|---|---|---|---|---|
| V1.4.1 | Control de acceso aplicado en un punto confiable del servidor | `<todo endpoint pasa por la política X; ninguno con [AllowAnonymous] salvo /health>` | `<test que enumera rutas>` | |
| V2.x | Autenticación | `<MFA exigido a rol admin>` | | |
| V3.x | Gestión de sesiones | `<expiración absoluta 8 h, inactividad 30 min>` | | |
| V4.1.x / V4.2.1 | Control de acceso / IDOR | `<GET /x/{id} de otro tenant → 404>` | `<abuso: tenant_cruzado>` | |
| V5.x | Validación, saneamiento y codificación | | | |
| V7.1.x | Registro sin datos sensibles | `<logs sin RUT/correo; prueba que busca patrones>` | | |
| V8.x | Protección de datos | | | |
| V9.x | Comunicaciones | `<TLS 1.2+, HSTS>` | | |
| V12.x | Archivos y recursos | `<tope de tamaño, tipos permitidos, sin rutas>` | | |
| V13.x | API y servicios web | `<CORS con lista, sin métodos de más>` | | |
| V14.x | Configuración | `<cabeceras, dependencias sin CVE altos>` | | |

No aplica: `<V6 criptografía almacenada — no se cifran datos en la app, lo hace TDE de Azure SQL; …>`

## 9. Datos personales — Ley 21.719

| Aspecto | Respuesta |
|---|---|
| Datos personales que se tratan (ref. activos) | `<A1: RUT, nombre…>` |
| Datos sensibles (salud, biométricos, origen, etc.) | `<ninguno / cuáles y base de licitud reforzada>` |
| Rol de DigitalGEO | `<responsable / encargado por cuenta del cliente (contrato)>` |
| Base de licitud | `<consentimiento / contrato / obligación legal / interés legítimo>` |
| Finalidad y minimización | `<qué se usa y qué se descartó>` |
| Retención y eliminación | `<plazo, mecanismo, prueba>` |
| Derechos del titular (acceso, rectificación, supresión, oposición, portabilidad, bloqueo) | `<cómo se atienden, plazo>` |
| Medidas de seguridad (deber de seguridad) | `<ref. sección 7>` |
| Notificación de vulneraciones | `<quién notifica a la Agencia y a los titulares, plazo>` |
| Transferencia internacional | `<región Azure, subencargados>` |
| Datos personales en logs, respaldos y entornos dev | `<seudonimización en dev; logs sin datos personales>` |
| Evaluación de impacto requerida | `<sí/no y por qué>` |

## 10. Riesgos aceptados

Solo Geovanni acepta riesgos, por escrito. Cada uno tiene ADR y fecha de vencimiento; al vencer, vuelve a bloquear.

| ID | Amenaza (T#) | Severidad | Motivo para aceptar | Control compensatorio | ADR | Aprobó (fecha) | Vence |
|---|---|---|---|---|---|---|---|
| R1 | `<T5>` | `<medio>` | | | `<docs/adr/00NN-…md>` | Geovanni, `<AAAA-MM-DD>` | `<AAAA-MM-DD>` |

## 11. Pendientes y próxima revisión

- `<amenaza abierta, responsable, fecha>`
- Próxima revisión: `<antes de la liberación vX.Y / AAAA-MM-DD>`
