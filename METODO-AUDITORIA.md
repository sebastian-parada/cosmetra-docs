# Método de la auditoría

Cómo se estructuró la auditoría completa de Cosmetra, posterior al refactor
de módulos (desde el 2026-09-29). Este documento describe el método y los
conteos; los hallazgos cerrados están en `HALLAZGOS-CERRADOS.md`; los
abiertos no se publican.

## Objetivo

Conocer y analizar todo el código (errores, vulnerabilidades y
oportunidades de mejora) para hacer el sistema serio y profesional.
Después, **el dueño decide** qué se arregla antes del piloto, qué acepta
como está y qué queda para después. Contexto legal: los datos son de salud;
aplican la Ley 25.326 (protección de datos personales) y la Ley 26.529
(derechos del paciente e historia clínica).

## Las seis tandas

Cada tanda es una sesión nueva del agente, con un prompt aprobado de
antemano que vale como confirmación de fase. Las tandas son de **solo
lectura**: la auditoría no arregla nada.

| Tanda | Qué cubre | Estado |
|---|---|---|
| 0 | Las tres preguntas que no admiten "parece": inyección SQL, aislamiento entre clínicas, secretos en el código. Antes, un diff de la copia real del servidor contra la del repo | Hecha |
| 1 | El PHP completo: autenticación, endpoints públicos, rate limiting, firmas, retención, transacciones | Hecha |
| 2 | El cliente: XSS, almacenamiento en el navegador, tokens en URL, terceros, cabeceras, CORS | Hecha |
| 3 | Correctitud y datos | En pausa |
| 4 | Cumplimiento legal (25.326 y 26.529) | En pausa |
| 5 | Profesionalismo y robustez | Después del piloto |
| 6 | Mantenibilidad; cobertura medida de los tests | Después del piloto |

**Punto de parada.** El plan fijó que después de las tandas 0, 1 y 2 se
paraba a decidir con el dueño si se seguía auditando o se arreglaba lo
grave primero. Se decidió arreglar primero los bloqueantes y recién
después seguir. Las tandas 5 y 6 quedaron para después del piloto desde el
principio.

## Las siete reglas comunes

1. Solo lectura de código. No se arregla nada, ni siquiera lo de una
   línea.
2. Nada contra producción, bajo ningún concepto.
3. Nada que escriba en el entorno de pruebas sin autorización explícita;
   toda prueba dinámica se propone con pasos, qué escribe y cómo se
   limpia.
4. Un crítico explotable hoy por un tercero sin cuenta **corta la tanda**:
   se avisa de inmediato.
5. Lo que ya está anotado (listas previas, hallazgos del refactor, tandas
   anteriores) se **cita**, no se redescubre ni se presenta como nuevo.
   Solo lo adicional cuenta como nuevo.
6. Cada hallazgo con el formato obligatorio, en la sección de su tanda;
   índice actualizado; documentos copiados a la carpeta compartida.
7. Cada tanda cierra con un resumen: cuántos hallazgos, cuántos nuevos,
   cuántos bloquean el piloto según la propuesta.

## Objeto del análisis

El PHP que se auditó fue **la copia real del servidor**, bajada por el
dueño, no la del repo: en ese momento los dos diferían. El primer paso de
la tanda 0 fue el diff triple (repo, pruebas, producción). Resultado:
diferían solo en la configuración de la base; la lógica era idéntica, así
que lo auditado valía para los tres.

## El formato obligatorio del hallazgo

```
### [T<tanda>-NN] Título corto que afirma el problema
- Gravedad: crítica | alta | media | baja
- Verificación: leído en el código | reproducido (cómo) | sin verificar
  (qué prueba haría falta y si escribe datos)
- Dónde: archivo:línea, función() (los tres, siempre)
- Qué es: dos o tres frases, técnicas.
- Impacto concreto: qué dato de qué paciente o qué cosmetóloga, quién
  puede explotarlo o cuándo se dispara solo, y qué ve o pierde la afectada.
- Origen: [NUEVO] | [YA EN LA LISTA DEL DUEÑO] | [YA EN HALLAZGOS DEL
  REFACTOR] — combinables; si amplía algo conocido, solo lo adicional
  cuenta como nuevo.
- Bloquea el piloto (propuesto): sí | no — el porqué en una frase.
- Decisión del dueño: pendiente
```

Los identificadores llevan la tanda (`T0`, `T1`, `T2`); los hallazgos
que aparecieron durante los arreglos llevan `AR` y los del cruce legal
`CL`. La redacción es deliberadamente **no operativa**: se describe el
defecto y su consecuencia, no el procedimiento para explotarlo.

Dentro de un hallazgo el número de línea sí va (junto a archivo y
función): el código está congelado mientras se audita y la línea es
evidencia. Fuera de los hallazgos, los documentos citan archivo y
función, nunca línea suelta: de 42 referencias con número de línea en el
documento de estado del refactor, más de la mitad ya no existían tras las
migraciones.

## Gravedad anclada a impacto

