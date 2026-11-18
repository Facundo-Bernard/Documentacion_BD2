# Contexto del proyecto: Prode Fixture 2030

Documento para continuar el trabajo en otro chat. Actualizado el 17/11/2026.

## Repositorios

- Proyecto principal: https://github.com/tomaszeppo/Grupo16_BD2_repo. Carpeta local: `C:\Users\lparlavecchio\Documents\proyectos\BASES DE DATOS 2`. Último commit incorporado en esta conversación: `f750dc8` en `main`.
- Documentación visual: https://github.com/Facundo-Bernard/Documentacion_BD2. El archivo `index.html` describe la arquitectura propuesta y se despliega como sitio estático en Vercel. Esta carpeta contiene ese repositorio; su último commit previo a este contexto es `964c829`.
- El repositorio principal avanzó durante la conversación. Antes de desarrollar, revisar su estado y los últimos cambios; el commit remoto `28b1bcf` indica una redistribución de responsabilidades que retira IRIS. Las descripciones antiguas pueden haber quedado desactualizadas.

## Objetivo

Aplicación de prode con dos apartados:

- **Usuario:** participar en un prode general y prodes de amigos; pronosticar partidos; consultar posiciones; ver un partido en curso con estado, jugadas, estadísticas y comentarios de su región.
- **Administrador:** activar un partido y ejecutar una simulación que genere comentarios y actividad. A futuro se evaluará la carga; para la primera etapa interesa ejecutar y mostrar el recorrido.

La parte asignada al usuario es la vista del partido minuto a minuto, especialmente **Cassandra para comentarios** e **InfluxDB para estadísticas y actividad temporal**. La integración de jugadas con Neo4j se coordinará después.

## Arquitectura acordada como dirección

- **MongoDB:** base general para datos persistentes e historial. Propuesta reciente: guardar la ficha y estado general del partido, equipos, horario, sede, estado, minuto y marcador. Los datos de otras fuentes que se deban conservar también deberán trasladarse a MongoDB mediante servicios o procesos explícitos.
- **Redis:** sesiones, ranking y cálculo de puntajes del prode. Los resultados y rankings que deban conservarse se trasladan a MongoDB.
- **Cassandra:** escritura y lectura de comentarios en vivo por partido y región.
- **InfluxDB:** evolución temporal de estadísticas deportivas, jugadas o movimientos, y actividad de la app como entradas, salidas y visualizaciones. Confirmar qué campos de eventos ya existen antes de implementarlos.
- **Neo4j:** estructura del fixture y relaciones entre jugadores, partidos y jugadas; por ejemplo, una jugada que influye en otra. Si una jugada se representa en varias fuentes, acordar un ID común y responsabilidades con el compañero.
- **Grafana:** visualización de métricas, según la infraestructura de monitoreo existente.

Las transferencias a MongoDB, su frecuencia, granularidad, reintentos e idempotencia son decisiones pendientes de implementación. Ninguna base copia datos automáticamente por estar en el diagrama. Definir la fuente de verdad de cada atributo para evitar estados contradictorios.

## Estado actual del partido

La propuesta conversada es que MongoDB mantenga el estado general: ID, equipos, sede, inicio, estado (`programado`, `en_juego`, `finalizado`), minuto y marcador. InfluxDB conservaría las observaciones a lo largo del tiempo. La pantalla consultaría la API para ambos.

**Revisar primero el código actualizado:** versiones previas de la vista consultaban estado en IRIS; los cambios remotos posteriores redistribuyen responsabilidades. No crear una segunda fuente de verdad sin comprobar lo que ya se migró. MongoDB e InfluxDB no comparten automáticamente una transacción: registrar fallas y poder reintentar sin duplicar eventos.

## Simulador de partido: propuesto, pendiente

Se quiere un archivo o servicio que simule 90 minutos y genere muchos comentarios. Una propuesta para desarrollo es **90 minutos simulados en 90 segundos reales**, configurable; no quedó confirmada como requisito definitivo.

Recorrido esperado:

1. Desde el panel administrador elegir y activar un partido e iniciar la simulación.
2. Publicar estado y minuto; generar jugadas y estadísticas para InfluxDB.
3. Generar comentarios para varias regiones y escribirlos en Cassandra a una tasa configurable. Aumentar la actividad en eventos como goles es una posibilidad.
4. Al minuto 90 detener la generación y marcar el partido como finalizado.
5. Ofrecer detención manual, impedir dos simulaciones simultáneas del mismo partido y registrar interrupciones. Acordar qué ocurre al reiniciar.

