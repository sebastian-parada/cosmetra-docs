# CLAUDE.md — Reglas de trabajo de Cosmetra

> **Versión pública, saneada.** Este es el archivo de reglas que lee el
> agente de IA al empezar cada sesión de trabajo sobre Cosmetra, tal como
> estaba el 2026-10-07 (558 líneas en su versión interna; 31 revisiones
> desde el 2026-09-16, listadas en `CHANGELOG-REGLAS.md`). Para publicarlo
> se reemplazaron direcciones, puertos, cuentas, rutas, claves y comandos
> de acceso por descripciones genéricas, y se quitó entera la sección de
> credenciales de prueba. Las etiquetas **`[lección]`** marcan las reglas
> que nacieron de un incidente real, con el incidente contado en el propio
> texto: son 15. Convención: **`app`** es producción, **`code`** es el
> entorno de pruebas donde trabaja el agente.

Leé este archivo completo antes de tocar código. Estas reglas no son sugerencias.

---

## 1. Qué es esto

Cosmetra es un SaaS de fichas clínicas para cosmetólogas, en producción real. Las fichas son historia clínica (Ley 26.529) y contienen datos de salud, la categoría más protegida de la Ley 25.326.

El refactor de módulos cerró el 2026-09-26 y se promovió a producción el 2026-09-29 (detalle en `HISTORIA.md`). **La etapa actual es la auditoría completa y los arreglos previos al piloto**: el plan, los hallazgos y las decisiones viven en `AUDITORIA.md`.

El dueño del proyecto es Sebastián. Su prioridad número uno sigue siendo que **la interfaz no se rompa visualmente**: el comportamiento visual se mantiene, y los cambios los pide él. Un arreglo técnicamente impecable que mueve un margen tres píxeles es un arreglo fallido.

---

## 2. Dónde trabajás y cómo se ve el resultado

**Trabajás en la PC de Sebastián**, en el repo local del refactor. Existe otra carpeta con la copia de referencia original, que **no se toca ni se lee de ahí**: solo existe como respaldo de Sebastián.

En el repo están el código, los tests (`tests/`) y el historial de git. Editás ahí.

Para ver el resultado en el navegador, **sincronizás al servidor** con un único comando de copia (`js`, `css` y los `.html`) hacia el webroot de `code`, usando una **cuenta de despliegue dedicada: solo subida por SFTP, sin shell, con permiso de escritura únicamente sobre los estáticos de `code`** y encerrada en ese webroot. La clave de esa cuenta vive fuera de cualquier carpeta sincronizada a la nube.

El resultado se ve en la URL de `code`, detrás de auth básica.

### Qué NUNCA tocás

- **`app`** en el servidor: es producción (sin excepciones vigentes; la excepción "`app` vacía" está cerrada, ver justo debajo).
- **El script de promoción a producción**: promover lo hace Sebastián a mano, después de revisar.
- **El `api/index.php` del servidor**: cada entorno tiene su propia configuración (ver abajo).

### Excepción "`app` vacía": CERRADA (2026-09-29)

La excepción (autorizada el 2026-09-26) cumplió su propósito: comparación
`code` vs `app`, promoción y suite de aceptación (ver `HISTORIA.md`). Desde
ahora **no se corre nada contra `app` sin una autorización nueva y
explícita de Sebastián, aunque siga vacía**. Queda vigente lo de siempre:
leer archivos estáticos públicos está permitido; subir, borrar, SSH o SQL
contra `app`, jamás.

### `api/index.php`: el del repo ES el que corre en `code` (desde el paso 0)

Desde el paso 0 del paquete de arreglos críticos (2026-09-29),
`api/index.php` **no tiene ningún secreto adentro**: carga los secretos
de la aplicación (las credenciales de la base y el resto) desde **un
archivo por entorno, fuera del repo y del webroot**, legible solo por el
usuario del PHP (permisos 600; el formato, sin valores, está en una
plantilla del repo) y **falla cerrado** si falta el archivo o cualquier
valor. Por eso el archivo del repo es, byte a byte, el que corre en `code`
(verificado por hash al instalarlo). Reglas:

1. **Los cambios al PHP se hacen sobre `api/index.php` del repo**, con el
   ciclo de la sección 4; lo instala Sebastián en `code` y se verifica por
   hash que repo y servidor coinciden.
2. **A `app` solo por promoción** (sección 13).
3. Las copias históricas del PHP anterior quedan fuera del repo, como
   referencia de cómo era antes del paso 0.
4. Los valores de los secretos los genera y los pone Sebastián; nunca se
   escriben en el repo, en los tests, en los documentos ni en el chat.

### Datos de pacientes reales

Desde el primer usuario real: **nunca** traer datos de pacientes reales a
esta máquina, a la carpeta compartida, a capturas de pantalla, a logs de
tests ni al chat. Los tests no corren nunca contra `app`. Si un dato real
aparece por accidente en un archivo local o en una salida, se avisa a
Sebastián y se borra de inmediato, registrando qué era y dónde estuvo.

---

## 3. Las tres reglas que no se rompen

### 3.1 El CSS está congelado

`css/main.css` **no se modifica**. Ni una línea, ni un valor, ni un espacio.

Si mientras migrás un módulo parecería que hace falta tocar el CSS: **frená y preguntá**. No lo toques "solo un poquito".

