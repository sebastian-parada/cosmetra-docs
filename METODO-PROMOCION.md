# Método de promoción a producción

Cómo se compara el entorno de pruebas (`code`) con producción (`app`)
antes de promover, y cómo se promueve. El procedimiento se probó en la
promoción del refactor completo (12 módulos, núcleo y 6 arreglos de
seguridad), el 2026-09-29, y quedó como permanente.

## Principio

**No se confía, se compara.** La suite en verde en `code` no demuestra
que `app` vaya a comportarse igual: demuestra que `code` se comporta como
la suite espera. Antes de promover, se compara lo que se va a subir con lo
que hay, y se calibra el instrumento de comparación contra lo que ya se
sabe.

## 1. Comparación a fondo antes de promover

Pedido del dueño: comparar los dos entornos a fondo, sin confiar solo en
que la suite pasa. Las únicas diferencias esperadas eran los arreglos de
seguridad y la reorganización en módulos; **todo lo demás era sospechoso
hasta demostrar lo contrario**.

**Por qué alcanzaba con comparar el DOM y los estilos.** El CSS y el HTML
estático estaban congelados y eran byte-idénticos entre los dos entornos
(verificado por hash). Una diferencia visual solo podía venir del HTML y
las clases que genera el JavaScript, de estilos inline, de estados
intermedios o del entorno. Eso se compara directo, con la aplicación
corriendo en los dos lados.

**Paso 0, preparación.** El dueño confirmó por SQL que producción estaba
vacía de datos reales. La verificación previa por API encontró tres
diferencias de estado respecto de pruebas (tres fichas de pruebas
manuales, un nombre de negocio distinto, un dato de perfil distinto); se
resolvieron por API con los mismos endpoints que usa la interfaz, y lo que
no se veía en ninguna captura se dejó sin tocar, por decisión del dueño.

**Paso 1, calibración.** Las 77 capturas de referencia, generadas en
`code`, se compararon contra `app` **sin regenerar nada**: 77/77 iguales,
pixel a pixel. Conclusión: no había ruido de entorno que restar;
cualquier diferencia posterior sería sospechosa de entrada, no descartable
por "es el entorno".

**Paso 2, control positivo.** Antes de construir la matriz de comparación
se validó que el arnés detectara **lo que ya se sabía que estaba**: los
cinco arreglos de XSS. Cada escenario reusó el payload de los tests y
exigió las dos condiciones a la vez (pruebas escapa **y** producción no),
cortando con error si cualquiera fallaba, en vez de reportar "sin
diferencias". 5/5 confirmadas, incluida la del PDF, que no es DOM. **El
arnés detecta lo que tiene que detectar; se puede confiar en él.**

