# Contexto del proyecto: Prode Fixture 2030

Actualizado el 18/11/2026. Este documento describe el estado del código del TPO y distingue las propuestas que todavía faltan. El proyecto principal es [Grupo16_BD2_repo](https://github.com/tomaszeppo/Grupo16_BD2_repo); esta documentación tiene su propia [vista de arquitectura](index.html). El TPO vive en `TPO/` y es independiente de los hitos anteriores. La implementación descrita está publicada en el commit [`d51e899`](https://github.com/tomaszeppo/Grupo16_BD2_repo/commit/d51e899) del repositorio principal.

## Objetivo y recorrido

El usuario crea o elige un usuario de laboratorio, consulta fixture y prodes, pronostica y sigue un partido desde **En vivo**. En el detalle ve el marcador y minuto, estadísticas comparadas, jugadas y comentarios de su región; puede comentar mientras el partido está en juego. El administrador selecciona un partido programado y, en **Simulaciones**, inicia un recorrido de 90 minutos simulados en aproximadamente 90 segundos reales. El acceso al panel usa `X-Admin-Clave`.

El frontend actual del TPO es HTML, CSS y JavaScript servido en `/demo/` por FastAPI. La vista React de `Hito-13-monitoreo/vista_hincha/react` pertenece al prototipo de los hitos: no se trasladó ni es la interfaz actual del TPO. El panel de monitoreo del Hito 13 también es independiente. IRIS no forma parte del TPO actual.

## Bases y responsabilidades actuales

| Componente | Responsabilidad en el TPO |
|---|---|
| Neo4j | Fixture, equipos de cada partido, estado operativo, estadio, jugadores y relaciones entre jugadas. Tras el cierre conserva los eventos del guion, sus protagonistas y relaciones, más el resumen y muestras por minuto del partido archivado. Es la fuente de consulta de ese histórico archivado. |
| MongoDB | Usuarios, catálogo de equipos y jugadores, prodes, pronósticos, resultados y ranking. `guiones_simulacion` conserva un guion por partido; `simulaciones` guarda la ficha, el guion copiado, minuto, marcador, estado de la simulación y estado del archivo. |
| Cassandra | Comentarios por partido, país y ventana real de 30 minutos. La API FastAPI permite leer recientes y publicar durante un partido en juego. |
| InfluxDB 3 Core | Muestras sintéticas por equipo y minuto: posesión, tiros, tiros al arco, pases acertados e intentados, faltas, córners y goles. El resumen calcula promedio de posesión y suma las acciones. |
| Redis | Punto de extensión para sesiones y ranking; no implementado en el TPO actual. El TPO usa MongoDB para puntajes y ranking. |
| Prometheus / Grafana | La API del TPO expone `/metrics`; el monitoreo completo de los hitos es independiente. No se debe asumir que el TPO ya levanta los paneles del Hito 13. |

Neo4j gobierna la transición operativa del partido. MongoDB mantiene una ficha de simulación con una copia del estado, minuto y marcador; no hay transacción distribuida entre bases. MongoDB conserva el resultado para puntuar, y el cierre actualiza el estado y marcador en Neo4j. Las transferencias o copias adicionales a MongoDB no suceden automáticamente.

## Simulación y guion

1. El administrador elige un partido programado con ambos equipos definidos. Una reserva en MongoDB impide iniciar otra simulación del mismo partido.
2. Se obtiene o crea un guion en `guiones_simulacion`: eventos con IDs estables, minuto, tipo, jugador, equipo y referencias a jugadas anteriores. Se resuelven los jugadores desde el catálogo de MongoDB. Se copia el guion a la ficha de la simulación.
3. Se generan comentarios iniciales por región, se pone el partido en juego en Neo4j y se actualiza la ficha en MongoDB.
4. Cada segundo real avanza un minuto simulado, se actualiza el marcador, se escribe una muestra de Influx por equipo y se generan comentarios de Argentina, Brasil, España y Uruguay. Las frases de gol coinciden con los goles del guion.
5. El marcador se calcula a partir de los goles del guion. Al minuto 90 el servicio cierra el partido y ejecuta la puntuación del prode. En eliminatorias empatadas la demo elige al equipo local como clasificado; no simula penales.
6. Después se consultan InfluxDB y el guion de MongoDB. Se exige una muestra por cada minuto 0–90 de ambos equipos, sin duplicados, y se comprueban los goles y las acciones del guion. Neo4j guarda los agregados, las muestras y las jugadas con `OCURRE_EN`, `PROTAGONIZA` e `INFLUYE_EN`. Los identificadores estables y `MERGE` permiten repetir el archivado sin duplicar.

La plantilla editable es `TPO/seed/guion_simulacion.json`: tipo, minuto, lado (`local` o `visitante`), dorsal y número opcional de la jugada anterior (`previa`). Con el entorno levantado y el fixture cargado, desde `TPO`:

```powershell
.\scripts\cargar_guion.ps1 -Partido P-14
```

El comando prepara el guion en MongoDB y se puede repetir sin sobrescribirlo ni borrar datos. La simulación también lo crea automáticamente si aún no existe. Editar la plantilla solo afecta a guiones futuros. `seed.run --reiniciar` borra datos de laboratorio y no es necesario para esta carga.

Los contadores sintéticos adicionales no identifican a un jugador por cada pase o tiro; no deben presentarse como eventos individuales. No hay MVP ni una fórmula implementada.

## InfluxDB y estadísticas

Por minuto y equipo se registra una muestra con el minuto simulado y un instante real de escritura. La vista calcula: posesión promedio; tiros y tiros al arco; pases acertados y precisión (acertados / intentados); faltas y córners. Las acciones son incrementos por minuto. Los goles del guion se reflejan tanto en las muestras como en el marcador. `GET /partidos/{id}/estadisticas` lee Influx durante el vivo y Neo4j tras un archivado completo. Simulaciones antiguas que no contienen métricas nuevas muestran un guion en vez de inventar valores.

Si una escritura en Influx falla, la simulación puede seguir, pero marca estadísticas incompletas; aún no hay reintento ni reconstrucción automática de la muestra. No se archiva como completo un partido con datos incompletos. Si el traspaso al grafo falla, el partido puede finalizar y `archivo_estado` queda pendiente. Un administrador puede repetir `POST /administracion/simulaciones/{id}/archivar`. No hay recuperación automática ante reinicio del proceso ni detención manual de simulación.

## Comentarios

- Cassandra particiona `comentarios_por_partido_pais` por `(partido_id, pais, ventana)` y ordena por fecha e ID. La ventana usa la hora real; no el minuto acelerado.
- `GET /partidos/{id}/comentarios/recientes?pais=ARG` devuelve hasta 20 de la ventana actual y, si hace falta, de la anterior. `POST /partidos/{id}/comentarios` toma usuario y región desde la identificación de la solicitud; solo acepta publicaciones en un partido en juego.
- La vista consulta cada 8 segundos, elimina duplicados por ID, mantiene una cola acotada y muestra un comentario cada 0,5 segundos, hasta 20 visibles. Con tráfico elevado muestra una selección reciente; no garantiza mostrar todos los almacenados. El comentario propio se coloca al principio de la cola.
- El simulador usa nombres y mensajes sintéticos variados, con reacciones en los minutos de gol.

La identificación actual del TPO (`X-Usuario-Id`) solo verifica que el ID exista: es acceso de laboratorio, no autenticación real. La clave fija de administrador también es de demostración. El POST del TPO todavía no garantiza idempotencia ante un reintento. El Hito 10 implementa una tabla auxiliar de idempotencia, cuya adecuación al modelo fue observada por el profesor: acordar una solución antes de afirmar garantías.

## Histórico y API

- `GET /en-vivo/partidos` y `GET /en-vivo/partidos/{id}`: fichas de simulación desde MongoDB.
- `GET /partidos/{id}/estadisticas`: agregados desde InfluxDB durante el vivo y desde Neo4j cuando el archivo está completo.
- `GET /partidos/{id}/historico`: resúmenes, muestras por minuto y jugadas archivadas en Neo4j.
- `GET /partidos/{id}/eventos` y `GET /partidos/{id}/eventos/{evento_id}/cadena`: jugadas y relaciones del grafo.
- `POST /administracion/simulaciones/{id}` inicia la simulación; `GET` sobre la misma ruta informa su avance. `POST .../{id}/archivar` reintenta el archivo. Requieren clave de administración.
- La API expone su contrato completo en `/docs`.

## Ejecución y recursos

Desde `TPO`, crear `.env` a partir de `.env.example`, configurar las tres claves y ejecutar `./scripts/levantar.ps1`; luego cargar el fixture con `docker compose run --rm herramientas python -m seed.run --dias-hasta-inicio 3`. Consultar `TPO/README.md` para requisitos y comandos. No publicar `.env` ni contraseñas.

Compose levanta MongoDB con tres nodos, Neo4j, Cassandra, InfluxDB y FastAPI. InfluxDB usa un volumen independiente, sin puerto publicado al host, con autenticación deshabilitada para este laboratorio y límite predeterminado de 0,5 CPU / 1 GB. No es configuración para un entorno compartido. Los límites de CPU y memoria son configurables. Para reducir el consumo conviene detener las pilas de hitos que no se necesiten; `docker compose stop` conserva los volúmenes. El inicio de varias bases puede exigir muchos recursos y los límites no garantizan una temperatura determinada.

## Evidencia y pendientes

- En pruebas locales se ejecutó una simulación real de 90 segundos, se guardaron y leyeron comentarios en Cassandra y se consultaron estadísticas en InfluxDB. Para P-13 se comprobó en Neo4j el histórico con 182 muestras, seis jugadas y la cadena Pase → Tiro → Gol; repetir el archivado conservó los mismos conteos.
- Pasaron 183 pruebas unitarias en la última ejecución registrada. No se debe extrapolar estos resultados a carga de producción, alta concurrencia o fallas distribuidas.
- Faltan pruebas de recuperación ante caídas, transición de ventana Cassandra y de integración en condiciones de carga; reintentos de escritura idempotentes; una estrategia para retomar simulaciones interrumpidas; autenticación y roles reales; políticas de retención e historial de comentarios; y observabilidad integrada de Cassandra e InfluxDB en el health del TPO.
- Los procesos de transferencia adicionales hacia MongoDB, la integración definitiva con Redis y el monitoreo completo siguen pendientes. Deben acordarse fuentes de verdad y recuperación antes de implementarlos.

## Antecedentes y preferencias

El Hito 12 midió 50.000 inserciones directas en Cassandra; esa medición no prueba la API del TPO. Su Data Mart en SQLite es una proyección de demostración con datos sintéticos. El usuario prefiere código simple y legible, bajo consumo de recursos, documentación directa y evitar pruebas de carga innecesarias; distinguir implementación, propuesta y evidencia.