### 3.2 El HTML estático está congelado

`fichas-cosmetologia.html` y los demás `.html` **no se modifican**. Incluye los `onclick="..."` inline, que por ahora se quedan: los módulos siguen exponiendo sus funciones en `window` como hoy.

(Anotado para una segunda pasada: reemplazar los `onclick` inline por listeners con `data-*`. No es parte de este refactor.)

### 3.3 El comportamiento visual se mantiene (restricción no negociable)

El HTML que generan las funciones de render (`renderFichaCard`, `renderTabla`, `renderTodos`, etc.) se preserva idéntico — mismas clases, mismos `id`, misma estructura, mismos atributos —, **salvo el cambio puntual que un arreglo autorizado por Sebastián requiera**. En ese caso: el cambio exacto al HTML se describe en el plan del arreglo antes de tocarlo, se verifica con tests y capturas, y todo lo demás sigue idéntico. Si una captura cambia donde el plan no lo predijo, es una regresión: frenar y avisar.

Si cambia una clase, cambia el CSS que se le aplica, y ahí se rompe lo visual.

La regla de fondo (2026-09-29): a Sebastián le gusta su UX/UI actual; puede haber mejoras, pero **no se cambia ni se daña lo que existe, y los cambios los pide él**.

---

## 4. Cómo se trabaja: una fase por vez

- **Una fase por vez**, y ninguna arranca sin confirmación de Sebastián
  (sección 8). Ejemplos de fase: una tanda de auditoría, un arreglo autorizado,
  la segunda pasada `onclick`→`data-*`.
- **El plan de una fase con cambios de código va en `PLAN-<fase>.md`** antes de
  tocar nada, con las mismas reglas de siempre: uno solo a la vez, se copia a
  la carpeta compartida al actualizarse, y al cerrar la fase el resumen pasa a
  `AUDITORIA.md` y el plan se borra. (Una tanda de auditoría no necesita plan
  propio: su prompt ya está aprobado en `PROMPTS-AUDITORIA.md`.)
- **Tests primero:** antes de cambiar un comportamiento, se escribe el test que
  fija el comportamiento nuevo y se verifica que **falla** contra el código
  actual — el equivalente del "tests contra el código viejo" del refactor,
  invertido para arreglos. El ciclo sigue siendo: test → cambio → sincronizar →
  suite completa `--workers=5 --repeat-each=5` → commit.

### Nunca

- Dos fases (o dos arreglos) a la vez
- Seguir a la fase siguiente con tests fallando
- Commitear con tests en rojo

### Cambios a `api/index.php`

El PHP no tiene ningún test propio y toca la base. Todo cambio sigue este
ciclo, sin excepciones:

1. **Test primero:** un test e2e dirigido (Playwright contra `code`) que fije
   el comportamiento esperado y **falle** contra el PHP actual — la suite es
   hoy el único arnés que ejercita la API. Commit del test aparte.
2. **Backup + hash** del `api/index.php` del servidor de `code`, antes de tocar.
3. **El cambio en `code` lo sube Sebastián, a mano** (decidido 2026-09-29). La
   cuenta de despliegue no se usa para escribir en `api/`: agregaría riesgo sin
   necesidad.
4. Suite completa `--workers=5 --repeat-each=5` contra `code`.
5. Verificación por hash después, y rollback = restaurar el backup (el
   mecanismo se ensaya antes del primer cambio real, como en la promoción).
6. A `app` solo por el procedimiento de promoción (sección 13). **Un cambio de
   PHP nunca comparte subida ni commit con uno de JS.**

Precisiones al ciclo (aprobadas por Sebastián el 2026-09-29, al planear el
paquete de arreglos críticos):

- **Dónde viven los secretos (decidido):** los secretos de la aplicación
  (las credenciales de la base y el resto) viven en un archivo aparte en el
  servidor, **fuera del repo y fuera del webroot**, con permisos 600 (mismo
  patrón que la clave del backup), uno por entorno. El PHP lo carga y **falla cerrado** si falta o
  si falta un valor (error genérico al navegador, detalle al log; nunca un
  valor por defecto embebido). Los valores los genera y los pone
  Sebastián: nunca se escriben en el repo, en los tests, en los documentos
  ni en el chat.
- **Arreglos sin rojo→verde observable** (rotar un secreto, mover
  configuración): no se inventa un test artificial. Se cierran con (a) un
  test o la suite que confirme que no se rompió lo que andaba, y (b) una
  verificación manual de Sebastián, descrita en el plan y anotada en el
  mensaje del commit.
- **Un arreglo que cruza PHP y cliente:** commits separados, pero las dos
  subidas van en la **misma ventana** (deploys consecutivos, suite completa
  recién después de ambos), en el orden que deje `code` en un estado no roto
  si alguien entra en el medio. El plan del arreglo dice ese orden y por qué.
- **Limpieza de lo que escriben los tests nuevos:** un test que escribe en una
  tabla sin teardown propio (por ejemplo la de intentos de login) define su
  limpieza **antes** de correrse: datos únicos por corrida que el propio
  sistema purga (y cuándo), o un `DELETE` puntual que corre Sebastián, con el
  criterio anotado en el plan. Nunca se deja una tabla creciendo sin plan.

### Cambios de base de datos

