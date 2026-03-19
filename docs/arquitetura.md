# Arquitetura do Sistema de Monitoramento

## Visão Geral

O sistema de monitoramento roda integralmente em uma **VM dedicada** em um dos servidores da infraestrutura, utilizando **Docker Compose** para orquestrar todos os containers. A stack é composta por ferramentas open source consolidadas no mercado.

| Aspecto                  | Detalhe                                                       |
| ------------------------ | ------------------------------------------------------------- |
| **VM de monitoramento**  | IP: `IP_DA_VM`                                           |
| **Orquestração**         | Docker Compose                                                |
| **Paradigma de coleta**  | Pull (Prometheus busca os dados) + Push (Promtail envia logs) |
| **Retenção de métricas** | Padrão Prometheus (15 dias)                                   |
| **Retenção de logs**     | 30 dias (configurado no Loki)                                 |

---

## Diagrama Geral

```
+------------------------------------------------------------------+
|              VM de Monitoramento  (IP_DA_VM)                |
|                                                                  |
|  +------------+  +---------------+  +--------+  +----------+   |
|  | Prometheus |  | SNMP Exporter |  |  Loki  |  |  Grafana |   |
|  |   :9090    |  |    :9116      |  | :3100  |  |   :3081  |   |
|  +-----+------+  +------+--------+  +---+----+  +-----+----+   |
|        |                |               |              |        |
|  +-----+------+  +------+--------+      |         le de todos  |
|  |  Blackbox  |  |   postgres_   |      ^              |        |
|  |  Exporter  |  |   exporter    |      |              |        |
|  |   :9115    |  |    :9187      |      | push (HTTP)  |        |
|  +------------+  +---------------+      |              |        |
+------------------------------------------|--------------+-------+
                                           |              |
                  +------------------------+              |
                  |  Promtail (cada srv)                  |
                  |  envia logs via HTTP POST             |
                  |                                       |
+-----------------|---------------------------------------|-----------+
|                 |      Infraestrutura Monitorada        |           |
|                 v                                       v           |
|  +-------------------------------------------+  (pull HTTP :9182)  |
|  |   Servidores Windows  (windows_exporter)  |<--------------------+|
|  |   servidor-01  servidor-05  servidor-04  servidor-03  servidor-02  |                      |
|  +-------------------------------------------+                      |
|                                                                      |
|  +-------------------------------------------+  (pull SNMP UDP 161) |
|  |   Switches HPE Comware                    |<--------------------  |
|  |   Switch 01  IP_SWITCH_01              |  via snmp-exporter   |
|  |   Switch 02  IP_SWITCH_02              |                       |
|  +-------------------------------------------+                      |
|                                                                      |
|  +-------------------------------------------+  (pull SNMP UDP 161) |
|  |   Firewall Seu Firewall                  |<--------------------  |
|  |   IP_FORTIGATE                            |  via snmp-exporter   |
|  +-------------------------------------------+                      |
|                                                                      |
|  +-------------------------------------------+  (probe ICMP ping)   |
|  |   Gateways de Rede                        |<--------------------  |
|  |   rede-10  rede-11  rede-12               |  via blackbox        |
|  |   rede-13  rede-14                        |                       |
|  +-------------------------------------------+                      |
|                                                                      |
|  +-------------------------------------------+  (pull HTTP :9187)   |
|  |   Banco PostgreSQL Remoto                 |<--------------------  |
|  |   IP_BANCO_REMOTO:2407                     |  via postgres_export |
|  +-------------------------------------------+                      |
+----------------------------------------------------------------------+
```

---

## Containers em Execução

Todos os serviços abaixo rodam na VM `IP_DA_VM` via Docker Compose.

| Container           | Imagem                                  | Porta  | Função                                                 |
| ------------------- | --------------------------------------- | ------ | ------------------------------------------------------ |
| `prometheus`        | `prom/prometheus`                       | `9090` | Coleta e armazena métricas de todos os targets         |
| `grafana`           | `grafana/grafana`                       | `3081` | Interface de visualização (dashboards e logs)          |
| `snmp-exporter`     | `prom/snmp-exporter`                    | `9116` | Traduz SNMP métricas Prometheus (switches e FortiGate) |
| `loki`              | `grafana/loki:2.9.0`                    | `3100` | Recebe, indexa e armazena logs dos servidores          |
| `blackbox`          | `prom/blackbox-exporter`                | `9115` | Sonda disponibilidade via ICMP (ping) nos gateways     |
| `postgres_exporter` | `prometheuscommunity/postgres-exporter` | `9187` | Coleta métricas do banco PostgreSQL externo            |
| `n8n`               | `docker.n8n.io/n8nio/n8n`               | `5678` | Automação de fluxos e notificações                     |

> **Alertmanager** está declarado no `docker-compose.yml` mas desativado. Pode ser habilitado futuramente para envio de alertas.

---

## Fontes de Dados Monitoradas

### Servidores Windows `windows_exporter`

| Servidor | IP             | Porta | Protocolo   |
| -------- | -------------- | ----- | ----------- |
| servidor-01   | IP_SERVIDOR_01 | 9182  | HTTP (Pull) |
| servidor-02   | IP_SERVIDOR_02  | 9182  | HTTP (Pull) |
| servidor-03   | IP_SERVIDOR_03 | 9182  | HTTP (Pull) |
| servidor-04   | IP_SERVIDOR_04 | 9182  | HTTP (Pull) |
| servidor-05   | IP_SERVIDOR_05 | 9182  | HTTP (Pull) |

