# La suite de pruebas

Descripción de la suite end-to-end de Cosmetra, sin código. Datos del
2026-10-07.

| Dato | Valor |
|---|---|
| Tests declarados | 258 en 4 archivos (Playwright, Chromium) |
| Capturas de referencia | 83 archivos: 28 pantallas o estados, en desktop oscuro, desktop claro y mobile oscuro (algunos estados solo en una variante) |
| Bloques de comportamiento | 42 `describe`: humo y profundidad por módulo, más regresión de hallazgos cerrados |
| Suite completa ×5, 5 workers | 1.290 ejecuciones: 1.285 pasaron, 5 salteadas a propósito, 0 fallaron; ~32 minutos (tres corridas medidas: 32,6 / 32,4 / 30,4) |
| La misma suite en serie | 2,4 horas |

Los tests corren contra el **entorno de pruebas desplegado**, no contra la
carpeta local: prueban lo que está en el servidor. Por eso el ciclo es
siempre cambiar → sincronizar → verificar por hash → correr.

## 1. Estructura

**Tres archivos principales y uno de investigación.**

- **Comportamiento** (179 tests): un bloque de *humo* y uno de
  *profundidad* por cada uno de los 12 módulos de la aplicación (Panel de
  Control, Ver Todos, Ficha completa, Sesiones, Notas, Configuración,
  Diagnóstico, Edición inline, Firmas digitales, Enviar y aprobar ficha
  de la paciente, Notificaciones, Exportación), más Login y Navegación.
  El humo recorre el camino principal de la pantalla; la profundidad
  fija los puntos de acoplamiento con el resto (estado compartido, qué
  pasa al volver de una pantalla a otra, qué cierra qué).
- **Visual** (77 tests, uno por captura): capturas de cada pantalla con
  `toHaveScreenshot()`, en tres bloques: desktop oscuro, desktop claro,
  mobile oscuro. Incluye modales, estados vacíos y estados con datos. Las
  otras 6 capturas de referencia se comparan solo en contextos
  particulares, fuera de la suite normal.
- **Login** (1 test de carrera, aislado en su archivo porque necesita su
  propio contexto).
- **Investigación** (1 test): reproduce a demanda una carrera conocida
  entre la navegación a una ficha y un formulario que se abre con un
  retardo fijo. Se saltea siempre, salvo que se pida con una variable de
  entorno. **Las 5 "salteadas" de cada corrida ×5 son este único test,
  repetido cinco veces.** No es un test roto ni pendiente: es una
  herramienta de diagnóstico que no tiene sentido correr en cada pasada.
- **Regresión de hallazgos cerrados**, dentro del archivo de
  comportamiento: cada arreglo de seguridad deja un test que falla sin el
  arreglo (los endpoints protegidos por secreto fallan cerrado sin él; el
  límite de intentos de login no se esquiva con cabeceras de IP; cada
  punto de XSS almacenado con su payload).

**Setup y teardown globales.** Antes de correr, un setup verifica cada
cuenta de prueba (0 fichas, 0 pendientes, configuración base) después de
limpiarla, y **corta la corrida** si alguna no está limpia o si se piden
más workers que cuentas. Al terminar, un teardown vuelve a limpiar. Los
datos que crea un test llevan un prefijo reconocible y los borra el propio
test o el teardown; un test que escribe en una tabla sin limpieza propia
define su limpieza **antes** de correrse.

## 2. La regla del ×5

Una corrida limpia no alcanza. Antes de commitear o de declarar una fase
lista:

```
npx playwright test --workers=5 --repeat-each=5
```

**Por qué es obligatorio.** En el módulo Notas, una sola corrida daba
resultados distintos cada vez (26/33, 29/33, 30/33) y eso hizo perder
horas discutiendo si era un bug real o ruido del entorno. Con cinco
repeticiones el patrón fue inmediato: un test fallaba 5 de 5 (bug real,
resultó ser una carrera en el login) y otros fallaban 1 de 5 de forma
dispersa (ruido). **Un test que falla 5/5 o 4/5 es un bug. Uno que falla
1/5 en tests distintos cada vez es ruido de entorno.**

Variante: **un fallo que aparece siempre en el mismo worker** es estado
de la cuenta de ese worker, no ruido; se mira esa cuenta antes de
sospechar del código.