Cambios de esquema o de datos por SQL (una migración, un UPDATE masivo,
endurecer la configuración): el script se escribe y se revisa **antes**, corre en
transacción cuando el motor lo permita, con backup previo verificado, **y lo
ejecuta Sebastián** — el asistente nunca corre SQL contra ningún entorno, ni
siquiera de lectura, sin su autorización explícita para ese comando.

---

## 5. Tests

```
npx playwright test --workers=5
```

Corren contra `code`. Las credenciales de la auth básica no están en este
documento.

**Importante:** los tests prueban lo que está en el servidor, no lo que está en la carpeta local. Sincronizá antes de correrlos o vas a estar probando código viejo.

**En paralelo, con una cuenta de prueba por worker (desde el 2026-09-30, paso 0e).** Cada worker corre contra su propia cuenta (la configuración de los tests la elige por el índice del worker): la 0 es la original y las otras cuatro viven en un archivo de credenciales ignorado por git. Reglas:

- **`[lección]` Nunca más workers que cuentas (5).** El setup global corta la corrida si pasa, y también si alguna cuenta no está en base (0 fichas, 0 pendientes, config base) después de limpiarla. Dos workers sobre la misma cuenta dan contención y fallos que no son reales — por eso existió el "siempre `--workers=1`" hasta esta fecha.
- **Sin `--workers`, el default es 1** (`playwright.config.js`) y todo corre contra la cuenta original, como siempre. Sirve para correr un test suelto o depurar.
- **Sin `fullyParallel`:** Playwright reparte por archivo; con `--repeat-each=5` son 10 bloques grandes, 2 por worker, cada uno con el orden interno de siempre.
- **Tiempos medidos (2026-09-30):** suite completa ×5 con 5 workers, ~32 min (tres corridas: 32,6 / 32,4 / 30,4); en serie eran 2,4 h, más que el límite de 2 h de una tarea en segundo plano del asistente.
- **Un fallo que aparece siempre en el mismo worker** es estado de esa cuenta, no ruido: mirar esa cuenta antes de sospechar del código.
- **El servidor no es el cuello** (medido: 97 % idle con 5 workers). Desde la migración (2026-10-06) cada sitio tiene su propio pool de PHP, pero CPU y base siguen compartidas: cuando `app` tenga usuarias, la suite no se corre en horario de uso.
- **`[lección]` Un test que escribe en un campo tiene que esperar primero cualquier foco diferido de la app** (`setTimeout(... focus() ...)`): `fill()` inserta el texto en lo que tenga el foco, y si la app mueve el foco en el medio, el texto cae en otro campo (pasó en Notas y en el DNI de la página de firma, 1/5 cada vez). `await expect(campo).toBeFocused()` antes de escribir.

### `[lección]` Antes de declarar "verde": repetir 5 veces

Una corrida limpia no alcanza. Antes de commitear o de avisar que una fase está lista:

```
npx playwright test --workers=5 --repeat-each=5
```

**Por qué es obligatorio:** en el módulo Notas, una sola corrida daba resultados distintos cada vez (26/33, 29/33, 30/33) y eso hizo perder horas discutiendo si era un bug real o ruido del entorno. Con `--repeat-each=5` el patrón fue inmediato: un test fallaba 5 de 5 (bug real, resultó ser una carrera en el login) y otros fallaban 1 de 5 de forma dispersa (ruido). **Un test que falla 5/5 o 4/5 es un bug. Uno que falla 1/5 en tests distintos cada vez es ruido de entorno.**

No reportes "está en verde" con una sola corrida.

### Sabotaje: verificar que un test detecta lo que dice detectar

Cuando un test existe específicamente para blindar un punto de acoplamiento riesgoso (una variable compartida que podría moverse mal, como fueron `tablaFiltro`, `configPanelOpen` o `_prevPage` en el refactor), no alcanza con que el test pase contra el código bueno — hay que probar que también *falla* contra una versión rota a propósito, antes de confiar en él.

1. Escribí el/los test(s) contra el código sin tocar, con una predicción escrita de qué falla y por qué si el punto de acoplamiento se rompe.
2. Confirmá `--repeat-each=5` en verde contra el código sin tocar.
3. Aplicá el sabotaje: la forma "intuitiva pero incorrecta" de mover eso (normalmente un módulo temporal `js/modules/_sabotaje_<algo>.js`, agregado a `modulesReady`). Sincronizá y verificá por hash.
4. Confirmá que el test falla como predijiste (y que cualquier test de control sigue en verde, si corresponde).
5. Revertí (`git checkout HEAD -- <archivo>`), **borrá también el/los archivo(s) nuevos que el sabotaje haya creado** (`rm`) — un módulo temporal no estaba en el commit anterior, así que `git checkout` no lo toca.
6. Sincronizá el revert.

**`[lección]` La sincronización no propaga borrados — no alcanza con no incluir el archivo la próxima vez.** El comando de copia de siempre sube lo que existe localmente, pero nunca borra en el servidor lo que ya no existe en local. Un archivo temporal de sabotaje (o cualquier archivo nuevo creado para una prueba) que se borra solo del lado local queda huérfano en el servidor para siempre si no se borra ahí también (un módulo de sabotaje huérfano apareció en el inventario del servidor semanas después). Después de revertir un sabotaje que creó un archivo nuevo: borrarlo también en el servidor por SFTP, y verificar con un pedido de lectura que da 404, no solo que el `rm` no tiró error.

