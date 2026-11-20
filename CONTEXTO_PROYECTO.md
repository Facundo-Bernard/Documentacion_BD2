# Contexto del proyecto: Prode Fixture 2030

Actualizado el 19/11/2026. Este documento describe el estado **local probado** del TPO en [Grupo16_BD2_repo](https://github.com/tomaszeppo/Grupo16_BD2_repo). Los cambios de simulación, comentarios y retención aquí descritos todavía no están publicados en ese repositorio; el código publicado puede diferir. Este repositorio contiene además la vista interactiva de arquitectura.

## Recorrido y responsabilidades

El administrador selecciona un partido programado y lo inicia desde Simulaciones. Noventa minutos simulados transcurren en aproximadamente 90 segundos reales. La vista de usuario muestra marcador, estadísticas, jugadas y comentarios de su región; una persona puede comentar mientras el partido está en juego.

| Componente | Función |
|---|---|
| FastAPI | Expone los endpoints y coordina las bases. El simulador usa HTTP para iniciar y cerrar el partido, publicar comentarios, enviar muestras y solicitar el archivo. La preparación del guion, los planteles y los metadatos internos aún usa acceso a las bases desde el servidor. |
| Neo4j | Fixture y estado operativo. Al completar el archivo conserva eventos, protagonistas y relaciones, resúmenes por equipo y jugador y agregados por minuto. La vista consulta allí las estadísticas y jugadas archivadas. |
| MongoDB | Usuarios, prodes, pronósticos, resultado, ranking, guiones y ficha de simulación (estado, minuto, marcador y estado del archivo). El cierre actual no copia el relato completo a MongoDB. No hay transacción distribuida entre bases. |
| InfluxDB 3 Core | Jugadas y muestras temporales crudas para el vivo. Al cerrar, se calculan resúmenes por equipo y jugador que se guardan en otra base Influx con retención separada. |
| Cassandra | Comentarios por partido, país y ventana real de 30 minutos; permanece disponible el historial paginado después del cierre. |
| Redis | Extensión prevista, no integrada en el TPO actual. |

La interfaz actual del TPO es HTML, CSS y JavaScript servido por FastAPI en `/demo/`. El prototipo React de los hitos y sus paneles de monitoreo son independientes.

## Simulación y cierre

1. MongoDB reserva la corrida y conserva el guion editable con ID, minuto, jugador, equipo, tipo y relaciones entre jugadas. Los planteles se resuelven contra el catálogo.
2. El simulador crea cuatro usuarios sintéticos, uno por región, y pone el partido en juego mediante la API. Publica los comentarios usando el mismo endpoint que la persona. Una muestra por jugador y minuto y las jugadas correspondientes se envían mediante el endpoint administrativo de muestras.
3. Cada segundo real representa un minuto. En ARG, BRA, ESP y URU se generan alternadamente tres y dos comentarios por segundo: 20 por región en cada intervalo de ocho segundos, aproximadamente 225 por región y 900 en total en una corrida completa. Las frases de gol se corresponden con el guion.
4. Al minuto 90 se cierra el resultado por la API y se inicia la puntuación del prode. La demo elige el equipo local como clasificado para una eliminatoria empatada; no simula penales.
5. El archivado valida las muestras de cada jugador de los minutos 0 a 90, las jugadas y los goles del guion contra el marcador. Agrega los datos de Influx y guarda resúmenes finales por equipo y jugador en Influx; Neo4j recibe resúmenes, agregados por minuto y eventos relacionados.
6. Si se completa, `archivo_estado` pasa a `completo`. Si el archivo falla queda `pendiente` y un administrador puede reintentarlo con `POST /administracion/simulaciones/{id}/archivar`. Las claves estables permiten repetir el archivo sin duplicar los resúmenes. No hay recuperación automática completa ante reinicio.

La plantilla `TPO/seed/guion_simulacion.json` permite editar tipos, minutos, lados, dorsales y referencias a jugadas anteriores. `TPO/scripts/cargar_guion.ps1 -Partido P-14` prepara un guion sin sobrescribir uno existente; el simulador también lo crea si no existe. No es necesario ejecutar `seed.run --reiniciar`, que borra datos.

## InfluxDB: datos crudos, agregados y retención

Las simulaciones nuevas usan dos bases separadas:

| Base | Contenido | Retención |
|---|---|---|
| `fixture2030_raw` | `estadisticas_jugador_minuto` y `jugadas_partido` de cada corrida; detalle usado durante el vivo y para validar el cierre. | 7 días desde el timestamp de cada dato, por defecto. |
| `fixture2030_resumenes` | `resumen_partido_equipo` y `resumen_partido_jugador`, con claves por partido y corrida. | Sin vencimiento configurado. |

Los resúmenes se materializan al finalizar; las muestras originales no se borran inmediatamente. La retención es temporal, no depende de que el partido se haya archivado, y la liberación física depende de InfluxDB. Si el archivo falla, debe corregirse y reintentarse antes de que venzan las muestras: después podría ser imposible reconstruir el histórico. El vencimiento real no se esperó durante la prueba.

La base anterior `fixture2030_metrics` no se modifica ni migra automáticamente. Las corridas anteriores siguen leyendo de ella; sus resúmenes se escriben en la base nueva si se reintenta el archivo mientras el detalle siga disponible. En el entorno local esa base anterior no tenía retención. No asumir lo mismo para otras instalaciones.

`INFLUX_RAW_DB`, `INFLUX_RESUMEN_DB` e `INFLUX_RAW_RETENCION` configuran nombres y duración al crear las bases. Cambiar la variable de retención no altera una base ya creada. Los datos crudos se identifican también por `corrida_id`; el ID de evento se guarda como campo. La posesión se obtiene de segundos por jugador y las acciones se agregan por jugador, equipo y minuto. Un gol cuenta como tiro y tiro al arco; córners permanecen en cero mientras no se generen. Son datos sintéticos.

## Comentarios y consultas

Cassandra particiona por `(partido_id, pais, ventana)` y ordena por fecha e ID. `POST /partidos/{id}/comentarios` usa la identificación de usuario y su país; requiere partido en juego. `GET /partidos/{id}/comentarios/recientes?pais=ARG` devuelve hasta 20 recientes. La pantalla consulta cada 8 segundos, muestra uno cada 0,5 segundos y limita la cola: puede omitir algunos comentarios en el vivo aunque permanezcan guardados. Veinte cada ocho segundos exceden los 16 que se pueden mostrar en ese intervalo.

Para un partido finalizado, `GET /partidos/{id}/comentarios/historial?pais=ARG` devuelve páginas de hasta 20 y `next_cursor` para recorrer ventanas anteriores. Los comentarios no tienen vencimiento automático configurado. El POST no garantiza idempotencia. La identificación `X-Usuario-Id` y la clave `X-Admin-Clave` son de laboratorio, no un sistema seguro de autenticación y roles.

Otras rutas: `GET /en-vivo/partidos`, `GET /partidos/{id}/estadisticas`, `/estadisticas/jugadores`, `/jugadas/temporales` y `/historico`. En vivo las estadísticas y jugadas usan Influx; con archivo completo usan Neo4j. `POST /administracion/simulaciones/{id}/muestras` recibe los minutos de la simulación y exige clave administrativa. El contrato completo está en `/docs`.

## Evidencia y límites

- Se probó una simulación nueva completa P-17: 90 minutos simulados, archivo completo, 2 resúmenes de equipo y 22 de jugador en la base permanente, y 31 jugadas consultadas desde Neo4j. Repetir el archivo mantuvo 22 resúmenes de jugador.
- Se verificó en los metadatos de Influx que la base cruda nueva tiene retención de siete días y la de resúmenes no tiene vencimiento. El vencimiento físico tras siete días no fue comprobado por espera real.
- Pasaron 196 pruebas unitarias en la última ejecución. Los datos de carga históricos y el Hito 12 no prueban que la aplicación soporte concurrencia de producción. El simulador genera tráfico por API, pero no equivale a una carga de espectadores simultáneos.
- Pendientes: recuperación de simulaciones interrumpidas, recuperación antes del vencimiento ante fallas prolongadas, idempotencia de comentarios, autenticación real y pruebas de carga y fallas distribuidas. La base de laboratorio no sustituye una estrategia de respaldos.

Compose levanta MongoDB con tres nodos, Neo4j, Cassandra, InfluxDB y FastAPI; el consumo y la temperatura dependen del equipo. Consultar `TPO/README.md` para la configuración, evitando publicar contraseñas o `.env`.
