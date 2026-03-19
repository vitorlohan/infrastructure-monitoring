# Configuração do Grafana

## 📌 Visão Geral

O Grafana é responsável pela visualização das métricas coletadas pelo Prometheus e logs do Loki.

---

## 🔗 Datasources configurados

### 🔴 Prometheus

- Tipo: Prometheus
- URL: http://prometheus:9090
- Access: Server
- Default: Sim

Responsável por:

- Métricas de servidores
- Switches (SNMP)
- Firewall
- Blackbox (ping)

---

### 📜 Loki

- Tipo: Loki
- URL: http://loki:3100
- Access: Server
- Default: Não

Responsável por:

- Coleta e visualização de logs

---

## 📊 Dashboards

Os dashboards são armazenados em:

/dashboards/

Importação:

1. Acessar Grafana
2. Ir em Dashboards → Import
3. Selecionar JSON

---

## ⚙️ Integração

Fluxo:

Prometheus → Grafana  
Loki → Grafana
