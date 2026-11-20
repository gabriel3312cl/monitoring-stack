# Monitoring stack

Monitoreo del VPS con Grafana, Prometheus, cAdvisor, Loki y Alloy. Adaptado del stack original para Docker/Coolify: Alloy sustituye a Promtail y recoge también las métricas del host, sin un Node Exporter separado.

## Acceso

Grafana: **https://ga.opendev.cl**. Requiere autenticación; registro y acceso anónimo desactivados. El dashboard inicial **VPS: recursos, Docker y seguridad** y sus fuentes se aprovisionan desde el Compose.

El dashboard muestra CPU, RAM y disco del servidor; tráfico entrante/saliente de `eth0`; CPU, memoria y red por contenedor; accesos SSH aceptados y fallidos; bloqueos de Fail2ban; registros de UFW y bloqueos de puertos Docker; y logs de contenedores.

El tráfico es volumen de bytes, no una captura de paquetes ni un historial de cada conexión. SSH usa los eventos reales de autenticación. Los logs del firewall son limitados en frecuencia y no representan un conteo completo de paquetes descartados. Los contenedores con red host, como Alloy, no tienen tráfico de red independiente: no se atribuye el tráfico completo del host a Alloy en el gráfico de contenedores.

## Despliegue manual en Coolify

1. Crear una aplicación desde `gabriel3312cl/monitoring-stack`, rama `main`, usando la GitHub App.
2. Elegir Docker Compose y `/docker-compose.yml`. Activar **Raw Compose** y desactivar autodeploy y previews.
3. Configurar `GRAFANA_USER` y una contraseña aleatoria y no vacía en `GRAFANA_PASSWORD`. Son variables de runtime; no variables de build. La contraseña se monta como secreto en Grafana y no se inyecta en otros servicios.
4. Asociar `https://ga.opendev.cl` al servicio `grafana`, puerto interno 3000. El Compose incluye las etiquetas de Traefik y usa la red externa `coolify`.
5. Crear el registro DNS mediante la integración de Cloudflare, con proxy activado. El proxy de Coolify debe usar un certificado válido y TLS Full (strict); en este VPS usa DNS-01.
6. Desplegar manualmente y verificar métricas, logs y autenticación antes de darlo por operativo.

No hace falta publicar ningún puerto Docker. Prometheus, Loki y cAdvisor viven en una red interna `10.91.0.0/24`. Prometheus usa `.2` y Loki `.3` para que Alloy, que usa la red del host, pueda enviar datos sin exponer receptores públicos. Comprobar que esa subred esté libre antes de usar este Compose en otro servidor.

Alloy usa la red y el PID namespace del host para medir la interfaz real del VPS. Su interfaz administrativa escucha solamente en `127.0.0.1:12345`; su estado de lectura y WAL se conserva en un volumen. El exporter Unix integrado usa la misma familia de métricas `node_*` que Node Exporter.

## Recursos y persistencia

Versiones fijadas en el Compose: Grafana 13.3.1, Prometheus 3.16.0, Loki 3.7.10, Alloy 1.22.0 y cAdvisor 0.61.1.

- Prometheus: 7 días de retención y objetivo máximo de 1 GB para bloques TSDB; WAL y archivos temporales pueden ocupar espacio adicional.
- Loki: 72 horas, con compactor y persistencia. La retención es temporal, no una cuota estricta de bytes.
- Límites de RAM: Grafana 384 MiB, Prometheus 256 MiB, Loki 256 MiB, Alloy 192 MiB y cAdvisor 128 MiB. Verificar consumo y OOM al ampliar la carga del VPS.
- Los cuatro volúmenes contienen Grafana, Prometheus, Loki y el estado de Alloy. No son un respaldo externo. No ejecutar `docker compose down -v` sobre producción.
- Los logs Docker tienen rotación de 10 MB y tres archivos por servicio.

Los dashboards están versionados y no se editan desde la UI. El usuario administrador puede usar Explore y crear dashboards propios aparte. No hay destinos de correo/Slack ni alertas externas configuradas por este despliegue.

## Logs y permisos del host

Alloy lee el journal de `ssh.service`, los eventos de kernel relacionados con UFW/Docker y `/var/log/fail2ban.log`, además de los logs de Docker. Necesita acceso de lectura al journal, a los archivos del host y a la API Docker. cAdvisor es un agente privilegiado. Ambos deben tratarse como agentes de confianza: montar un socket Docker con `:ro` no convierte su API en una API de solo lectura.

En el VPS, el firewall Docker añade registros `VPS-DOCKER-DROP` antes de descartar conexiones, limitados a seis mensajes por minuto con una ráfaga inicial de diez. La regla se conserva en `/usr/local/sbin/vps-docker-firewall`. No alterar sus permisos de entrada ni hacer intentos SSH fallidos deliberados desde la IP administrativa.

## Validación y diagnóstico

```bash
# Validar sin mostrar valores sensibles; usa una contraseña temporal solo para validar.
GRAFANA_PASSWORD=validation-only docker compose config --quiet
```

Comprobar en Grafana que las fuentes Prometheus/Loki están sanas, que llegan métricas `node_*` y `container_*` con nombres de contenedores y que existen streams `ssh`, `fail2ban`, `firewall` y `docker`. Los rates necesitan al menos dos muestras; al arrancar pueden tardar alrededor de un minuto.

La retención de tres/siete días comienza desde esta instalación. No se migró el historial de monitoreo de la HP. Para recuperar la configuración basta con este repositorio y los secretos runtime; para conservar historial se deben respaldar los volúmenes con el procedimiento adecuado de cada motor.