7. Re-verificá por hash que el archivo real (`legacy.js`, etc.) volvió a coincidir con `master`.
8. Confirmá `--repeat-each=5` en verde de nuevo.

**`[lección]` "Hay un test que pasa por ahí" no es lo mismo que "un test que fallaría si se rompe".** Antes de dar por cubierto un acoplamiento porque algún test lo ejercita, revisá qué verifica exactamente ese test — puede parar antes de llegar al punto que te importa. Escribí el test que falte contra el código sin tocar, no después. Evidencia completa (el sabotaje de Notificaciones que lo mostró: 2 tests en verde con `notifyConfigChange` roto) en `HISTORIA.md`, "Tercera nota de método" (Progreso).

### Capturas visuales: la red de seguridad de lo que más importa

Los tests de comportamiento no detectan que un margen cambió, que un botón se movió, o que una sección quedó desalineada. Y eso es justo la prioridad número uno de Sebastián.

Por eso hay capturas de referencia con `toHaveScreenshot()` en `tests/visual.spec.js`. Funcionan así:

```
npx playwright test --workers=5                    # compara contra las referencias
npx playwright test --workers=1 --update-snapshots # actualiza las referencias (SOLO con autorización)
```

**Regla dura: nunca corras `--update-snapshots` por tu cuenta.** Si una captura no coincide, eso es exactamente la señal que el sistema existe para dar. Frená y avisale a Sebastián con la imagen del diff que genera Playwright. Actualizar la referencia sin que él lo apruebe es borrar la evidencia de una regresión visual.

**Y cuando esté autorizado: siempre con `--workers=1` y la cuenta original** (decidido 2026-09-30), acotado con `-g` a las capturas autorizadas. Una referencia se genera en una sola cuenta y en serie, nunca en paralelo: así todas las referencias salen del mismo lugar. Un dato propio de la cuenta que llegue a una captura (hoy solo el link de "Solicitar Ficha") se tapa con `mask`; no se normaliza el DOM antes de la foto.

Las capturas cubren cada pantalla en desktop y mobile, modo claro y oscuro. Antes de una fase que toque una pantalla, verificá que existan las capturas de esa pantalla; si faltan, creálas primero contra el código actual sin tocar — es la verdad de cómo tiene que verse.

**`[lección]` Una captura de un modal tiene que cerrarlo después de la foto.** Si el
test termina con el modal abierto, el backdrop tapa el resto de la
página y bloquea los clicks de lo que corra después en la misma sesión
de navegador — en particular el `afterEach` que vuelve a modo oscuro
clickeando la sidebar, que se cuelga hasta hacer timeout. Encontrado
migrando Ficha completa (2026-09-23), con `--repeat-each=5`: las
capturas de "modal de ficha duplicada" y "modal de borrado" fallaban
5/5 en el describe de al lado por este motivo. Después de
`toHaveScreenshot()`, un click al botón de cerrar/cancelar del modal
(`closeWarnModal()`/`closeModal()`, según el modal) antes del `finally`
de teardown.

**`[lección]` Una captura con backdrop (modal, overlay) no ve lo que hay debajo.** Cubre cómo se ve el modal en sí — centrado, botones, tema —, pero no lo que el modal deja o cambia debajo (qué se cerró, qué quedó abierto, qué estado se tocó): eso necesita un test de comportamiento aparte. "Hay una captura del modal" no es una respuesta para ese efecto colateral. Al planear una fase que toque modales: para cada efecto de abrir/cerrar uno, decidí qué test lo verifica. Evidencia completa (el sabotaje de Exportación donde una captura pasó 5/5 con el menú "Más" abierto detrás del modal) en `HISTORIA.md`, "Progreso", cuarta nota de método.

**Capturas con datos reales, no solo pantallas vacías.** Las capturas de pantallas vacías (sin fichas, sin notas, sin sesiones) no alcanzan para pantallas donde el estado con datos se ve distinto del estado vacío — Sesiones, Panel de Control con fichas cargadas, Ver Todos con resultados, etc. Antes de dar por cerrada cualquier fase que toque una pantalla con un estado "con datos" visualmente distinto de su estado vacío: agregá a `visual.spec.js` al menos una captura de ese estado con datos reales (creando una ficha de prueba si hace falta, con el mismo patrón de teardown inmediato). No alcanza con la captura de la pantalla vacía sola.

**`[lección]` Al regenerar capturas por un fix, regenerá todas las del mismo grupo, no solo la que falló.** Encontrado migrando Firmas (2026-09-24): una captura del Panel de Control con firma había quedado de una corrida anterior a un fix de un bug de instrumentación (id de ficha cruzado), con la firma aplicada a la ficha equivocada — y pasó todas las comparaciones siguientes sin fallar, porque el cambio real (2 badges + 2 porcentajes) quedaba debajo del umbral de `maxDiffPixelRatio: 0.01` (ver "Encontrado y no arreglado" en `ESTADO.md`, confirmado con este caso real). Sus 5 capturas hermanas sí se habían regenerado en una corrida posterior y estaban bien; esta había quedado atrás sin que ningún test lo mostrara. Si una captura de un grupo (mismo tipo, distintas variantes: desktop oscuro/claro, mobile oscuro) necesita regenerarse por un fix, regenerá las demás del mismo grupo en la misma corrida — no hay forma de confiar en que "esta no cambió" sin volver a generarla, porque una referencia vieja y mal generada puede seguir pasando en verde indefinidamente.

