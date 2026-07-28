# Severity and Reporting

## Clasificación de severidad

Usa la severidad tal como la reporta la fuente (OSV.dev/GitHub Advisory
suelen traer `CRITICAL`/`HIGH`/`MEDIUM`/`LOW`; Socket.dev reporta un score
0-100 por categoría). Si dos fuentes discrepan, reporta la más alta y
menciona la discrepancia — nunca promedies ni minimices el hallazgo más
severo.

| Severidad | Recomendación por defecto |
|---|---|
| Crítica/Alta, o alerta de Socket.dev de malware/backdoor confirmado | No instalar. Sugerir alternativa de inmediato, sin esperar confirmación del usuario para proponerla. |
| Media | No instalar por defecto, pero la decisión es más discutible — expón el trade-off (qué tan crítico es el paquete para la feature vs. el riesgo) y deja decidir. |
| Baja, o solo señales blandas (paquete nuevo, pocos mantenedores, sin hallazgo formal) | Se puede instalar, pero repórtalo igual — el usuario decide si quiere esperar a que el paquete madure o instalarlo ya. |
| Sin hallazgos en ninguna de las 5 fuentes | Repórtalo explícitamente como "sin hallazgos" — nunca lo omitas asumiendo que "no news is no need to mention it". |

## Plantilla de aviso al usuario

Reporta siempre, con o sin hallazgos, algo con esta forma (adapta el tono,
no copies literal):

```
Chequeo de seguridad de `{paquete}` ({ecosistema}):
- OSV.dev: {sin hallazgos | N CVEs, el más severo: {id} ({severidad}) — {resumen}, link: {url}}
- GitHub Advisory: {link directo si aplica}
- Socket.dev: {score/alertas relevantes}
- Registro: {antigüedad, mantenedores, descargas — solo si hay algo notable}

{Si hay hallazgo relevante:} Recomiendo no instalarlo. Alternativa sugerida: {paquete alternativo, con una razón breve}.
¿Deseas instalarlo de todas formas bajo tu responsabilidad, usar la alternativa, o que busque otra opción?
```

## Patrón de consentimiento "instalar bajo tu responsabilidad"

Si el usuario decide instalar pese al hallazgo, la instalación puede
proceder — pero:
- La decisión debe quedar **explícita** en la conversación (el usuario la
  confirma, Claude no la asume ni la infiere de un silencio).
- Se persiste en `.claude/skill-decisions.md` (ver `SKILL.md`) como
  "riesgo aceptado por el usuario el {fecha}: {resumen del hallazgo}" —
  para que quede trazado y no se re-pregunte cada vez, pero tampoco se
  pierda el registro de que hubo un riesgo conocido y aceptado.

## Flujo de sugerir alternativa

Cuando se recomienda no instalar, Claude puede:
- Proponer directamente un paquete alternativo conocido/confiable que
  resuelva el mismo caso de uso (si lo conoce con confianza razonable).
- O preguntarle al usuario si prefiere nombrar él mismo la alternativa.
- O, si el usuario lo pide, investigar opciones vetadas (aplicando este
  mismo chequeo a cada candidata antes de proponerla — no basta con que
  "suene confiable").

## Fallback: sin acceso a herramientas de chequeo

Si en la sesión actual no hay `WebFetch`/`WebSearch` disponibles, ni shell
con acceso a internet para correr `npm audit`/`dotnet list package
--vulnerable`, Claude no puede completar el chequeo obligatorio. En ese
caso:
- Dilo explícitamente — nunca simules que el chequeo se hizo ni instales
  "total, seguro está bien".
- Pregunta al usuario cómo proceder: que él verifique manualmente (dale
  los links directos a `https://osv.dev`, `https://github.com/advisories`,
  y la página de Socket.dev del paquete para que lo revise), o que
  autorice expresamente instalar sin chequeo, asumiendo el riesgo.
