# Cosmetra: cómo se trabajó

> **English summary.** Cosmetra is a Spanish-language SaaS for clinical
> records in cosmetology, built in Argentina and subject to health-data
> law. This repository publishes the **method**, not the product: the
> written rules an AI coding agent worked under, how a security audit was
> structured, how every change is verified (an end-to-end suite repeated
> five times, visual snapshots, deliberate sabotage of the tests), and how
> a server migration was planned. One person decides, authorizes each
> change and verifies by hand; the agent reads, proposes and writes code
> under those rules. No code, no history of the main repository, no open
> findings, no infrastructure details. Everything below is in Spanish.

Cosmetra es un SaaS de fichas clínicas para cosmetólogas, en Argentina.
Las fichas son historia clínica (Ley 26.529) y contienen datos de salud,
la categoría más protegida de la Ley 25.326.

Este repositorio muestra el **método**, no el producto: las reglas de
trabajo del agente de IA, cómo se estructuró la auditoría de seguridad,
cómo se verifica cada cambio y cómo se planificó una migración de
servidor.

## Cómo se trabajó

Una persona (el dueño del proyecto) decide, autoriza cada cambio por ítem
y verifica a mano lo que los tests no ven. Un agente de IA (Claude Code)
lee, propone y escribe código bajo reglas escritas: CSS y HTML congelados,
test que falla antes y pasa después, una fase por vez, nada en producción
sin promoción planificada, nada de SQL ni de secretos en sus manos.

El reparto, en una línea: los métodos y criterios, las decisiones técnicas
y de negocio, y toda la ejecución en servidor y base de datos son de la
persona; la lectura, la propuesta, el código y los tests son del agente,
siempre bajo las reglas de `CLAUDE.md` y con aprobación antes de cada fase.

El producto, en dos líneas: una ficha clínica digital en cuatro bloques
(datos personales, historial cosmético, salud general y diagnóstico), con
firma electrónica en el teléfono de la paciente y un formulario público
para que la paciente complete sus datos antes de la consulta.

## Qué hay acá

| Archivo | Qué es |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | Las reglas de trabajo del agente, saneadas. Las reglas que nacieron de un incidente real llevan la marca `[lección]`. |
| [`CHANGELOG-REGLAS.md`](CHANGELOG-REGLAS.md) | Fecha y motivo de cada revisión de esas reglas. |
| [`METODO-AUDITORIA.md`](METODO-AUDITORIA.md) | Cómo se estructuró la auditoría de seguridad: tandas, reglas, formato del hallazgo, escala de gravedad, criterio de bloqueo. |
| [`HALLAZGOS-CERRADOS.md`](HALLAZGOS-CERRADOS.md) | Los hallazgos ya cerrados, con su arreglo. Solo los cerrados. |
| [`METODO-MIGRACION.md`](METODO-MIGRACION.md) | Cómo se planificó y verificó la migración a un servidor nuevo: diferencias intencionales, cinco capas de verificación, 27 riesgos, vuelta atrás por fase. |
| [`METODO-PROMOCION.md`](METODO-PROMOCION.md) | Cómo se compara el entorno de pruebas con producción y cómo se promueve: controles, hashes, caché, vuelta atrás. |
| [`TESTS.md`](TESTS.md) | La suite de pruebas descripta, sin código: estructura, capturas, la regla del ×5, los sabotajes. |

Convención en todos los documentos: **`app`** es producción y **`code`**
es el entorno de pruebas, donde trabaja el agente. Los documentos citan
otros archivos internos (`HISTORIA.md`, `OPERACION.md`, `ESTADO.md`,
`HALLAZGOS.md`, `PLAN-*.md`, `AUDITORIA.md`) que **no están publicados**.

## Qué no hay, a propósito

Código, historial del repositorio principal, hallazgos abiertos,
configuración de servidor, esquema de base de datos, capturas de pantalla,
datos. Es un sistema con datos de salud: ante la duda, no se publica.

Los textos están saneados: nombres de servidores, cuentas, rutas, claves y
accesos fueron reemplazados por descripciones genéricas. Los números de
línea y los nombres de funciones y archivos de código se dejaron solo
donde ilustran el método.

## Números a la fecha

Recuento del 2026-10-07. Cada número apunta al documento que lo respalda.

| Dato | Valor | Dónde |
|---|---|---|
| Tests end-to-end (Playwright) | 258, en 4 archivos | `TESTS.md` |
| Capturas de regresión visual | 83 (28 pantallas o estados, en 3 variantes) | `TESTS.md` |
| Suite completa, repetida 5 veces, 5 workers | 1.290 ejecuciones: 1.285 pasaron, 5 salteadas a propósito, 0 fallaron; ~32 minutos | `TESTS.md` |
| Hallazgos de la auditoría | 26 en 3 tandas; registro consolidado de 53 (con los del servidor y los del cruce legal); 6 bloqueantes identificados | `METODO-AUDITORIA.md` |
| Hallazgos cerrados publicados | 11 de la auditoría + 6 arreglos de seguridad del refactor | `HALLAZGOS-CERRADOS.md` |
| Riesgos registrados en la migración | 27; 10 diferencias intencionales; 9 fases con vuelta atrás | `METODO-MIGRACION.md` |
| Reglas del agente | 558 líneas en su versión interna, 31 revisiones, 15 reglas marcadas `[lección]` | `CLAUDE.md`, `CHANGELOG-REGLAS.md` |

Período del trabajo documentado: septiembre y octubre de 2026.

## Licencia

Textos bajo [Creative Commons Attribution 4.0](LICENSE) (CC BY 4.0).
