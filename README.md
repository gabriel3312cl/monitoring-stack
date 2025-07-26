# monitoring-stack

Stack completo de monitoreo para Docker con Grafana, Prometheus, Node Exporter, cAdvisor, Loki y Promtail.

## 📊 Componentes

- **Grafana**: Visualización y dashboards
- **Prometheus**: Recolección y almacenamiento de métricas
- **Node Exporter**: Métricas del host (CPU, memoria, disco, red)
- **cAdvisor**: Métricas detalladas de containers Docker
- **Loki**: Almacenamiento y consulta de logs
- **Promtail**: Recolección de logs de containers

## 🚀 Inicio Rápido

### Prerrequisitos
- Docker Engine 20.10+
- Docker Compose 2.x+
- 2GB RAM mínimo disponible
- 10GB espacio en disco

### Instalación

1. **Clonar el repositorio:**
```bash
git clone https://github.com/tu-usuario/monitoring-stack.git
cd monitoring-stack
```

2. **Configurar variables de entorno (OPCIONAL):**
```bash
# Copiar el archivo de ejemplo
cp .env.example .env

# Editar con tus valores personalizados
nano .env
```

3. **Levantar el stack:**
```bash
docker-compose up -d
```

4. **Verificar que todos los servicios estén corriendo:**
```bash
docker-compose ps
```

## 🌐 Accesos

| Servicio | URL | Usuario | Contraseña |
|----------|-----|---------|------------|
| **Grafana** | http://localhost:3000 | admin | admin |
| **Prometheus** | http://localhost:9090 | - | - |
| **cAdvisor UI** | http://localhost:8080 | - | - |
| **Loki** | http://localhost:3100 | - | - |

## 📈 Dashboards Recomendados

Después de acceder a Grafana, importa estos dashboards:

1. **Docker Container & Host Metrics**: ID `10619`
2. **Docker and System Monitoring**: ID `893`
3. **cAdvisor Container Metrics**: ID `14282`
4. **Docker Prometheus Monitoring**: ID `193`
5. **Loki Docker Logs**: ID `13639`

### ¿Cómo importar dashboards?

1. En Grafana, ve a **Dashboards** → **Import**
2. Ingresa el ID del dashboard (ej: `10619`)
3. Selecciona **Prometheus** como datasource
4. Click en **Import**

## 📁 Estructura del Proyecto

```
monitoring-stack/
├── docker-compose.yml          # Configuración principal de servicios
├── prometheus.yml              # Configuración de Prometheus
├── promtail-config.yml        # Configuración de Promtail
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── prometheus.yml  # Datasources automáticos
└── README.md
```

## ⚙️ Configuración

### Variables de Entorno

El stack está preparado para usar variables de entorno. Para personalizar la configuración:

1. **Copia el archivo de ejemplo:**
```bash
cp .env.example .env
```

2. **Edita el archivo `.env` con tus valores:**
```bash
# Ejemplo: Cambiar puerto de Grafana
GRAFANA_PORT=3001

# Ejemplo: Cambiar credenciales
GRAFANA_USER=admin
GRAFANA_PASSWORD=mi-password-seguro

# Ejemplo: Cambiar retención de Prometheus
PROMETHEUS_RETENTION=30d
PROMETHEUS_RETENTION_SIZE=50GB
```

3. **Variables disponibles:**

| Variable | Descripción | Valor por defecto |
|----------|-------------|-------------------|
| `GRAFANA_PORT` | Puerto para Grafana | 3000 |
| `PROMETHEUS_PORT` | Puerto para Prometheus | 9090 |
| `CADVISOR_PORT` | Puerto para cAdvisor | 8080 |
| `LOKI_PORT` | Puerto para Loki | 3100 |
| `NODE_EXPORTER_PORT` | Puerto para Node Exporter | 9100 |
| `GRAFANA_USER` | Usuario admin de Grafana | admin |
| `GRAFANA_PASSWORD` | Contraseña admin de Grafana | admin |
| `PROMETHEUS_RETENTION` | Tiempo de retención de datos | 15d |
| `PROMETHEUS_RETENTION_SIZE` | Tamaño máximo de almacenamiento | 10GB |

**Nota:** Si no creas un archivo `.env`, se usarán los valores por defecto.

### Personalizar Retención de Datos

