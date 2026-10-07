# Método de la migración de servidor

Cómo se planificó, verificó y ejecutó la migración de Cosmetra a un
servidor nuevo (plan escrito el 2026-09-30; corte el 2026-10-06). Este
documento describe el método; no describe el servidor.

## Por qué se migró, y por qué así

El servidor contratado vencía y renovarlo costaba más del doble que
contratar uno nuevo con pago anual. Al mismo tiempo, la auditoría había
dejado varios hallazgos de arquitectura del servidor (pool compartido
entre entornos, usuario de base compartido, cuenta de despliegue sin
encierro, claves de acceso ajenas) que son mucho más fáciles de resolver
en una máquina vacía que operando en vivo.

**Servidor en paralelo, no mudanza en vivo.** Producción estaba vacía de
datos reales: no había nada que pudiera quedar partido entre dos
servidores. Rehacer pools, usuarios y SSH sobre un servidor vacío, con el
otro andando, es mucho menos riesgoso que tocar el que está en uso.

**No era una mudanza pura: eran tres cambios a la vez** (servidor nuevo,
arquitectura nueva y la promoción a producción del PHP sin secretos). Eso
va contra "una causa por vez", y el plan lo compensó de una sola forma:
**todo se verifica en el nuevo antes del corte, con el viejo intacto**, y
cada diferencia entre viejo y nuevo tiene que estar en una lista cerrada
de diferencias intencionales. Una diferencia que no esté en esa lista es
una regresión.

**El corte casi no depende de la propagación de DNS.** Los registros
pasan por el proxy del CDN: las visitantes siempre resuelven direcciones
del CDN, y cambiar el origen se aplica en segundos. El riesgo real del
corte no es la propagación: es un registro olvidado.

## La lista cerrada de diferencias intencionales

Toda comparación viejo/nuevo tiene que dar **igual, salvo** estas diez:

1. Pools y usuarios de PHP por sitio (uno por entorno, sin shell, con
   acceso solo a su webroot y su archivo de secretos).
2. Dueños y permisos de los archivos de secretos y de los webroots.
3. Usuarios y permisos de la base de datos: uno por entorno, permisos
   tabla por tabla, y la aplicación no puede modificar ni borrar el
   registro de auditoría.
4. SSH: cuenta de despliegue encerrada, sin contraseñas, claves
   autorizadas en un archivo central administrado por el dueño, lista de
   usuarios permitidos, sin certificados de terceros.
5. Auth básica también sobre la API del entorno de pruebas (antes cubría
   solo las páginas).
6. El PHP de producción: el del repo, sin secretos, en lugar del anterior.
7. Los valores de **todos** los secretos.
8. Memoria de intercambio, direcciones, nombre de host, claves de host
   SSH y puerto de SSH.
9. Archivos que no pasan: código muerto, un módulo de sabotaje huérfano y
   un repositorio dentro de un webroot.
10. Sin servidor de correo (el anterior tenía uno sin uso).

Todo lo demás se replica **igual, a propósito**: cron de rangos del CDN,
rotación de logs, renovación de certificados, cabeceras, límites de PHP,
zona horaria, juego de caracteres y la configuración de la base, incluidas
las deudas conocidas: se replican y no se arreglan acá, porque arreglarlas
sería una causa más.

## Las cinco capas de verificación

De la más barata a la más cara. Mismo criterio que la comparación entre
entornos antes de una promoción: no se confía, se compara.

| Capa | Qué se compara | Con qué | Tiene que dar |
|---|---|---|---|
| **A. Configuración** | Configuración completa del servidor web, del PHP (servicio y línea de comandos), variables de la base, zona horaria, versiones, timers y crons | Un script de inventario de solo lectura, corrido en los dos servidores, y un `diff` | Igual, salvo la lista cerrada |
| **B. Archivos** | Hash de **cada** archivo de los dos webroots | Manifiestos de hashes en los dos servidores, más la verificación por HTTP contra el repo | Pruebas nuevo = repo; producción nuevo = producción viejo, "ni de más ni de menos" |
| **C. Base** | Esquema y datos de las dos bases | Volcado del esquema de los dos lados + `diff`; conteo de filas y checksum por tabla, justo después de importar | Esquema idéntico, mismos conteos y checksums en las veinte tablas |
| **D. Comportamiento y visual** | La aplicación entera en el entorno de pruebas nuevo | Suite completa ×5 con 5 workers, **con las mismas capturas de referencia** (sin regenerar nada) | Los mismos números que la línea de base del viejo |
| **E. Lo nuevo anda y encierra** | Las propiedades que el viejo no tenía | Pruebas negativas, una por una | Todas pasan |

