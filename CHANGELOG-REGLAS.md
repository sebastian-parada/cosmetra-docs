# Revisiones de `CLAUDE.md`

Fecha y motivo de cada commit que tocó las reglas de trabajo del agente,
del más nuevo al más viejo. Los mensajes son los originales, con nombres
internos reemplazados por descripciones. Total: 31 revisiones entre el
2026-09-16 y el 2026-10-07.

| Fecha | Revisión |
|---|---|
| 2026-10-07 | §5: desde la migración cada sitio tiene su propio pool de PHP; la regla de no correr la suite en horario de uso se mantiene |
| 2026-10-07 | §11: los secretos de la aplicación y el usuario de base propio quedaron resueltos también en producción con la migración |
| 2026-09-30 | Suite en paralelo con una cuenta de prueba por worker; actualizar capturas de referencia siempre en serie y con la cuenta original; regla del foco diferido antes de escribir en un campo |
| 2026-09-29 | El PHP del repo pasa a ser el que corre en el entorno de pruebas: los secretos de la aplicación salen del código y del repo |
| 2026-09-29 | Cuatro precisiones al ciclo de cambios al PHP (dónde viven los secretos, arreglos sin rojo→verde observable, arreglos que cruzan PHP y cliente, limpieza de lo que escriben los tests nuevos) |
| 2026-09-29 | Las referencias a hallazgos apuntan al registro de la auditoría |
| 2026-09-29 | Pasada de consistencia (aprobada con el diff a la vista) |
| 2026-09-29 | Actualización a la etapa de auditoría: ocho correcciones de secciones obsoletas y seis agregados, aprobados uno por uno |
| 2026-09-27 | Plantilla del documento de estado, tras partirlo en tres archivos |
| 2026-09-27 | Convenciones durables y notas de método que quedaban en el diario pasan a las reglas |
| 2026-09-26 | Autoriza correr tests contra producción solo mientras esté vacía (excepción cerrada después, el 2026-09-29) |
| 2026-09-26 | Regla: copiar `CLAUDE.md` a la carpeta compartida en el mismo commit en que se edita |
| 2026-09-26 | Regla: citar archivo y función, no número de línea, en los documentos |
| 2026-09-25 | Documenta el patrón de simular la respuesta del servidor en los tests |
| 2026-09-24 | Regla: regenerar todo el grupo de capturas, no solo la que falló |
| 2026-09-23 | Commitear el cambio antes de correr la suite completa, no después |
| 2026-09-23 | Revisar el estado del índice de git antes de cada commit |
| 2026-09-23 | Toda captura de un modal tiene que cerrarlo después de la foto |
| 2026-09-23 | Procedimiento de sabotaje documentado, incluido que la sincronización no propaga borrados |
| 2026-09-22 | El plan de cada módulo también se copia a la carpeta compartida |
| 2026-09-22 | Regla: el plan de cada módulo se escribe en su propio archivo antes de tocar código |
| 2026-09-22 | Regla: pedir permiso antes de matar un proceso |
| 2026-09-22 | Capturas de un grupo regeneradas (página completa, sin avisos colgados); índice de deuda en el documento de estado |
| 2026-09-21 | Carpeta compartida para la revisión visual de capturas |
| 2026-09-20 | Nueva sección 7: criterio de éxito del refactor |
| 2026-09-20 | Cierre de la promoción del arreglo del login; regla de promociones a producción |
| 2026-09-19 | Las capturas en modo oscuro fuerzan el esquema de color oscuro; 16 referencias regeneradas |
| 2026-09-19 | Rutas actualizadas tras mover el repo |
| 2026-09-18 | Ciclo con capturas y repetición ×5; reglas de confirmación previa |
| 2026-09-16 | Limpieza automática de fichas de prueba; tests de Sesiones |
| 2026-09-16 | Estado inicial: código, tests y reglas del refactor |
