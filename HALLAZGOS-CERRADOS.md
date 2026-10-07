# Hallazgos cerrados

Solo los hallazgos **ya cerrados**, con su arreglo. Los abiertos no se
publican: publicar vulnerabilidades sin arreglar es un mapa para atacar.
Numeración propia de este documento; los identificadores internos no se
publican. Las descripciones son deliberadamente no operativas: qué era y
cómo se cerró, sin nombres de endpoints, parámetros, usuarios ni rutas.

Fecha de corte: 2026-10-07. Los hallazgos del servidor cerrados "con la
migración" rigen desde que producción se sirve desde el servidor nuevo
(2026-10-06).

## Tabla 1 — Auditoría (desde 2026-09-29)

| # | Gravedad | Qué era | Cómo se cerró | Cuándo |
|---|---|---|---|---|
| A-01 | crítica | La clave que firma los tokens de sesión estaba escrita en el código, con el mismo valor en el repo, en pruebas y en producción. Quien leyera el repo podía fabricar un token válido para cualquier clínica | Los secretos pasaron a un archivo fuera del repo y del webroot, con permisos 600, uno por entorno; el PHP **falla cerrado** si falta el archivo o un valor. Valor nuevo y distinto por entorno, generado por el dueño; nunca escrito en el repo, los tests, los documentos ni el chat. El repo dejó de tener ningún secreto adentro | Pruebas: 2026-09-29. Producción: 2026-10-06, con la migración |
| A-02 | alta | El token de un endpoint de mantenimiento estaba en el código, idéntico en los tres lugares, con un comentario "cambiá este valor en producción" nunca cumplido | Mismo arreglo que A-01. Test de regresión: el endpoint falla cerrado sin su secreto | Ídem |
| A-03 | media | Usuario y contraseña de la base de datos en el código; pruebas y producción compartían la misma credencial (y el repo traía además la de un hosting anterior) | Mismo archivo de secretos que A-01, con un usuario de base **propio por entorno** (ver A-08) | Pruebas: 2026-09-29/30. Producción: 2026-10-06 |
| A-04 | alta | La IP del cliente se tomaba de una cabecera que el cliente controla. Alimentaba los límites de intentos y quedaba como evidencia de quién firmó, en la base y en el PDF de la firma | La IP sale de la cabecera del CDN **solo cuando el pedido viene de un rango del CDN**; la lista de rangos la mantiene un cron diario; la cabecera genérica ya no se lee. Test rojo→verde 5/5 (el límite de login no se esquiva con cabeceras). Verificación manual del dueño: su login dejó su IP pública real | Pruebas: 2026-09-30. Producción: 2026-10-06 |
| A-05 | media | Pruebas y producción corrían en el mismo pool de PHP, con el mismo usuario de sistema: la separación entre entornos era por convención, no por permisos | En el servidor nuevo, un pool por sitio con usuario propio sin shell y `open_basedir` acotado a su webroot y su archivo de secretos. El PHP de pruebas no puede leer nada de producción | 2026-10-06, con la migración |
| A-06 | media | Un error de edición en la configuración del pool compartido tumbó pruebas y producción a la vez durante 8 minutos: A-05 demostrado en la práctica | Mismo arreglo que A-05. Regla permanente: `--test` de la configuración antes de cada recarga, porque el servicio sigue siendo uno solo | Ídem |
| A-07 | alta | La cuenta de despliegue (solo subida, sin shell) no estaba encerrada: podía leer por SFTP el PHP de producción y, con él, sus secretos | Mitigación inmediata por permisos del archivo. En el servidor nuevo, la cuenta nace **encerrada** (chroot) en el webroot de pruebas, con escritura solo en estáticos y nunca en el PHP. Verificado: desde esa cuenta no se ve nada fuera | Mitigado 2026-09-30; cerrado 2026-10-06 |
| A-08 | media | Contraseña de base corta, compartida entre entornos y expuesta en el registro de una sesión de trabajo | Un usuario de base por entorno, con permisos **tabla por tabla** y solo lectura/escritura de filas (sin crear ni borrar tablas); sobre el registro de auditoría, solo insertar y leer. El usuario compartido no existe en el servidor nuevo. 17 pruebas de permisos: 5 positivas y 12 rechazos verificados | 2026-10-05/06 |
| A-09 | media | Los errores del PHP se descartaban: no aparecían en ningún log visible | Configuración del pool: los errores van al log de cada sitio | 2026-09-29 |
| A-10 | baja (tests) | Carrera entre el `fill()` de Playwright y un foco diferido de la aplicación: el texto del test caía en otro campo (fallos 1/5 en dos lugares) | Los tests esperan el foco antes de escribir. Regla en `CLAUDE.md` | 2026-09-30 |
| A-11 | baja (visible) | El dominio de la marca daba error 526 (y la página de bienvenida del servidor web por HTTP) | Redirección 302 desde el CDN al login de producción, verificada desde afuera | 2026-09-30 |

## Tabla 2 — Refactor (cerrados y en producción desde 2026-09-29)

Seis arreglos de seguridad que viajaron dentro de los mismos archivos del
refactor de módulos. Cada uno con su test de regresión (payload con
`<img onerror>` y `& < > " '`, verificado que falla 5/5 sin el arreglo) y
verificado en producción con un **control positivo** antes de promover
(`METODO-PROMOCION.md`).

| # | Tipo | Qué era | Cómo se cerró |
|---|---|---|---|
| R-01 | XSS almacenado | El nombre de la paciente se insertaba sin escapar en los avisos de Notificaciones | Escapado en el render; test con payload |
| R-02 | XSS almacenado | El nombre se insertaba sin escapar en el modal de envío de la firma | Ídem |
| R-03 | XSS almacenado | La imagen de la firma se insertaba sin escapar en el `src` del PDF de la firma; un valor malicioso guardado agregaba atributos al `<img>` | Escapado; el `<img>` del PDF queda con exactamente sus dos atributos esperados |
| R-04 | XSS almacenado | La fecha de nacimiento se insertaba sin escapar en la tarjeta de un envío pendiente de aprobar | Escapado; test con payload |
| R-05 | XSS almacenado | Cinco etiquetas de valores libres de la tarjeta de la ficha se insertaban sin escapar | Escapado; test con payload para las cinco |
| R-06 | Integridad de datos | Celdas del CSV exportado que una planilla evalúa como fórmula (`= + - @` al inicio) | Prefijo neutralizador, salvo números con signo y dígitos separados por espacio o guion, que se dejan como están; 27 tests de Exportación |