- **Crítica:** datos de pacientes legibles o modificables por un tercero,
  o pérdida de historia clínica, alcanzable hoy sin precondiciones raras.
- **Alta:** lo mismo, con precondiciones reales pero plausibles.
- **Media:** dato que se muestra o guarda mal, o defensa ausente que otro
  control tapa.
- **Baja:** el resto.

## Criterio "bloquea el piloto"

Con pocas cosmetólogas conocidas como primeras usuarias, un hallazgo
bloquea si:

- (a) un tercero sin cuenta puede leer o escribir datos de pacientes;
- (b) se puede perder o corromper historia clínica en uso normal;
- (c) es un incumplimiento legal vigente desde la primera paciente real;
- (d) no hay vuelta atrás probada.

No bloquea: hardening contra atacantes dirigidos, self-XSS, mejoras de
producto, deuda, y lo que solo empeora con escala. La decisión final es
del dueño, por ítem, y queda escrita en la columna "Decisión" del índice.

## La regla del aislamiento entre clínicas

Ajuste del dueño al plan: el aislamiento entre clínicas **no podía quedar
como "sin verificar" ni como "parece que filtra bien"**. Si tras leer el
código quedaba cualquier duda sobre cualquier endpoint, se preparaba la
prueba dinámica real (dos cuentas, pasos exactos, qué escribe cada paso y
cómo se limpia) para que el dueño la autorizara. La respuesta tenía que
ser SÍ o NO, verificado.

Resultado: SÍ, verificado por lectura de cada acción autenticada y por
esquema. La única duda que la lectura no podía cerrar (si dos claves
únicas de la base incluían el identificador de la clínica) la cerró el
dueño consultando el esquema directamente. La prueba dinámica quedó como
no necesaria: no quedaba ninguna pregunta que respondiera mejor.

## Conteos

| Qué | Cuánto |
|---|---|
| Hallazgos de las tres tandas | 26 (4 + 14 + 8) |
| Registro consolidado (tandas + camino de los arreglos y del servidor + cruce legal) | 53, cada uno con gravedad, verificación, impacto, origen y decisión |
| Bloqueantes del piloto identificados | 6 |
| Inyección SQL | 100 % de las consultas parametrizadas |
| Aislamiento entre clínicas | SÍ, verificado |
| Secretos en el PHP | sí, varios; todos salieron del código y se rotaron (`HALLAZGOS-CERRADOS.md`) |

## El cruce contra el marco legal

Cuatro documentos legales (análisis de las dos leyes, demostración de la
firma, políticas) se escribieron **sin ver el repo**, en otro contexto, y
después se cruzaron contra el código real. Cada afirmación de los
documentos se buscó en el código: existe, no existe, existe distinto. Lo
que no coincide es un hallazgo más, marcado como técnico o como terreno de
la tanda 4.

## Las quince preguntas que el código no responde

Las responde el dueño o un abogado, no una tanda; la tanda 4 las refina.
Se listan como temas, sin estado:

1. Roles legales: responsable o encargado del tratamiento (25.326), y qué
   contrato tiene que existir con cada clínica. El proveedor del servidor
   encuadra como prestador de servicios (art. 25); contrato pendiente.
2. Dispositivo robado o PC compartida del local: qué queda expuesto.
3. Baja de una clínica: qué pasa con sus fichas, quién las conserva,
   portabilidad.
4. Supresión (25.326) frente a conservación de historia clínica (26.529).
5. Muerte del servidor: backups, dónde viven, si se probó restaurar,
   cuánto se tarda en volver y cuántas horas de datos se pierden.
6. Quién más puede leer la base: proveedor, CDN; cifrado en reposo; país
   del servidor.
7. Retención de fichas de una clínica dada de baja, con plazo escrito.
8. Respuesta escrita, en lenguaje humano, a la cosmetóloga que pregunta
   dónde están los datos de sus pacientes.
9. Registro de la base de datos ante la autoridad de aplicación.
10. Paciente que pide copia de su ficha: procedimiento, aunque sea manual.
11. Pacientes menores de edad: consentimiento del representante legal.
12. Plan de incidentes: qué se hace, a quién se avisa, si el registro de
    auditoría alcanza como evidencia.
13. Dependencia de una sola persona para operar el sistema.
14. Links de firma y formulario que viajan por mensajería de la
    cosmetóloga, con token incluido.
15. Retención de logs del servidor (IPs y tokens en URLs también son datos
    personales).

## Lecciones del camino

- **Un nombre publicado en los registros públicos de certificados recibe
  escaneos en minutos.** Un nombre temporal usado durante la migración
  fue recorrido por un bot el mismo día de emitir su certificado (todo
  403). Regla para todo nombre nuevo: nace con la protección puesta.
- **"Anotar, no arreglar" también vale para lo de una línea.** Un arreglo
  oportunista durante una tanda mezcla causas: la tanda deja de ser una
  foto del código tal como estaba.
- **El objeto del análisis es lo que corre, no lo que está en el repo.**
  Si difieren, eso es el primer hallazgo.