**Pruebas de la capa E** (las que no se pueden "ver", hay que intentarlas):
el pool de un entorno no puede leer el secreto del otro; el usuario de
base de un entorno no entra a la base del otro y ninguno puede crear
tablas; ninguno puede modificar ni borrar el registro de auditoría, y
después de la suite el log de errores no muestra ningún permiso denegado;
cada archivo de secretos pasa sus siete comprobaciones leído por el
usuario de su pool; desde la cuenta de despliegue no se ve nada fuera de
su encierro y no se puede escribir en el PHP; SSH con contraseña
rechazado; la API de pruebas sin credenciales devuelve 401 del servidor
web y nunca llega al PHP; un error del PHP llega al log del sitio
correcto y no al del otro; la base no escucha hacia afuera; después de un
**reinicio real**, todo vuelve solo; el backup corre de punta a punta
(cifrado, subida, bajada, descifrado y restauración a una base temporal).

Resultado de la parte de base: **17 pruebas, 17 bien** (5 positivas que
dan los conteos de la capa C, 12 rechazos verificados).

**Criterio para declarar la migración buena.** Antes del corte: capas A a
E completas, sin ninguna diferencia fuera de la lista cerrada, y el
rollback del corte ensayado. Después del corte: la suite ×5 contra el
entorno de pruebas (ya servido por el nuevo) en verde con los mismos
números, hashes verificados en el origen y por el borde, el log de acceso
del viejo en silencio, y el backup de esa noche en su destino y
restaurable.

## La línea de base y el congelamiento de código

**Línea de base** (2026-09-30, contra el servidor viejo): suite completa
×5 con 5 workers → 1.285 pasaron, 0 fallaron, 5 salteadas, 33,9 minutos.
Es el número contra el que se mide el nuevo: mismos 1.285 en verde,
mismas 5 salteadas (una corrida que saltee otra cantidad también es una
diferencia a explicar). El tiempo es referencia gruesa.

**Congelamiento de código** (regla del dueño): desde la línea de base
hasta cerrar la migración no se cambia nada de JavaScript, HTML, CSS ni
PHP, **y tampoco los tests, sus capturas ni la configuración de la
suite**: son el instrumento con el que se compara, y si cambian entre la
medición del viejo y la del nuevo, las dos mediciones no son comparables.

El motivo no es que un cambio "ensucie" la mudanza: es poder afirmar, si
algo falla al verificar el nuevo, **"el código es idéntico, entonces es la
migración"**. Con cambios en el medio, cada diferencia obliga a
preguntarse cuál de las dos cosas fue.

**Si igual surge algo urgente**, la comparación no se rompe, se mueve: el
arreglo se hace con su ciclo normal en el viejo, se vuelve a tomar la
línea de base, el nuevo recibe el mismo commit verificado por hash, y se
repiten solo las capas que dependen del código (B y D). Durante el corte
no se arregla nada; durante la observación el arreglo va al nuevo y el
viejo queda un commit atrás, anotado.

## Las fases

| Fase | Qué | Verificación | Vuelta atrás |
|---|---|---|---|
| **F0. Preparación** | Inventario del viejo con el script de solo lectura; exportar la zona DNS; línea de base de la suite; decisiones de diseño (16, numeradas, cada una con recomendación y decisión escrita) | Inventario y zona exportados; línea de base tomada | Nada que deshacer |
| **F1. Base del sistema** | Base del sistema endurecida antes de cargar secretos; zona horaria, memoria de intercambio, reinicio de prueba | Reinicio y todo vuelve | Reinstalar la imagen |
| **F2. Pila** | Servidor web, PHP, base, certificados, herramienta de backup; los dos pools; los dos sitios con nombres temporales | Capa A: `diff` de inventarios limpio salvo la lista cerrada | Ídem |
| **F3. Secretos y base** | Archivos de secretos con valores nuevos; usuarios de base; volcado de las bases del viejo → copia directa servidor a servidor → importación; cron de rangos del CDN | Siete comprobaciones por archivo de secretos; capa C; capa E de base | Borrar bases y usuarios del nuevo; el viejo no se tocó |
| **F4. Sitios** | Estáticos de producción copiados del viejo; PHP del repo instalado por el dueño; cuenta de despliegue encerrada; sincronización del entorno de pruebas desde el repo; registros DNS temporales con proxy; certificados emitidos por validación DNS | Capa B (22 de 22 estáticos por sitio, más el PHP); primeros chequeos de salud | Quitar los registros temporales |
| **F5. Verificación** | Capas D y E completas. **Ensayo del corte y del rollback** con los nombres temporales, midiendo cuánto tarda cada cambio en verse | Criterio "antes del corte" | No hay nada en producción todavía |
| **F6. Corte** (ventana de ~45 min) | Última suite verde en el viejo. En el viejo: apagar el cron de backup y **detener el PHP** (queda en pie pero mudo: lo que le llegue da un error visible en vez de escribir en la base equivocada). Volcado final → importación → capa C otra vez. Cambiar el origen de los registros DNS. Verificar origen y borde. Encender el backup en el nuevo. Certificados con renovación automática. Dar de baja los nombres temporales | Suite ×5 contra el entorno de pruebas; hashes por origen y borde; log del viejo en silencio; zona DNS sin la dirección vieja | Un paso: volver el origen de los registros al viejo y arrancar su PHP. Ensayado en F5 |
| **F7. Observación** (3 a 5 días) | El viejo encendido y mudo, como rollback. Cada día: backup en destino, logs de error del nuevo, log de acceso del viejo (silencio), cron de rangos, timer de certificados. **Ninguna usuaria real hasta apagar el viejo** | Días sin novedades | El mismo de F6 mientras el viejo exista |
| **F8. Apagar el viejo y cerrar** | Destruir en el viejo lo sensible (bases, secretos, claves, volcados); revocar su autorización de backup **y volver a autorizar la del nuevo el mismo día** (ver abajo); dar de baja; documentos de cierre; los tests que recién ahí se pueden escribir | Backup del nuevo subido después de la reautorización | Ya no hay |

