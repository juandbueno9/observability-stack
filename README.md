# Observability Stack

Stack de monitoreo listo para usar con Docker Compose: **métricas, logs, alertas y disponibilidad de sitios web**, con la misma arquitectura que uso en producción para monitorear servidores Linux en AWS.

![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?logo=grafana&logoColor=white)
![Loki](https://img.shields.io/badge/Loki-F46800?logo=grafana&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)

<!-- Agrega aquí una captura: ![Dashboard](docs/dashboard.png) -->

## Arquitectura

```
                ┌──────────────┐  alertas  ┌──────────────┐
                │  Prometheus  │──────────►│ Alertmanager │──► correo
                └──────┬───────┘           └──────────────┘
          scrape       │        consulta
   ┌───────────────────┼─────────────┐    ┌──────────┐
   ▼                   ▼             │    │ Grafana  │◄── tú
node-exporter     blackbox (HTTP/SSL)│    └────┬─────┘
(CPU, RAM, disco)                    └─────────┤
                                               ▼
   contenedores ──► Promtail ──────────────►  Loki (logs)
```

| Componente | Función | Puerto |
|---|---|---|
| Prometheus | Recolecta y guarda métricas, evalúa reglas de alerta | `127.0.0.1:9090` |
| Alertmanager | Agrupa alertas y notifica por correo | `127.0.0.1:9093` |
| node-exporter | Métricas del sistema operativo | interno |
| Blackbox exporter | Disponibilidad HTTP y vencimiento de SSL | interno |
| Loki + Promtail | Logs centralizados de los contenedores (retención 30 días) | interno |
| Grafana | Visualización de métricas y logs | `127.0.0.1:3000` |

## Inicio rápido

```bash
git clone https://github.com/juandbueno9/observability-stack.git
cd observability-stack
cp .env.example .env          # cambia la clave de Grafana
docker compose up -d
docker compose ps
```

- Grafana: http://localhost:3000 (Prometheus y Loki ya vienen configurados como fuentes de datos).
  Importa el dashboard **Node Exporter Full** con el ID `1860`.
- Prometheus: http://localhost:9090/alerts
- Alertmanager: http://localhost:9093

## Alertas incluidas

| Alerta | Condición | Severidad |
|---|---|---|
| `InstanciaCaida` | Un target no responde por 2 min | critical |
| `CPUAlta` | CPU > 85 % por 10 min | warning |
| `MemoriaAlta` | RAM > 90 % por 10 min | warning |
| `DiscoCasiLleno` | < 10 % libre | critical |
| `SitioCaido` | Sondeo HTTP falla por 2 min | critical |
| `CertificadoPorVencer` | Certificado SSL vence en < 15 días | warning |

## Decisiones de seguridad

- Los puertos se publican solo en `127.0.0.1`. En un servidor, pon **Nginx con TLS** al frente de Grafana o accede por **túnel SSH** (`ssh -L 3000:localhost:3000 servidor`).
- La clave de Grafana es obligatoria y viene de `.env`, que está en `.gitignore`.
- El registro de usuarios en Grafana está deshabilitado.
- Las imágenes tienen versión fija, para que una actualización no rompa el stack sin aviso.

## Monitorear más servidores

En cada servidor instala node_exporter y agrégalo a `prometheus/prometheus.yml`:

```yaml
  - job_name: node
    static_configs:
      - targets: ["node-exporter:9100", "10.0.1.20:9100", "10.0.1.21:9100"]
```

Recarga sin reiniciar: `curl -X POST http://localhost:9090/-/reload`

## Validar la configuración

```bash
docker run --rm -v $PWD/prometheus:/etc/prometheus --entrypoint promtool \
  prom/prometheus:v2.53.0 check config /etc/prometheus/prometheus.yml
```

## Próximos pasos
- [ ] Dashboards de Grafana versionados en el repo
- [ ] Alertas de Alertmanager hacia Telegram o Slack
- [ ] Despliegue en AWS con Terraform

## Licencia
MIT
