# Reporte de Laboratorio: Stack de Observabilidad


## Descripción de la Actividad
Este proyecto consiste en la implementación de un stack completo de observabilidad para una arquitectura basada en contenedores. El objetivo principal fue desplegar, configurar y conectar herramientas de monitoreo para recopilar métricas y logs de aplicaciones, visualizar el estado del sistema en tiempo real y automatizar notificaciones ante incidentes.


Se implementó un ciclo cerrado de alertas donde el sistema de monitoreo no solo detecta anomalías, sino que notifica automáticamente a la propia aplicación mediante webhooks.


## Actividades Realizadas y Cumplimiento de Entregables


A continuación se detallan las actividades ejecutadas, alineadas directamente con los criterios de evaluación del laboratorio:


* **Stack levantado y todos los servicios accesibles:**
  Se orquestó la infraestructura completa utilizando Docker Compose. Se configuraron y levantaron con éxito múltiples contenedores, asegurando la accesibilidad y comunicación entre los servicios de aplicación (frontend y backend) y las herramientas de observabilidad (Prometheus, Loki, Grafana, cAdvisor). Adicionalmente, se automatizó la configuración de las fuentes de datos (Data Sources) como código para un aprovisionamiento directo.


* **Dashboard con panel de CPU por contenedor con umbral en 50%:**
  Se integró cAdvisor y Prometheus para recolectar métricas de consumo de recursos aislados. Con esta data, se creó un panel específico en Grafana utilizando PromQL para visualizar el uso de CPU a nivel de contenedor, estableciendo visualmente un umbral crítico de límite al 50%.


* **Dashboard con panel de CPU del host:**
  Se configuró la recolección de métricas a nivel de máquina host. Posteriormente, se diseñó un panel en el dashboard de Grafana que permite monitorear el consumo de CPU global de la infraestructura física o virtual, diferenciándolo del consumo individual de los contenedores.


* **Panel de logs de aplicación funcionando con filtro por nivel:**
  Mediante la configuración de Loki y el agente recolector, se centralizaron los logs generados por el código del frontend y backend. Se construyó un panel en Grafana que muestra estos registros en tiempo real, implementando filtros que permiten discriminar la información según su nivel de severidad (INFO, WARN, ERROR, etc.).


* **Panel de logs de infraestructura funcionando:**
  Se habilitó un panel dedicado exclusivamente para la lectura de los eventos y registros internos generados por las propias herramientas del stack y Docker, manteniendo una separación clara entre los eventos de la aplicación y los de la infraestructura subyacente.


* **Alarma de CPU > 50% configurada correctamente:**
  Se definió una regla de alerta en Grafana basada en la métrica de CPU del contenedor del backend. La condición evalúa de forma constante si el uso de procesamiento supera el límite establecido del 50%.


* **Evidencia de la alarma en estado Firing tras generar carga:**
  Para validar la regla anterior, se ejecutó una prueba de estrés contra la aplicación mediante peticiones continuas (carga sostenida por más de 30 segundos). Esto permitió superar el periodo de gracia (pending period) y observar cómo el sistema cambiaba exitosamente el estado de la alarma a Firing.


* **Ciclo cerrado alarma -> log vía webhook:**
  Se configuró un Contact Point de tipo Webhook apuntando a la ruta del backend. Al dispararse la alarma en estado Firing, Grafana emitió automáticamente una petición HTTP POST. El backend recibió este evento y registró un log automático, cerrando el ciclo de notificación sin intervención humana.


---


## Comandos Principales


* `docker-compose up -d`
  **Descripción:** Lee el archivo docker-compose.yml, construye las imágenes si es necesario y levanta todos los contenedores del stack en segundo plano, permitiendo seguir utilizando la terminal.


* `docker-compose down`
  **Descripción:** Detiene la ejecución de los contenedores del stack y los elimina de la red de Docker, liberando los puertos y recursos.


* `docker compse down"`
  **Descripción:**  detener (conserva dashboards de Grafana)


  `docker compse down -v`
  **Descripción:**  detener y borrar datos (reset total)


  `docker compse ps`
  **Descripción:**  Para verificar que todo lo que creamos con anterioridad se haya creado correctamente.
---


## Respuestas a las Preguntas de Evaluación


**1. ¿Por qué necesitamos Loki además de Prometheus si ya tenemos /metrics?**
Porque cumplen funciones distintas y complementarias. Prometheus se encarga exclusivamente de datos cuantitativos (métricas temporales, contadores, porcentajes de CPU, memoria), lo que nos permite saber cuándo ocurre un problema y su magnitud. Loki, por su parte, gestiona datos cualitativos (logs, texto, trazas de errores). Mientras Prometheus te avisa que hay un pico de errores 500, Loki te permite leer el texto exacto de la excepción en el código para entender por qué ocurrió.


**2. ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?**
Aporta automatización, reproducibilidad y control de versiones (GitOps). Si los contenedores se destruyen o se necesita desplegar el entorno en otro servidor, las conexiones a Prometheus y Loki se establecen automáticamente al levantar el stack. Esto elimina la necesidad de configurar conexiones manualmente a través de la interfaz web, reduce errores humanos y acelera los tiempos de despliegue y recuperación ante desastres.


**3. El panel "CPU contenedor" y el panel "CPU host" pueden mostrar valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?**
Muestran valores distintos porque la "CPU host" mide el consumo total de la máquina física (o máquina virtual) completa, la cual puede tener múltiples núcleos y mucha capacidad. La "CPU contenedor" mide únicamente el consumo del proceso aislado dentro de Docker. Un contenedor puede estar al 100% de su capacidad límite, pero representar solo un 2% del uso total del host.
Para alertar sobre una aplicación concreta se debe utilizar la CPU contenedor, ya que refleja fielmente si ese servicio específico está saturado o en problemas, independientemente de si el servidor general tiene recursos de sobra.


**4. ¿Qué diferencia hay entre el evaluation interval y el pending period de una alarma?**
* **Evaluation interval (Intervalo de evaluación):** Es la frecuencia con la que Grafana ejecuta la consulta (query) para revisar el estado del sistema. Por ejemplo, si está configurado a 10 segundos, Grafana verificará el porcentaje de CPU cada 10 segundos.
* **Pending period (Periodo de espera/gracia):** Es el tiempo continuo que la condición de alerta debe mantenerse superando el umbral antes de enviar la notificación real. Si la condición se cumple, la alerta entra en estado Pending. Si el problema persiste durante todo ese periodo (por ejemplo, 30 segundos continuos), pasa a estado Firing y se envía el aviso. Sirve para evitar falsos positivos causados por picos breves y momentáneos.