## 3. Capturas de referencia

Los tests de comportamiento no detectan que un margen cambió o que un
botón se movió. Eso es justo la prioridad número uno del dueño del
proyecto: la interfaz no se rompe visualmente. Por eso hay capturas de
referencia que se comparan en cada corrida.

Reglas que gobiernan las capturas:

- **Nunca se actualizan las referencias sin autorización.** Una captura
  que no coincide es exactamente la señal que el sistema existe para dar;
  actualizarla sin aprobación es borrar la evidencia de una regresión. La
  revisión la hace el dueño con la imagen del diff que genera Playwright.
- **Cuando está autorizado, siempre en serie y con la cuenta original**,
  acotado a las capturas autorizadas: todas las referencias salen del
  mismo lugar. Un dato propio de la cuenta que llegue a una captura se tapa
  con `mask`; no se normaliza el DOM antes de la foto.
- **Una captura de un modal tiene que cerrarlo después de la foto.** Si
  no, el backdrop tapa la página y bloquea los clicks de lo que corra
  después en la misma sesión.
- **Una captura con backdrop no ve lo que hay debajo.** Cubre cómo se ve
  el modal (centrado, botones, tema), no lo que el modal deja o cambia
  debajo. Eso necesita un test de comportamiento aparte. Encontrado en un
  sabotaje (§5).
- **Capturas con datos, no solo pantallas vacías.** Toda pantalla cuyo
  estado con datos se ve distinto del vacío tiene al menos una captura
  con datos reales, creados por el test y borrados de inmediato.
- **Al regenerar una captura por un fix, se regenera todo su grupo**
  (desktop oscuro, desktop claro, mobile oscuro). Una referencia vieja y
  mal generada puede seguir pasando en verde indefinidamente: pasó con
  una captura de Firmas que tenía la firma aplicada a la ficha equivocada
  y pasó todas las comparaciones porque el cambio quedaba debajo del
  umbral.
- **El umbral tiene una debilidad conocida:** `maxDiffPixelRatio: 0.01`
  equivale a unos 13.000 píxeles en desktop. Un cambio chico (dos badges,
  dos porcentajes) pasa. Está anotado como deuda, no resuelto.
- **Modo oscuro:** el tema no vive en el servidor sino en el navegador, y
  Playwright emula el esquema claro por defecto. Los bloques "oscuro"
  fuerzan `colorScheme: 'dark'`; hasta que se descubrió, las capturas
  "oscuras" posteriores al login salían en modo claro.

## 4. Una cuenta de prueba por worker

La suite completa ×5 en serie tardaba 2,4 horas, más que el límite de una
tarea en segundo plano del agente. La solución fue correr en paralelo con
**una cuenta de prueba por worker** (cinco cuentas; el worker 0 usa la
original). Reglas:

- **Nunca más workers que cuentas.** El setup global corta la corrida si
  pasa. Dos workers sobre la misma cuenta dan contención y fallos que no
  son reales.
- **Sin `--workers`, el default es 1** y todo corre contra la cuenta
  original: sirve para un test suelto o para depurar.
- **Sin `fullyParallel`:** Playwright reparte por archivo; con ×5 son 10
  bloques grandes, 2 por worker, cada uno con el orden interno de siempre.
- **El servidor no fue el cuello** (medido: 97 % de CPU libre con 5
  workers). Igual, cuando haya usuarias reales la suite no se corre en
  horario de uso: CPU y base siguen siendo compartidas con producción.

Resultado: de 2,4 horas a ~32 minutos, con la misma suite.

## 5. Sabotajes: verificar que un test detecta lo que dice detectar

Cuando un test existe para blindar un punto de acoplamiento riesgoso (una
variable compartida que podría moverse mal al migrar código), no alcanza
con que pase contra el código bueno: hay que probar que **falla** contra
una versión rota a propósito, antes de confiar en él.

Procedimiento, en ocho pasos:

1. Escribir el test contra el código sin tocar, con una **predicción
   escrita** de qué falla y por qué si el acoplamiento se rompe. La
   predicción se commitea antes de correr.