### Cómo le pasás las capturas a Sebastián para revisar

Cuando haya capturas pendientes de su visto bueno: vaciá la carpeta de revisión en la carpeta compartida y copiá ahí **solo** las capturas pendientes de esa revisión (los PNG de `tests/visual.spec.js-snapshots/` que correspondan). Nunca copies la carpeta de snapshots entera — solo lo que está pendiente en ese momento. El repo (`tests/visual.spec.js-snapshots/`) sigue siendo la fuente de verdad; esa carpeta compartida es nada más para que Sebastián pueda mirarlas sin que se las subas a mano.

### Simular la respuesta del servidor con `page.route` (patrón nuevo, 2026-09-25)

Para probar cómo **el cliente** pinta datos que un test no puede producir de verdad, o que dependen del reloj o de un estado que cambia solo, se simula en el navegador la respuesta de un endpoint con `page.route` en vez de sembrar datos reales. Primer uso: los avisos de Notificaciones — un aviso de embarazo necesita 9 meses, un link de firma vence 24 h después de pedirlo, y el texto de un aviso de ausencia cambia con el tiempo ("6 meses" → "1 año"): nada de eso se puede fijar con datos reales, y los avisos de firma vencida fueron lo que más problemas dio en todo el refactor.

- **Helper:** `simularAvisos(page, avisos)` en `tests/cosmetra.spec.js`, con la misma forma de respuesta que devuelve la API real. Un endpoint nuevo se simula con un helper hermano, no con un `route` suelto en cada test.
- **Llamarlo antes de `login(page)`**: el login también pide la lista y una respuesta real ya habría pintado el globo.
- **Si el test puede disparar una escritura** (descartar, borrar) sobre un dato simulado, simular también esa acción: si no, se escribe en la base real con ids falsos y quedan filas que no limpia ningún teardown.
- **Qué prueba y qué no:** prueba el código del cliente (render, títulos, globos, handlers). **No** prueba que el servidor genere bien los datos — eso es PHP y tiene su propio ciclo (sección 4) —, así que cada área que use el patrón mantiene además una prueba de punta a punta con datos reales (Notificaciones: el humo de "una ficha con una sesión antigua").
- **En capturas:** una captura con datos simulados **no** reemplaza sin más a la de "datos reales" que pide la regla de arriba. Cada plan de fase tiene que decir explícitamente qué se simula y por qué, y Sebastián lo aprueba (Notificaciones: aprobado el 2026-09-25).

### Estado de cobertura

Tests de humo + profundidad de los 12 módulos: ~250 tests únicos entre `cosmetra.spec.js`, `visual.spec.js` y `login.spec.js`, con ~80 capturas de referencia, de las cuales 77 corren en la suite normal (las otras 3 solo en contextos particulares). Corren siempre, después de cada cambio.

Lo que sigue **sin ningún test** está listado en `HALLAZGOS.md` (entrada de cobertura, 2026-09-26); la cobertura medida con V8 queda para la tanda 6 de la auditoría. **El test que le falte a un área se escribe cuando Sebastián autoriza un arreglo en esa área**, como primer paso del ciclo (sección 4), no antes.

---

## 6. Arquitectura: cómo quedó

### Estado compartido: cómo quedó de verdad

`js/store.js` tiene solo dos claves vivas (`tablaFiltro`, `todosUltimasSesiones`) y tres muertas ya anotadas; **no hay ningún `store.subscribe()` en el repo**. El resto del estado compartido son `let`/`const` de nivel superior de `legacy.js`, bare-visibles para los módulos ES (patrón verificado con prueba real).

**La idea original del store central con suscripciones no se descartó: está PENDIENTE.** El estado compartido disperso en `legacy.js` es deuda técnica anotada, no una decisión tomada. Si algún día se retoma el store central, es una fase nueva con plan propio.

Vanilla JS puro, sin librerías ni dependencias nuevas.

### Módulos

Cada módulo es dueño de su pantalla: su estado propio, su render, sus handlers. El estado compartido con el resto pasa por los `let`/`const` de `legacy.js` o por `store.js` (las dos claves de arriba).

Cada módulo expone en `window` lo que el HTML llama por `onclick` (regla 3.2) y lo que el núcleo u otros módulos llaman bare — el detalle por módulo está en `HISTORIA.md`.

### Cómo cargan los módulos hoy

Como `fichas-cosmetologia.html` solo carga `js/legacy.js` (sin `type="module"`, sin bundler), los módulos se cargan con `import()` dinámico desde dentro de `legacy.js`. Si el `import()` falla, no se muestra sidebar/main (quedan con el `display:none` que ya tienen en el HTML crudo) y se muestra un error persistente en `#login-err` en vez de dejar la app muda.

**Orden de ejecución garantizado:** `#sidebar` y `#main` tienen `display:none` escrito directamente en el HTML (no por JS), y los botones que navegan a cada página están anidados dentro de esos dos elementos. Por eso no hay forma de que el usuario dispare una función de un módulo antes de que `await modulesReady` se resuelva.