El panel del Hito 13 ya posee un botón de **pico de comentarios** que ejecuta el script `Hito-13-monitoreo/scripts/carga_comentarios.py`. Es una ráfaga y tiene verificación propia; **no equivale a un simulador de 90 minutos**, no avanza el reloj ni genera jugadas/estadísticas completas.

Cassandra no necesita mantener un estado general de partido: guarda y consulta comentarios. La API o servicio del partido decide si se aceptan publicaciones. Para no romper las consultas actuales por ventana temporal, usar la hora real de inserción para las particiones de Cassandra y, si se necesita, registrar el minuto simulado como dato adicional.

## Comentarios: decisión acordada

El simulador puede generar muchos comentarios, mientras la pantalla muestra una selección de la región del usuario:

- Endpoint propuesto e implementado en Hito 10: `GET /partidos/{id}/comentarios/recientes?pais=ARG`. Devuelve hasta 20, más recientes primero.
- Cassandra usa particiones por partido, país y ventana de 30 minutos. El servicio consulta la ventana actual y, si necesita completar los 20, la anterior.
- La vista precarga hasta 20 cada **8 segundos**, encola y muestra **uno cada 0,5 segundos**, con un máximo de 20 visibles. Quita duplicados por ID y acota la cola. Si el tráfico es mayor que la capacidad de presentación, muestra una selección reciente, no todos los mensajes.
- El usuario publica mediante el POST existente; la región y el usuario se toman de la sesión en el servidor de la vista. Los reintentos usan la misma clave de idempotencia.
- El cursor del GET anterior sirve para paginar; no asumir que obtiene automáticamente solo mensajes nuevos.

Pendientes: probar con Cassandra real y al cruzar media hora; revisar si el búfer y la estrategia de descarte son apropiados; actualizar OpenAPI versionado si el contrato lo requiere. Bajo muchos usuarios, considerar compartir consultas de un mismo partido y región en el servidor en vez de una lectura por usuario.

**Corrección histórica del profesor:** la API Hito 10 usa una tabla adicional `comentarios_idempotencia` que se señaló por apartarse de los modelos previos. El endpoint reciente no resuelve esa devolución. Acordar cómo manejar reintentos sin presentar como garantizada una solución aún no adoptada.

## Lo implementado en la última conversación

En el proyecto principal, commit `f750dc8`:

- Endpoint y esquema de respuesta de comentarios recientes en `Hito-10-fixture2030-APIREST/app/routes/comentarios.py`, `app/services/comentarios.py` y `app/schemas/comentarios.py`.
- Vista React sencilla en `Hito-13-monitoreo/vista_hincha/react/`, con sesión, selección de región, partidos en juego, comentarios, precarga y POST. Se sirve en `/react` desde el servidor de la vista. La vista anterior se conservó.
- Instrucciones de compilación y ejecución en `Hito-13-monitoreo/vista_hincha/react/README.md`.
- Se compiló React y se comprobó sintaxis Python. **No se ejecutó una prueba integrada con las bases ni una carga real.** En la instalación npm se informaron cuatro vulnerabilidades de severidad alta en dependencias de desarrollo; revisar su estado actual.
- No se creó el simulador de 90 minutos ni se implementó un reloj o marcador de simulación. La pantalla React lo indica como pendiente.
- La vista previa del Hito 13 también tenía SSE y consultas periódicas; revisar qué conviene reutilizar frente a la nueva vista React.

En el proyecto principal se conservó el HTML que llegó del remoto y se guardó una versión local anterior como `arquitectura-hito-12-version-local.html`. El sitio de documentación usa su propio `index.html`; las copias pueden diferir.

## Antecedentes del Hito 12

La documentación vigente del Hito 12 reporta una medición de **50.000 INSERT directos a Cassandra** desde un cliente Python, no una prueba de la API completa. La comparación entre una base histórica y una variante de dos procesos es exploratoria: registra mejoras en throughput y p99 junto con una regresión en la latencia máxima. Para atribuir causalidad y repetibilidad faltan controles y repeticiones equivalentes.

Existe una proyección Data Mart en SQLite con comentarios extraídos de Cassandra y conteos validados. El conjunto de demostración usa un partido sintético y un minuto; actualización periódica y recuperación ante fallas siguen propuestas. La comparación con PostgreSQL es conceptual. Consultar el README actual del Hito 12 antes de reutilizar estos resultados.

## Preferencias de trabajo

El usuario pide código **acotado, legible y simple**, con rutas y vistas claras. Prefiere evitar pruebas de carga o pruebas literales innecesarias mientras se implementa. No inventar resultados; distinguir siempre código existente, hipótesis y evidencia real. Prefiere documentación directa para un equipo técnico, sin texto decorativo. No hacer cambios adicionales ni ejecutar cargas destructivas solo por aparecer como propuestas en este documento.
