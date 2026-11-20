# Arquitectura del Prode Fixture 2030

Revisión del 20 de noviembre de 2026 sobre el [TPO publicado en `91cae0e`](https://github.com/tomaszeppo/Grupo16_BD2_repo/tree/91cae0e/TPO). La interfaz de usuario y la administración se sirven desde FastAPI en `/demo/`. Este sitio React documenta el sistema. Las consultas individuales de pronóstico y las columnas de pronóstico y puntos del partido en el modal de grupo pertenecen a una revisión local posterior.

## Del pronóstico al histórico

| Momento | Qué ocurre | Dónde se consulta |
|---|---|---|
| Antes del partido | Cada usuario carga el marcador local y visitante por API. Puede corregirlo. El mismo pronóstico vale en todos sus prodes. | MongoDB guarda una fila por usuario y partido. Neo4j gobierna el fixture. |
| Inicio | Administración reserva la simulación. El simulador obtiene el guion y los planteles de MongoDB, abre sesiones de hinchas y envía hasta ocho pronósticos de prueba argentinos por HTTP. El cambio a **En juego** registra el inicio real en Neo4j y dispara la preparación del ranking. | La lista `GET /en-vivo/partidos` sale de la ficha en MongoDB, que contiene partido, corrida, minuto y marcador. |
| Minutos 0–90 | Un segundo real representa un minuto simulado. La API de muestras calcula el marcador contando los goles del guion hasta ese minuto y escribe jugadas y posesión en Influx. Los hinchas publican por el mismo endpoint que una persona. | Estadísticas y relato salen de Influx. El historial del chat sale de Cassandra. Redis distribuye las novedades por SSE. |
| Resultado | La API valida el marcador, registra el resultado del prode en MongoDB y finaliza el partido en Neo4j. La puntuación se ejecuta en segundo plano. | Clasificación acumulada en Redis con la configuración de Compose. El partido puede estar finalizado mientras se puntúan los pronósticos. |
| Archivo | El simulador pide el archivo por HTTP. La API contrasta Influx con el guion, persiste agregados y confirma el histórico en Neo4j. | Con archivo completo, estadísticas y jugadas se leen del grafo. Los comentarios siguen en Cassandra. |

### Neo4j · autoridad deportiva

El fixture pasa de **Programado** a **En juego** y luego a **Finalizado**. Se requieren dos equipos definidos. En eliminatorias, administración puede asignar dos equipos existentes y distintos antes del inicio. La clasificación del torneo se carga a mano. La fase de grupos rechaza un clasificado. En eliminatorias gana quien marque más goles y, si empatan, el cierre exige elegir uno de los dos. La simulación elige al local en ese caso.

Una jugada manual exige partido en juego, protagonista de alguno de los equipos y antecedentes del mismo partido. Repetir el ID con los mismos datos recupera la jugada. Reutilizarlo con otros datos produce conflicto. Si el grafo ya registra goles, el cierre comprueba su concordancia con el marcador. El histórico vincula jugador y evento con `PROTAGONIZA`, evento y partido con `OCURRE_EN`, y antecedente y consecuencia con `INFLUYE_EN`. El análisis de una cadena recorre hasta diez saltos.

### MongoDB · pronóstico y resultado durable

El replica set de tres nodos conserva usuarios y país, equipos y jugadores, integrantes e invitaciones de prodes privados (máximo 100), guiones, fichas de simulación y el resultado oficial para puntuar. Pronósticos y resultados requieren confirmación de mayoría. El identificador usuario–partido evita duplicados. La última edición fija la hora usada para puntuar.

Cada proceso de FastAPI conserva el calendario durante 30 s por defecto. Poner el partido en juego invalida la copia de ese proceso. Con `PRONOSTICOS_HASTA_INICIO_REAL=true` (Compose), un partido programado puede seguir abierto pasada la hora prevista. Con `false` también rige el cierre por horario. Toda carga desde el instante real de inicio queda anulada al puntuar, incluso si otro proceso la aceptó durante esa ventana.

`resultados` guarda marcador, clasificado, tiempos y avance de la puntuación. `pronosticos` conserva el puntaje individual de ese encuentro. `puntajes` suma partidos aplicados y `rankings_historicos` conserva los primeros 100 y el total de usuarios por cierre. La ficha de simulación guarda una copia del guion para validación. El detalle temporal se archiva por otro recorrido.

### Redis · puntuación, sesiones y distribución del vivo

1. **Preparar.** Con `REDIS_ENABLED=true` se reconstruye el ranking desde los acumulados de MongoDB si hace falta. Al iniciar el partido se copia en un hash, por lotes de 500 por defecto, el marcador pronosticado y la hora de cada participante. La preparación se repite al cierre para incluir ediciones tardías que deben anularse.
2. **Puntuar.** El marcador exacto vale **3**, acertar ganador o empate vale **1** y cualquier otro resultado vale **0**. Por ejemplo, para un 1–1 final, predecir 2–2 suma un punto y predecir 1–2 suma cero. Un pleno suma toda la anticipación desde la última edición hasta el inicio programado, acotada entre 0 y 5 días. Acertar solo el signo suma la mitad en segundos enteros. Un error o anulación suma cero anticipación.
3. **Ordenar.** Por participante, un script Lua actualiza el hash de acumulados, el sorted set `prode:ranking` y la marca del partido aplicado. Los reintentos evitan sumar dos veces. El orden es puntos, exactos y segundos de anticipación descendentes, luego ID de usuario ascendente. La tabla privada exige pertenecer al grupo y ordena los acumulados de sus integrantes al consultar. Un usuario con cero puntos en este partido puede conservar puntos de encuentros anteriores.
4. **Persistir.** El ranking se actualiza progresivamente por lotes. Tras una espera configurable (15 s por defecto), el trabajo vuelca resultados y acumulados en MongoDB, guarda la foto y limpia las claves temporales del encuentro. MongoDB registra los pasos y la toma del trabajo. Las señales de vida durante la puntuación permiten distinguir trabajo activo de abandonado y reintentar cierres pendientes. Con el ranking Redis desactivado, MongoDB ejecuta puntuación, suma y foto.

Redis también guarda sesiones renovables de 24 h por defecto. `POST /sesiones` entrega token y cookie HttpOnly. Los clientes de API pueden enviar `X-Sesion`. Elegir un usuario es suficiente en este laboratorio. Las operaciones administrativas requieren además `X-Admin-Clave`. Pub/Sub distribuye estado y chat. La conexión SSE renueva presencia cada 5 s y una conexión sin señal deja de contar tras 15 s. HyperLogLog estima usuarios únicos por país. El primario usa AOF y una réplica asíncrona sirve para medir el atraso. La API opera contra el primario. La pérdida de Redis afecta las sesiones aun con el ranking configurado en MongoDB.

## InfluxDB · vivo, análisis y retención

Las simulaciones nuevas usan dos bases separadas:

| Base | Contenido | Retención |
|---|---|---|
| `fixture2030_raw` | `estadisticas_jugador_minuto` y `jugadas_partido` de cada corrida; detalle usado durante el vivo y para validar el cierre. | 7 días desde el timestamp de cada dato, por defecto. |
| `fixture2030_resumenes` | `resumen_partido_equipo` y `resumen_partido_jugador`, con claves por partido y corrida. | Sin vencimiento configurado. |

Los resúmenes se materializan al finalizar; las muestras originales no se borran inmediatamente. La retención es temporal, no depende de que el partido se haya archivado, y la liberación física depende de InfluxDB. Si el archivo falla, debe corregirse y reintentarse antes de que venzan las muestras: después podría ser imposible reconstruir el histórico. El vencimiento real no se esperó durante la prueba.

La base anterior `fixture2030_metrics` no se modifica ni migra automáticamente. Las corridas anteriores siguen leyendo de ella; sus resúmenes se escriben en la base nueva si se reintenta el archivo mientras el detalle siga disponible. En el entorno local esa base anterior no tenía retención. No asumir lo mismo para otras instalaciones.

`INFLUX_RAW_DB`, `INFLUX_RESUMEN_DB` e `INFLUX_RAW_RETENCION` configuran nombres y duración al crear las bases. Cambiar la variable de retención no altera una base ya creada. Los datos crudos se identifican también por `corrida_id`; el ID de evento se guarda como campo. La posesión se obtiene de segundos por jugador y las acciones se agregan por jugador, equipo y minuto. Un gol cuenta como tiro y tiro al arco; córners permanecen en cero mientras no se generen. Son datos sintéticos.

La base de resúmenes también conserva `audiencia_partido`, `comentarios_partido` y `audiencia_resumen`. Al cerrar cada minuto se toman presencia y contadores de Redis. Al cierre del partido se guardan los espectadores únicos estimados. Los endpoints de análisis permiten consultar por país y tramo y comparar corridas de demo o carga. Las consultas por partido acotan el tiempo para limitar los archivos que abre InfluxDB. El promedio entre partidos tiene un costo creciente.

Para archivar se exigen muestras del plantel para cada minuto 0–90, eventos coincidentes con el guion y goles coherentes con el resultado. El grafo recibe eventos, resúmenes de jugador y equipo y agregados de equipo por minuto. Si falla, `archivo_estado` queda pendiente. Administración puede repetir `POST /administracion/simulaciones/{id}/archivar` mientras exista el detalle íntegro. La puntuación y el archivo deportivo son trabajos independientes. Una corrida interrumpida durante el juego carece de reanudación automática completa.

## Cassandra · comentario, entrega e historial

El país se obtiene del usuario de MongoDB. El POST exige sesión, partido en juego, UUID del cliente y texto de 1 a 280 caracteres. Redis reserva el UUID durante 10 min y aplica modo lento por usuario (5 s por defecto). Un envío nuevo devuelve 201, una repetición ya completada devuelve 200 y uno simultáneo o demasiado rápido devuelve 429. Cassandra conserva el texto antes de distribuirlo por Redis a la tribuna del país y al canal global. Son escrituras separadas.

La clave de partición es `(partido_id, pais, ventana)` con ventanas UTC reales de media hora. Fecha e ID ordenan filas. Los dos datacenters del laboratorio, América y Europa, usan consistencia `LOCAL_ONE` y el adaptador intenta la otra región ante fallas regionales. El historial global descubre países en Cassandra y contempla partidos antiguos sin índice de países.

La pantalla carga historial y recibe novedades por `GET /partidos/{id}/vivo` (SSE). Para recuperar una página se usa `GET /partidos/{id}/comentarios?pais=ARG&limit=20` o `alcance=todos`. El campo `siguiente` se envía como `antes_de`. La consulta explora hasta seis ventanas y extiende la página si varios comentarios empatan en el instante de corte. En partidos terminados empieza desde el cierre. Los textos se conservan sin TTL. También existe la consulta de recientes por país.

La simulación prepara 25 hinchas por país y rota sus sesiones. Con los cuatro países base y la tasa predeterminada de 4 envíos por país y segundo durante 90 intervalos, **intenta 1.440 comentarios**, más los de países adicionales presentes al inicio. El total aceptado depende de las respuestas de la API. Los 91 lotes de muestras incluyen el minuto cero y el 90. El guion se puede preparar con `TPO/scripts/cargar_guion.ps1 -Partido P-14` y editar a partir de `TPO/seed/guion_simulacion.json`.

## Operación y evidencia

Compose reúne FastAPI, los cinco motores, inicialización de esquemas, Prometheus, exportadores y Grafana. Las dos regiones comparten una máquina. No hay transacción distribuida entre motores. Conviene distinguir el indicador de simulación completa, el de estadísticas archivadas y el de puntuación completa al interpretar una prueba.

En la medición publicada con 100.000 pronósticos, la tabla Redis se completó en 10,4 s y la persistencia y el cierre completos tardaron 127,7 s. Con MongoDB como motor de ranking, el cierre completo tardó 42,3 s. Las cifras corresponden al laboratorio; la precarga directa de usuarios, sesiones y pronósticos se excluyó de la medición. Repetir el cierre mantuvo los acumulados. [Mediciones y condiciones](https://github.com/tomaszeppo/Grupo16_BD2_repo/blob/91cae0e/TPO/docs/rendimiento.md).

El detalle de claves, índices y relaciones, las decisiones de consistencia y las rutas públicas se pueden contrastar con la misma revisión. [Modelo de datos](https://github.com/tomaszeppo/Grupo16_BD2_repo/blob/91cae0e/TPO/docs/modelo_de_datos.md) · [Decisiones por motor](https://github.com/tomaszeppo/Grupo16_BD2_repo/blob/91cae0e/TPO/docs/decisiones.md) · [API](https://github.com/tomaszeppo/Grupo16_BD2_repo/blob/91cae0e/TPO/docs/api.md).