**Quién hace qué:** todo lo que corre en un servidor lo hace el dueño; el
agente escribe scripts y plantillas, compara salidas, sincroniza el
entorno de pruebas con la cuenta de despliegue, corre la suite, verifica
por HTTP y mantiene los documentos. Los volcados viajan **de servidor a
servidor**, nunca por la PC, y siempre tirando desde el nuevo: ninguna
credencial del nuevo se guarda ni se usa en el viejo.

**Ningún valor del viejo viaja al nuevo:** secretos de la aplicación,
clave de cifrado del backup, autorización del almacenamiento remoto. La
clave anterior se conserva solo fuera del servidor, para poder leer los
backups anteriores al corte.

## Los 27 riesgos registrados

Título y mitigación, en genérico. Las probabilidades estimadas para este
servidor concreto no se publican.

| # | Qué puede salir mal | Cómo se detecta temprano | Vuelta atrás o mitigación |
|---|---|---|---|
| R1 | La base nueva arranca con una configuración por defecto distinta de la del viejo y la aplicación falla al guardar | Capa A (diff de la configuración); la suite falla en masa al guardar | Replicar la configuración del viejo antes de medir |
| R2 | Zona horaria distinta: vencimientos de firma, fechas y avisos corridos tres horas | Capa A; hora del sistema y de la base consultadas en los dos a la vez | Igualar antes de medir |
| R3 | El CDN sigue mandando tráfico al viejo por un registro olvidado | La zona exportada lista todos los registros; tras el corte el viejo da error visible en vez de atender, y su log lo muestra | Corregir el registro; no hay datos partidos porque el viejo no escribe |
| R4 | Propagación de DNS | Exportación de la zona (¿hay alguno sin proxy?); ensayo con tiempos medidos | Bajar el TTL un día antes si hubiera alguno sin proxy |
| R5 | Certificados: el nuevo no puede emitir por HTTP mientras los nombres apunten al viejo; el emitido a mano no se renueva solo; demasiados intentos agotan el cupo de la autoridad | Pedido a los nombres temporales da 200; ensayo de renovación en verde como condición de cierre; ensayo antes de cada emisión real | Emitir por validación DNS antes del corte y pasar a renovación automática después; certificado de origen del CDN como último recurso |
| R6 | El backup corre en los dos lados y escribe en la misma carpeta | Revisar el cron del nuevo al terminar F2: no tiene que estar | Encender en el nuevo recién en el corte; apagar en el viejo primero; ensayos a otra carpeta |
| R7 | El backup del nuevo no anda y nadie se entera | Ensayo con restauración; mirar el log la mañana siguiente | Volcado manual cifrado mientras se arregla |
| R8 | Quedarse afuera del nuevo al cerrar las contraseñas de SSH | Segunda sesión abierta; probar la clave desde una sesión nueva antes de cerrar | Consola de emergencia del panel, probada antes de tocar SSH |
| R9 | Diferencia de versión de PHP, base o servidor web | Capa A | Si es de parche y la suite pasa, se acepta y se anota; si no, fijar la versión |
| R10 | Correo | — | El dominio no tiene registros de correo y la aplicación no manda mails; no crear registros sin decidirlo |
| R11 | Los datos cambian entre la copia de ensayo y el corte | Capa C repetida en el corte | Volcado final con el PHP del viejo detenido |
| R12 | El dominio gratis del plan toca la delegación del dominio real o se renueva solo con costo | Consulta pública de los servidores de nombres después de contratar | No aceptar nada que diga "transferir" o "cambiar DNS"; desactivar renovaciones |
| R13 | El viejo vence antes de terminar, o se renueva solo al precio caro | Fecha exacta y estado de la renovación, en F0 | Fecha límite dura del corte; renovar un mes antes que cortar apurado |
| R14 | El plan nuevo no es equivalente (memoria, disco, centro de datos, imagen) | Preguntas al proveedor antes de pagar | No contratar sin las respuestas |
| R15 | Archivos sueltos en el viejo que no están en el repo y se pierden al apagar | Manifiesto de hashes de la capa B ("ni de más ni de menos"); inventario del home | Decidir uno por uno antes de F8 |
| R16 | La auth básica sobre la API de pruebas (lo único realmente nuevo) rompe algo: las páginas públicas, un cliente de la suite | Suite completa en F5 | Volver a la configuración anterior y dejar el hallazgo abierto |
| R17 | El encierro del pool bloquea algo que el PHP hacía | Suite y log de error del sitio | Agregar la ruta que falte |
| R18 | Caché del CDN sirviendo archivos viejos tras el corte | Hash por el borde después del corte | Purga por URL |
| R19 | Clave de host SSH y dirección nuevas: la sincronización avisa de host desconocido y alguien acepta una huella sin mirar | Anotar la huella desde la consola del panel y compararla en la primera conexión | Actualizar los hosts conocidos y las reglas |
| R20 | Secretos viejos que sobreviven en el disco del viejo y en las copias del proveedor | — | Todos los valores del nuevo son distintos; lo sensible se destruye antes de la baja |
| R21 | Un episodio transitorio conocido se repite durante la verificación y parece un fallo de la migración | La línea de base del mismo día y el patrón del episodio | Repetir la corrida; no cortar con una corrida dudosa |
| R22 | El backup del plan nuevo se da por bueno sin conocerlo | Las mismas preguntas que para el anterior | El backup propio sigue siendo el único probado |
| R23 | El servidor nuevo nace con claves de acceso del proveedor, o algo las repone después | Recuento de claves al recibirlo, al cerrar F1 y después del reinicio; el inventario lo informa | SSH solo lee el archivo central: aunque reaparezcan, no abren nada |
| R24 | Quitar esas claves rompe algo del panel del proveedor | **Cerrado** antes de empezar: el proveedor respondió por escrito que su panel no depende de ellas | La prueba de F1 del panel queda como confirmación, no como incógnita |
| R25 | Secretos del viejo ya expuestos a terceros | El registro del sistema muestra si alguna clave se usó en el período que cubre | Ningún valor del viejo pasa al nuevo; la autorización vieja se revoca en F8 |
| R26 | F1 arranca sin saber si la imagen trae esas claves | — | El diseño funciona igual con o sin ellas; el recuento del primer paso responde la pregunta en el propio servidor |
| R27 | Permisos tabla por tabla: una tabla sin su permiso hace fallar la aplicación | Consulta de "tablas sin permiso" (cero filas); suite; log de error | Dar el permiso que falte; a futuro, toda migración que cree una tabla trae su permiso |

## Lo que se aprendió en la ejecución

- **El CDN no cambia de origen de golpe:** en el ensayo del rollback,
  repartió entre los dos servidores durante unos minutos. Los comandos
  del corte lo contemplan (verificar varias veces, por origen y por
  borde, hasta que no quede ninguna respuesta del viejo).
- **Revocar la autorización del almacenamiento remoto corta también el
  backup del nuevo**, porque los dos usan la misma aplicación con la misma
  cuenta y la revocación es por aplicación, no por token. El paso de F8
  es, entero y el mismo día: revocar, volver a autorizar en el nuevo con
  alcance acotado, correr el backup a mano y **ver el archivo nuevo en el
  destino**. Es de los errores que se descubren tres días después, cuando
  ya no hay backups.
- **Una primera corrida distinta no se descarta ni se acepta: se repite.**
  Una corrida de verificación dio dos fallos por lentitud puntual sin
  causa en el servidor; se anotó como ruido con evidencia y la siguiente
  dio los números de la línea de base.
- **Nadie se entera si el backup falla.** Con la dependencia de la
  autorización, un aviso automático (o un chequeo externo que espere una
  señal por día) deja de ser una mejora cómoda.