- O `windows_exporter` é instalado diretamente em cada servidor Windows como serviço
- O Prometheus acessa `http://IP:9182/metrics` a cada **30 segundos**
- Métricas: CPU, memória, disco, rede, serviços do Windows

### Servidores Windows Logs (Promtail Loki)

O **Promtail** também é instalado em cada servidor Windows e envia os logs do Event Log para o Loki:

| Canal       | Conteúdo                         | Filtro                                                              |
| ----------- | -------------------------------- | ------------------------------------------------------------------- |
| Application | Eventos de aplicações e serviços | Todos os eventos                                                    |
| System      | Eventos do sistema operacional   | Todos os eventos                                                    |
| Security    | Eventos de autenticação          | Apenas Event IDs críticos: 4624, 4625, 4634, 4648, 4720, 4726, 4740 |

### Switches HPE Comware SNMP

| Switch     | IP             | Módulo      | Intervalo |
| ---------- | -------------- | ----------- | --------- |
| Switch 01 | IP_SWITCH_01 | `hp_switch` | 60s       |
| Switch 02 | IP_SWITCH_02 | `hp_switch` | 60s       |

- O Prometheus solicita ao **SNMP Exporter** (`snmp-exporter:9116`) que consulte os switches via **UDP 161**
- Métricas: CPU, memória, temperatura, fan, PSU, interfaces, tráfego, erros, PoE

### Firewall Seu Firewall SNMP

| Dispositivo   | IP           | Módulo      | Intervalo |
| ------------- | ------------ | ----------- | --------- |
| Seu Firewall | IP_FORTIGATE | `fortigate` | 3 minutos |

- Métricas: CPU, memória, sessões ativas, interfaces

### Gateways de Rede ICMP (Blackbox Exporter)

| Rede    | Gateway      | Intervalo |
| ------- | ------------ | --------- |
| rede-10 | IP_FORTIGATE | 15s       |
| rede-11 | IP_GATEWAY_REDE_02 | 15s       |
| rede-12 | IP_GATEWAY_REDE_03 | 15s       |
| rede-13 | IP_GATEWAY_REDE_04 | 15s       |
| rede-14 | IP_GATEWAY_REDE_05 | 15s       |

- O Prometheus solicita ao **Blackbox Exporter** que envie um **ICMP ping** ao gateway
- Monitora disponibilidade e latência de cada rede

### Banco PostgreSQL `postgres_exporter`

| Banco  | Endereço       | Porta |
| ------ | -------------- | ----- |
| CPlus5 | IP_BANCO_REMOTO | 2407  |

- O `postgres_exporter` conecta diretamente ao banco e expõe métricas em `postgres_exporter:9187`
- O Prometheus coleta essas métricas internamente (sem saída da VM)

---

## Fluxos de Coleta por Protocolo

### Pull via HTTP Servidores Windows e PostgreSQL

```
Prometheus  --(GET :9182/metrics)-->  windows_exporter (cada servidor)
Prometheus  --(GET :9187/metrics)-->  postgres_exporter (interno Docker)
```

### Pull via HTTP com intermediário Switches e FortiGate (SNMP)

```
Prometheus  --(GET :9116/snmp?target=IP&module=X)-->  snmp-exporter

                                                    (UDP 161 / SNMP)

                                                      Switch / FortiGate
```

### Pull via HTTP com intermediário Gateways (ICMP)

```
Prometheus  --(GET :9115/probe?target=IP&module=icmp)-->  blackbox-exporter

                                                           (ICMP ping)

                                                             Gateway
```

### Push via HTTP Logs dos servidores Windows

```
Promtail (cada servidor)  --(POST :3100/loki/api/v1/push)-->  Loki
```

---

## Dashboards Disponíveis no Grafana

| Dashboard             | Arquivo                                         | Datasource |
| --------------------- | ----------------------------------------------- | ---------- |
| Windows Server Status | `dashboards/servers/windows-server-status.json` | Prometheus |
| Switch HPE Comware    | `dashboards/network/switch-hpe.json`            | Prometheus |
| FortiGate Firewall    | `dashboards/firewall/fortigate.json`            | Prometheus |
| Logs Windows Event    | `dashboards/logs/logs-ServerEvent.json`         | Loki       |
| Blackbox / Gateways   | `dashboards/blackbox/blackbox.json`             | Prometheus |
| PostgreSQL            | `dashboards/database/postgresql.json`           | Prometheus |

Acesse o Grafana em: `http://IP_DA_VM:3081`

---

## Volumes Docker (Dados Persistentes)

| Volume            | Container | Conteúdo                                               |
| ----------------- | --------- | ------------------------------------------------------ |
| `grafana-storage` | grafana   | Dashboards, datasources, usuários                      |
| `loki-data`       | loki      | Logs armazenados (retidos por 30 dias)                 |
| `n8n_data`        | n8n       | Fluxos de automação e credenciais                      |
| `portainer_data`  | portainer | Dados do Portainer (declarado mas container não ativo) |

---

## Documentação Relacionada

| Documento                | Caminho                                      |
| ------------------------ | -------------------------------------------- |
| Instalação da stack      | `docs/instalacao.md`                         |
| Configurações detalhadas | `docs/configuracao.md`                       |
| Switches HPE Comware     | `docs/info/switches/hpe-comware.md`          |
| Firewall FortiGate       | `docs/info/firewall/fortigate.md`            |
| Servidores Windows       | `docs/info/servers/windows-server-status.md` |
| Logs Windows Event       | `docs/info/logs/logs-ServerEvent.md`         |
| Roadmap                  | `docs/roadmap.md`                            |