En `docker-compose.yml`, modifica el comando de Prometheus:
```yaml
command:
  - '--storage.tsdb.retention.time=30d'  # Retener datos por 30 días
  - '--storage.tsdb.retention.size=10GB' # Máximo 10GB
```

## 📊 Queries Útiles

### Prometheus Queries

**CPU por Container:**
```promql
rate(container_cpu_usage_seconds_total[5m]) * 100
```

**Memoria por Container:**
```promql
container_memory_usage_bytes{name!=""}
```

**Red - Bytes Recibidos:**
```promql
rate(container_network_receive_bytes_total[5m])
```

**Disco I/O:**
```promql
rate(container_fs_writes_bytes_total[5m])
```

### Loki Queries

**Logs de un container específico:**
```
{container_name="nginx"}
```

**Logs con errores:**
```
{job="containerlogs"} |= "error"
```

## 🛠️ Mantenimiento

### Backup

```bash
# Backup de Grafana
docker run --rm -v grafana-data:/data -v $(pwd):/backup alpine \
  tar czf /backup/grafana-backup-$(date +%Y%m%d).tar.gz -C /data .

# Backup de Prometheus
docker run --rm -v prometheus-data:/data -v $(pwd):/backup alpine \
  tar czf /backup/prometheus-backup-$(date +%Y%m%d).tar.gz -C /data .
```

### Actualizar Imágenes

```bash
# Detener servicios
docker-compose down

# Actualizar imágenes
docker-compose pull

# Levantar servicios
docker-compose up -d
```

### Limpiar Datos Antiguos

```bash
# Limpiar volúmenes (⚠️ CUIDADO: Esto borrará todos los datos)
docker-compose down -v
```

## 🚨 Troubleshooting

### Conflicto de puertos

Si tienes servicios usando los puertos por defecto:
```bash
# Crear archivo .env
cp .env.example .env

# Editar y cambiar los puertos
nano .env
# Por ejemplo:
# GRAFANA_PORT=3001
# PROMETHEUS_PORT=9091

# Reiniciar el stack
docker-compose down
docker-compose up -d
```

### Los containers no aparecen en cAdvisor

Verifica que cAdvisor tenga los permisos correctos:
```bash
docker logs cadvisor
```

### Grafana no se conecta a Prometheus

1. Verifica que Prometheus esté corriendo: `docker-compose ps prometheus`
2. Verifica la configuración del datasource en Grafana
3. La URL debe ser `http://prometheus:9090` (no localhost)

### No veo logs en Loki

1. Verifica que Promtail esté corriendo: `docker logs promtail`
2. Asegúrate de que los volúmenes estén montados correctamente
3. Verifica permisos en `/var/lib/docker/containers`

### Alto uso de disco

```bash
# Ver uso de volúmenes
docker system df -v

# Limpiar datos de Prometheus (perderás histórico)
docker exec -it prometheus rm -rf /prometheus/*
docker-compose restart prometheus
```

## 📝 Comandos Útiles

```bash
# Ver logs de todos los servicios
docker-compose logs -f

# Ver logs de un servicio específico
docker-compose logs -f grafana

# Reiniciar un servicio
docker-compose restart prometheus

# Ejecutar comandos en containers
docker exec -it prometheus promtool check config /etc/prometheus/prometheus.yml

# Ver métricas en tiempo real
watch -n 1 docker stats
```

## 🔒 Seguridad

Para ambientes de producción:

1. **Cambia las contraseñas por defecto**
2. **Configura HTTPS** con reverse proxy (nginx/traefik)
3. **Limita acceso** por IP o VPN
4. **Habilita autenticación** en Prometheus
5. **Configura backups automáticos**

## 🤝 Contribuir

1. Fork el proyecto
2. Crea tu feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit tus cambios (`git commit -m 'Add some AmazingFeature'`)
4. Push al branch (`git push origin feature/AmazingFeature`)
5. Abre un Pull Request

## 📄 Licencia

Este proyecto está bajo la licencia MIT. Ver el archivo `LICENSE` para más detalles.

## 🆘 Soporte

- 📧 Email: soporte@tuempresa.com
- 💬 Slack: #monitoring-help
- 📚 Wiki: https://wiki.tuempresa.com/monitoring

---

⭐ Si este proyecto te fue útil, considera darle una estrella en GitHub!