**Paso 2b, recorte justificado.** La matriz original (6 anchos × ~20
pantallas × 2 temas) se descartó con el motivo escrito: como el CSS era
el mismo archivo en los dos lados, el ancho de pantalla solo podía
importar donde el JavaScript leyera el ancho de la ventana. Se buscaron
esos lugares en todo el código: seis, los seis ya cubiertos por una
captura. Se compararon directo (4/4 exactos) y se hizo un barrido de
anchos acotado: 8 de 9 combinaciones con 0 diferencias; la novena dio 11
diferencias sub-píxel (0,01 a 0,04 px) en elementos sin relación entre
sí. **No se descartó sola**: autorizado por el dueño ("no la descartes
vos, pasámela"), se investigó y se repitió tres veces más: 0/0/0. Ruido
intermitente, no una diferencia real. El criterio acordado para
distinguirlos: lo que cambia entre corridas es ruido; lo que se repite es
real.

**Paso 3, datos reales completos.** Una ficha con todos los campos llenos
(cada grupo Sí/No en "Sí" con su condicional revelado), diagnóstico
completo y dos sesiones, creada con **el mismo payload exacto** por API en
los dos entornos. Cuatro pantallas comparadas por `outerHTML` normalizado
(tolerancia cero, normalizando solo los ids autoincrementales) y por la
geometría de cada elemento. La primera corrida fue una falsa alarma
propia: el sufijo aleatorio del apellido se generaba por separado en cada
cuenta, así que el "mismo" payload no era idéntico; corregido y repetido.
Con el payload realmente idéntico: HTML idéntico en las cuatro pantallas
en las cuatro corridas; los deltas de geometría de la primera (0,04 a
0,21 px) no se repitieron en tres corridas más. La ficha se borró en las
dos cuentas al final de cada corrida.

**Resultado:** control positivo 5/5, calibración 77/77, 0 diferencias
reales fuera de lo esperado.

## 2. Procedimiento de promoción (permanente)

**Quién.** El dueño, a mano. El agente arma el plan, verifica desde
afuera y corre la suite; **nunca ejecuta una promoción**.

**Antes de subir.**

- Lista exacta de archivos con su ruta y el **hash esperado** de cada uno,
  tomado del working tree (los bytes probados en `code`), nunca de
  `git show`: con `core.autocrlf` activo, el repo guarda LF pero la copia
  de trabajo puede tener CRLF; el JavaScript funciona igual, pero no
  serían los bytes probados y la verificación por hash fallaría. **Al
  comparar por hash, ignorar CRLF o comparar contra lo que sirve el
  servidor.**
- Una carpeta de staging con **solo** esos archivos: nunca el árbol del
  repo, nunca `--delete`, nunca el PHP junto con el JavaScript (cada uno
  tiene su propio ciclo y su propia subida).
- Verificación del staging contra la tabla: ningún archivo de más, ninguno
  de menos (14/14 en la promoción del refactor).
- Confirmación de que en producción los archivos nuevos no existen todavía
  (404 en el origen) y de que el estado previo está limpio.
- Orden de subida: dependencias primero; **el archivo que dispara la
  carga de los módulos, último**, para que la aplicación no quede a medias
  si alguien entra en el medio.
- La vuelta atrás, **ensayada antes**: el archivo anterior guardado con su
  hash; volver es reinstalarlo.

**Después de subir, siempre.**

1. El hash de cada archivo **en el origen**, no solo lo que muestra el
   navegador.
2. **Purgar la caché del CDN** de esos archivos: el HTML congelado pide
   los scripts con un parámetro de versión fijo que no se puede cambiar, y
   los estáticos se cachean cuatro horas.
3. Volver a verificar **desde afuera**, por el borde del CDN y por el
   origen, con un parámetro nuevo en la URL. En la promoción del refactor:
   14 archivos × (borde + origen) = **28/28** con el hash de la tabla, y la
   URL real del HTML sirviendo el archivo nuevo tras la purga.
4. Corrida de aceptación en producción, una sola vez (250 pasaron, 0
   fallaron en la promoción del refactor, la suite de ese momento), y
   verificación de que no quedó ningún dato de la corrida.
5. Revisión visual del dueño, en una ventana sin sesión.

Esa verificación desde afuera la hace el agente solo con pedidos de
lectura a archivos estáticos públicos, sin credenciales ni escrituras, y
con el visto bueno del dueño.

**Cada arreglo que llega a `code` deja a `app` atrás hasta la próxima
promoción.** Esa divergencia se registra en el documento de la etapa
apenas se commitea el arreglo, con lo que hace falta para llevarlo a
producción (archivos, orden, qué debe existir antes en el servidor), para
que nunca haya que reconstruir de memoria qué falta promover.

## 3. Por qué el procedimiento es así: el incidente del 2026-09-20

En una promoción anterior (un arreglo del login), un archivo equivocado
llegó a producción y el CDN siguió sirviendo otro distinto durante horas.
De ahí salen tres reglas que hoy son permanentes: verificar el hash en el
origen y no en el navegador; purgar la caché y volver a verificar por el
borde con un parámetro nuevo; y promover con un plan de archivos y hashes
aprobado, nunca suelto, porque cada promoción es un riesgo sobre clientes
reales y una promoción suelta mezcla las causas si algo falla.