---

## 7. Criterio de la etapa: auditar sin tocar, arreglar con autorización

- **Durante una tanda de auditoría rige "anotar, no arreglar", tal cual:** solo
  lectura, hallazgos a `HALLAZGOS-AUDITORIA.md` con su formato, cero arreglos
  oportunistas aunque sean de una línea.
- **Fuera de las tandas, arreglar es el objetivo de la etapa**, con estas
  reglas:
  1. Cada arreglo lo autoriza Sebastián por ítem — la columna "Decisión" del
     índice de `HALLAZGOS-AUDITORIA.md` es la fuente de esa autorización.
  2. Un commit por arreglo, nunca mezclado con otro arreglo ni con tests de
     otro tema.
  3. Antes de tocar, confirmar si es preexistente y escribir el test que fija
     el comportamiento correcto (falla con el código actual, pasa con el
     arreglo).
  4. El comportamiento visual no cambia salvo lo que el plan del arreglo
     describe (regla 3.3).
  5. Lo que aparezca en el camino y no sea el arreglo autorizado, se anota en
     `HALLAZGOS-AUDITORIA.md` y se sigue.
- **Código muerto:** mismo criterio de siempre — se puede borrar solo con la
  evidencia completa de "sin caller" (buscar el nombre en todo el repo,
  incluyendo `onclick` inline del HTML, `onclick` generados en strings,
  accesos dinámicos `window[...]` y las páginas que no cargan `legacy.js`),
  el OK de Sebastián, y en commit aparte. Ante la duda, no se borra: se anota.
- **Tests y herramientas de verificación:** igual que siempre — cualquier
  cambio a tests necesita autorización explícita y va en commit aparte; se
  puede arreglar solo algo sencillo (chico, localizado y que no cambia qué se
  verifica). **Nunca cuenta como sencillo** aflojar o borrar una aserción,
  subir timeouts para que algo pase, regenerar capturas de referencia o
  saltear un test: eso siempre necesita autorización con la evidencia de por
  qué.

---

## 8. Cuándo frenar y avisarle a Sebastián

**Frená y preguntá cuando:**

- Parecería que hace falta tocar CSS o HTML
- Hay una decisión de producto de por medio (un comportamiento raro que no sabés si es bug o intencional)
- Encontraste algo urgente o crítico (ver sección 7)
- Terminaste un arreglo que cambia algo visible y necesita revisión visual
- Algo no está claro y estarías adivinando
- Vas a arrancar una fase de trabajo nueva (ver "Confirmar antes de arrancar una fase nueva" abajo)
- Necesitás cortar un proceso con `taskkill` o similar (ver "Cortar procesos" abajo)

**Pedile revisión visual solo cuando los tests no alcanzan.** Posición de elementos, espaciados, cómo se siente una animación, comportamiento en mobile. Eso ningún test lo detecta.

**No lo molestes por:** cosas menores, avances rutinarios, confirmaciones de lo que ya está acordado acá. Si cada cambio necesita su visto bueno, el trabajo no avanza.

**Mientras corre una tanda de auditoría:** solo lectura; nada que escriba en `code` sin autorización previa; nada contra `app`; cero arreglos aunque sean de una línea; hallazgos a `HALLAZGOS-AUDITORIA.md` con su formato; y un crítico explotable hoy por un tercero sin cuenta **corta la tanda**: se avisa de inmediato. (El detalle vive en `AUDITORIA.md`, "Reglas comunes" — esta línea existe para que ninguna sesión lo pierda de vista.)

### Confirmar antes de arrancar una fase nueva

Antes de empezar una fase de trabajo nueva que no fue pedida explícitamente en el mensaje de esta sesión —aunque `ESTADO.md` diga que es "el siguiente paso"— frená y confirmá con Sebastián, incluso si técnicamente es lo que sigue según el plan.

Ejemplos de "fase nueva": arrancar una tanda de auditoría, empezar un arreglo (aunque ya esté autorizado en la columna Decisión), tocar cualquier cosa que no sea continuación directa de lo que Sebastián pidió en su último mensaje.

No hace falta confirmar entre los pasos internos del ciclo de una fase ya en curso (test → cambio → sincronizar → suite completa → commit, sección 4) — eso sigue corriendo solo, como hasta ahora.

**Al arrancar en frío una sesión nueva:** después de leer `CLAUDE.md` y `AUDITORIA.md`, decile a Sebastián en una frase dónde quedó todo y qué harías como siguiente paso, y esperá su confirmación antes de tocar código — incluso si el pedido de Sebastián en ese mensaje sonaba a "seguí no más". (Excepción: si el mensaje de la sesión es uno de los prompts aprobados de `PROMPTS-AUDITORIA.md`, ese prompt ES la confirmación de fase.)

### Cortar procesos

Si necesitás frenar un proceso en curso (por ejemplo, una corrida de Playwright que arrancó mal o quedó colgada): primero probá el mecanismo del propio harness para detener la tarea en segundo plano que vos mismo lanzaste. Si eso no alcanza y hace falta `taskkill` (u otro comando que mate procesos directo), **preguntale a Sebastián antes de ejecutarlo** — no lo hagas por tu cuenta, aunque el proceso sea uno que vos mismo arrancaste. Explicale qué proceso es, por qué hace falta cortarlo, y esperá su confirmación.

