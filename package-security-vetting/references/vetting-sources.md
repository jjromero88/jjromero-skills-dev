# Vetting Sources — las 5 fuentes obligatorias

Aplican en conjunto, siempre, a todo paquete que no cayó en la vía rápida
de `trusted-origins.md`. Ninguna es opcional. Si alguna no está disponible
en la sesión actual (sin acceso a internet, sin shell del proyecto),
repórtalo explícitamente en vez de omitirla en silencio — ver
`severity-and-reporting.md`.

## 1. OSV.dev — CVEs/advisories conocidos

API gratuita de Google, sin autenticación, agrega 30+ fuentes de
vulnerabilidades (incluida GitHub Advisory Database) y cubre npm y NuGet
entre otros ecosistemas. Es la misma fuente que usa `osv-scanner`, el
scanner oficial de Google.

```
POST https://api.osv.dev/v1/query
Content-Type: application/json

{"package": {"name": "{paquete}", "ecosystem": "npm"}, "version": "{versión}"}
```

Para NuGet, `"ecosystem": "NuGet"` (verifica el string exacto contra el
schema vigente de OSV si el nombre del ecosistema no matchea — puede
haber variado). Respuesta: `vulns[]` con `id`, `summary`, `details`,
`references`, `affected` (rangos de versión y severidad). Para vetear
varios paquetes de una sola vez, existe `POST /v1/querybatch` con una
lista de queries.

## 2. GitHub Advisory Database — link canónico

`https://github.com/advisories` — navegable y con API (REST/GraphQL). En
la práctica, la mayoría de los hallazgos de OSV.dev para npm/NuGet
provienen de aquí — úsala para dar al usuario un link directo y legible al
reporte, no solo el ID crudo de OSV.

## 3. Herramienta oficial del ecosistema (chequeo local real)

Si Claude tiene acceso a shell dentro del proyecto real (no solo
investigando el paquete antes de agregarlo), corre el chequeo oficial
como confirmación adicional — cero scraping, es tooling de primera parte:

- **npm**: `npm audit` (audita `package-lock.json` contra la base de
  datos de advisories de npm).
- **.NET**: `dotnet list package --vulnerable --include-transitive`
  (feature oficial del SDK, misma fuente de datos que GitHub Advisory
  Database).

Si el paquete todavía no está en el proyecto (se está evaluando antes de
instalar), este paso se puede omitir en favor de OSV.dev/GitHub Advisory
— aplícalo sobre todo para confirmar el estado real tras instalar, o al
auditar dependencias ya existentes.

## 4. Socket.dev — riesgo de supply-chain sin CVE reportado

Fuente obligatoria (no solo recomendada) para el ángulo que las bases de
CVE no cubren: scripts de instalación sospechosos, código ofuscado,
cuentas de mantenedor nuevas o anónimas, patrones típicos de paquetes
maliciosos aún no catalogados como vulnerabilidad formal. Página pública
por paquete, sin autenticación:

```
https://socket.dev/npm/package/{paquete}
https://socket.dev/nuget/package/{paquete}
```

Revisa el score y las alertas específicas que reporte (install scripts,
ofuscación, riesgo de mantenedor). Un score bajo o una alerta de riesgo
alto cuenta como hallazgo aunque OSV.dev no reporte ningún CVE.

## 5. Metadata del registro — señales de confianza complementarias

No reemplaza a las 4 anteriores, pero ayuda a detectar un paquete
sospechoso que aún no tiene ni CVE ni alerta de Socket (ej. recién
publicado, typosquat reciente sin actividad todavía):

- **npm**: `https://registry.npmjs.org/{paquete}` (JSON público) — revisa
  fecha de publicación, número de mantenedores, presencia de link a
  repositorio, descargas semanales (vía `https://api.npmjs.org/downloads/point/last-week/{paquete}`).
- **NuGet**: `https://www.nuget.org/packages/{paquete}` — revisa badge de
  publicador verificado, descargas totales, y la sección de
  vulnerabilidades conocidas que NuGet.org muestra directamente en la
  página del paquete (alimentada por GitHub Advisory Database).

Señales de alerta: publicado hace pocos días/semanas, un solo mantenedor
sin historial, cero descargas o descargas anómalamente bajas para lo que
promete el paquete, sin link a repositorio público, nombre muy similar a
un paquete popular.