2. Confirmar ×5 en verde contra el código sin tocar.
3. Aplicar el sabotaje: la forma "intuitiva pero incorrecta" de mover eso
   (normalmente un módulo temporal). Sincronizar y verificar por hash.
4. Confirmar que el test falla como se predijo, y que cualquier test de
   control sigue en verde.
5. Revertir, y **borrar también los archivos nuevos** que el sabotaje haya
   creado: un `git checkout` no toca lo que no estaba en el commit.
6. Sincronizar el revert. **La sincronización no propaga borrados:** un
   archivo temporal borrado solo del lado local queda huérfano en el
   servidor para siempre. Se borra ahí también y se verifica con un pedido
   de lectura que da 404, no solo con que el `rm` no tiró error.
7. Re-verificar por hash que el archivo real volvió a coincidir.
8. Confirmar ×5 en verde de nuevo.

**Qué revelaron los sabotajes.** Dos tipos de problema, dos veces cada
uno:

- **Dos tests con falso negativo** (Notificaciones): el sabotaje rompió a
  propósito el acoplamiento entre Configuración y el aviso local de
  cambio de configuración. Los dos tests que ya existían para ese camino
  **siguieron en verde con la función rota**: solo verificaban lo que
  ocurre antes de la llamada que falla (la persistencia en el servidor),
  no el aviso ni el toast de error que aparecía después. Sin los tests de
  profundidad nuevos, una migración mal hecha habría dejado la suite en
  verde y el defecto habría llegado a producción. Lección, hoy regla:
  **"hay un test que pasa por ahí" no es lo mismo que "un test que
  fallaría si se rompe"**.
- **Dos agujeros de cobertura.** En Panel de Control, un sabotaje mostró
  que ningún test fijaba el efecto de un filtro activo sobre el mensaje
  de "Ver Todos" vacío: se escribió el test que faltaba y se verificó
  contra el sabotaje reaplicado antes de commitear nada. En Exportación,
  un sabotaje dejó un menú abierto detrás de un modal y la captura mobile
  **pasó 5/5** igual (la fila del menú quedaba bajo un fondo casi negro,
  por debajo del umbral); solo lo detectó un test de comportamiento. De
  ahí la regla de que una captura con backdrop no ve lo que hay debajo.

## 6. Simular la respuesta del servidor

Para probar cómo **el cliente** pinta datos que un test no puede producir
de verdad, o que dependen del reloj, se simula en el navegador la
respuesta de un endpoint con `page.route`, en vez de sembrar datos reales.
Primer uso: los avisos de Notificaciones, donde un aviso de embarazo
necesita nueve meses, un link de firma vence 24 horas después y el texto
de un aviso de ausencia cambia con el tiempo.

Reglas del patrón:

- Un helper por endpoint simulado, con la misma forma de respuesta que
  devuelve la API real. Nunca un `route` suelto en cada test.
- Llamarlo antes del login, porque el login también pide esa lista.
- Si el test puede disparar una escritura sobre un dato simulado
  (descartar, borrar), simular también esa acción: si no, se escribe en
  la base real con ids falsos.
- **Qué prueba y qué no:** prueba el código del cliente. No prueba que el
  servidor genere bien los datos: eso tiene su propio ciclo, así que cada
  área que use el patrón mantiene además una prueba de punta a punta con
  datos reales.
- En capturas, una captura con datos simulados no reemplaza a la de
  datos reales: cada plan de fase dice qué se simula y por qué, y el
  dueño lo aprueba.

## 7. Dos carreras que enseñaron reglas

- **`fill()` contra un foco diferido.** La aplicación mueve el foco a un
  campo con un `setTimeout`. Si el test escribe en el medio, el texto cae
  en otro campo. Regla: `await expect(campo).toBeFocused()` antes de
  escribir. Apareció como fallo 1/5 en dos lugares distintos antes de
  entenderse.
- **Leer antes del `await`, pintar después.** El render de la tabla del
  Panel lee el filtro activo antes de esperar los datos y pinta las
  columnas después; un test que cambia el filtro en el medio ve una tabla
  inconsistente. Los tests esperan a que la caché de la tabla esté lista
  antes de tocar el filtro, y el comportamiento de la aplicación quedó
  anotado para su propio ciclo: la auditoría manda anotar, no arreglar.