---

## 9. Commits

Uno por arreglo o fase cerrada, con todos los tests pasando.

```
git add <archivos del arreglo>
git commit -m "Arregla <qué> en <dónde> (hallazgo TX-NN)"
```

Cada commit es un punto seguro al que volver. Si algo sale mal tres arreglos después, se puede volver acá.

**`[lección]` Antes de cada commit, mirar `git status --short` y confirmar que lo
que se va a subir es exactamente lo que se quiso cambiar, nada más.**
Un `git add <archivo>` puede arrastrar cambios que ya estaban en el
índice de antes (por ejemplo, `js/legacy.js` apuntando a una rama de
referencia durante una sesión de capturas) y terminar mezclados
en un commit que iba a ser de otra cosa. Encontrado el 2026-09-23: casi
pasó con el commit de una regla de `CLAUDE.md`, detectado a tiempo
revisando el propio commit después de hacerlo.

**`[lección]` Commitear el cambio de código antes de correr la suite completa
con `--repeat-each=5`, no después.** El orden correcto del ciclo
(sección 4) es: cambiar, sincronizar, verificar por hash, **commitear**,
y recién ahí correr la suite completa — así lo que da verde es
exactamente lo que quedó guardado en el repo, no un estado intermedio
sin commitear que después se sube por separado. Encontrado el
2026-09-23, cerrando Diagnóstico: la migración quedó sin commitear
mientras corría la suite completa (que sí dio verde), y se commiteó
recién después. No pasó nada — el hash del servidor coincidía con lo
que se terminó commiteando — pero fue por casualidad de que nada tocó
el repo en el medio, no porque el orden estuviera bien.

---

## 10. Documentos: AUDITORIA.md y el archivo del refactor

- **Lo que se lee en frío en esta etapa: `CLAUDE.md` + `AUDITORIA.md`.** El
  plan, las reglas y el estado de las tandas viven en `AUDITORIA.md`; **los
  hallazgos y las decisiones de Sebastián, en `HALLAZGOS-AUDITORIA.md`** (índice
  adelante); los prompts de las tandas en `PROMPTS-AUDITORIA.md`.
- **`ESTADO.md`, `HISTORIA.md` y `HALLAZGOS.md` quedan congelados como el
  archivo del refactor** (cerrado 2026-09-29): se consultan puntualmente, no se
  les agrega nada nuevo (solo correcciones de erratas, con aviso). `ESTADO.md`
  era lo que se leía en frío durante el refactor; ese rol pasó a `AUDITORIA.md`.
- Todo documento que se actualice se copia a la carpeta compartida en el
  mismo paso, para que el asistente de planificación de Sebastián lo lea al
  retomar.

### `[lección]` Cómo citar código en los documentos: archivo y nombre de función

**Un número de línea suelto se rompe con cada cambio de código.** Encontrado el
2026-09-26: de 42 referencias `legacy.js:<línea>` en `ESTADO.md`, más de la
mitad ya no existían tras las migraciones.

- **Citá archivo + nombre de función** (`notificaciones.js`,
  `descartarNotif()`), no `legacy.js:1494`. El nombre se encuentra con un
  grep aunque la función se mueva; el número no.
- **El número de línea solo cuando sea imprescindible** (por ejemplo, para
  señalar una sola línea dentro de una función larga). Si lo ponés, va
  siempre junto al archivo y al nombre de la función, para poder
  reencontrarlo cuando se corra.
- **Precisión para la auditoría:** dentro de un hallazgo de
  `HALLAZGOS-AUDITORIA.md` el número de línea SÍ va (junto a archivo y función,
  como pide su formato) — el código está congelado mientras se audita y la
  línea es evidencia.
- **Al cerrar una fase que movió código, grep de las referencias afectadas**
  en los documentos vigentes y corregirlas en el mismo cierre, no después.

### `[lección]` CLAUDE.md también se copia a la carpeta compartida, en el mismo commit en que se edita

Igual que `ESTADO.md`: cuando se edita `CLAUDE.md`, se copia a la carpeta
compartida **antes de commitear**, en el mismo paso, no después ni "cuando
se acuerde". La copia compartida es la que lee el asistente de
planificación de Sebastián; si queda atrás, le falta la regla nueva.
Encontrado el 2026-09-26: ninguna regla decía que había que copiarlo, y la
copia compartida había quedado en la versión anterior al commit de la regla
de citar archivo y función. Después de copiar, confirmar que quedaron
idénticos (`cmp`).

---

## 11. Deuda técnica conocida

Anotada para no perderla; se arregla solo con autorización por ítem (sección 7):

- *(Un ítem de configuración de la base, omitido en la versión pública.)*
- **`[lección]` Tema claro/oscuro en los tests (corregido 2026-09-19).** El tema **no** se guarda en el servidor: vive en `localStorage` (`applyTheme()` en `legacy.js`), y cada test arranca con un contexto limpio, así que el test de modo claro no afecta a los demás. Sin preferencia guardada la app copia `prefers-color-scheme`, y Playwright emula `light` por defecto: por eso los describes "oscuro" de `visual.spec.js` fuerzan `colorScheme: 'dark'`. Sin eso, las capturas "-oscuro" post-login salen en modo claro (así estuvieron hasta esta fecha).
- **Workers de Playwright (resuelto 2026-09-30, paso 0e).** Una cuenta de prueba por worker, default `workers: 1`, suite completa con `--workers=5` (sección 5).
- **Secretos de la aplicación hardcodeados en `api/index.php`:
  resuelto en `code` con el paso 0 (2026-09-29, archivo de secretos fuera del
  repo, sección 2) y en `app` con la migración de servidor (2026-10-06,
  `PLAN-migracion.md` §8).**
  El usuario de base propio por entorno también quedó resuelto en la
  migración (paso 0c en `code`; en el servidor nuevo `app` tiene el suyo y
  el usuario compartido no existe).
- **`onclick` inline en el HTML.** Segunda pasada, fase futura propia.

Esta lista no está completa. Lo que encuentres se suma a `HALLAZGOS-AUDITORIA.md` (ver secciones 7 y 10).

---

## 12. Datos útiles

*Sección interna omitida en la versión pública.*

---

## 13. Promociones a producción

La promoción del refactor ya se hizo y verificó (2026-09-29, `HISTORIA.md`). Hoy `code` y `app` coinciden en los 14 archivos promovidos. El procedimiento de abajo queda como **permanente**, tal como se probó.

- **Cada arreglo que llegue a `code` deja a `app` atrás hasta la próxima promoción: esa divergencia se registra en `AUDITORIA.md` apenas se commitea el arreglo**, para que nunca haya que reconstruir de memoria qué falta promover.
- **`[lección]` Cuándo se promueven los arreglos pre-piloto (todos juntos o en tandas) lo decide Sebastián**; ninguna promoción sale sin su plan de archivos/hashes aprobado. El motivo de fondo sigue vigente: cada promoción suelta es un riesgo sobre clientes reales y mezcla las causas si algo falla (el 2026-09-20 pasó exactamente eso: un archivo equivocado llegó a `app` y el CDN siguió sirviendo otro distinto durante horas).
- **Quién promueve:** Sebastián, a mano (sección 2). Vos nunca ejecutás una promoción.
- **Antes de promover:** vos armás el plan y Sebastián lo confirma: lista exacta de archivos con su ruta y el hash esperado de cada uno; una carpeta de staging con **solo** esos archivos (nunca el árbol del repo, nunca `--delete`, nunca `api/index.php` del repo — ver sección 2); el orden de subida (dependencias primero; el archivo que dispara los `import()`, último); y cómo volver atrás, con el rollback ensayado antes.
- **Después de subir, verificar siempre:** (1) el hash de cada archivo **en el origen**, no solo lo que muestra el navegador; (2) **purgar la caché del CDN** de esos archivos (el HTML congelado pide `?v=N` y no se puede cambiar, y los `.js`/`.css` se cachean 4 horas); (3) volver a verificar desde afuera, por el borde del CDN y por el origen (con un parámetro nuevo en la URL). Esa verificación desde afuera la hacés vos solo con pedidos de lectura a archivos estáticos públicos y consultas públicas de solo lectura, sin credenciales ni escrituras, y con el visto bueno de Sebastián para hacerla.
- **`[lección]` Ojo con CRLF al comparar por hash (`core.autocrlf=true`):** un `git switch` reescribe con CRLF los archivos que cambian entre ramas, así que la copia de trabajo puede quedar con CRLF aunque el repo guarde LF — el hash de un archivo de la copia de trabajo puede no coincidir con el del repo por esto solo, sin que el contenido real haya cambiado. Comparar ignorando CRLF, o contra `git show`. Los hashes de una promoción son siempre los del working tree (los bytes probados en `code`), nunca los de `git show`.
- **Única excepción a "con plan aprobado":** una urgencia real, decidida por Sebastián en el momento. Se registra en `AUDITORIA.md` y se deja anotado que `code` deja de coincidir con `app` en esa parte, hasta que se sincronice.

---

## 14. Resumen

1. Trabajás en el repo local, sincronizás a `code` para ver y testear
2. CSS y HTML congelados; el comportamiento visual se mantiene — los cambios los pide Sebastián (regla 3.3)
3. Una fase por vez, con confirmación de Sebastián antes de arrancarla (los prompts de `PROMPTS-AUDITORIA.md` ya son esa confirmación)
4. Durante una tanda de auditoría: solo lectura, hallazgos a `HALLAZGOS-AUDITORIA.md`, no se arregla nada; un crítico explotable corta la tanda
5. Un arreglo solo con autorización por ítem, test primero (falla antes, pasa después), y su propio commit
6. Todo en verde con `--workers=5 --repeat-each=5` (una cuenta por worker) antes de cada commit — una sola corrida no alcanza
7. Nunca corras `--update-snapshots` por tu cuenta: una captura que no coincide es la señal, no el problema; autorizado, siempre con `--workers=1` y la cuenta original
8. Nunca tocar `app` ni el script de promoción (la excepción "`app` vacía" está cerrada); el `api/index.php` del repo es el de `code` (sin secretos) y a `app` va solo por promoción
9. Cambios al PHP: ciclo de la sección 4, los sube Sebastián a mano; SQL solo lo corre Sebastián
10. Las promociones las hace Sebastián con plan de archivos/hashes aprobado; cada divergencia `code`≠`app` se registra en `AUDITORIA.md`
11. Ante la duda, preguntar
