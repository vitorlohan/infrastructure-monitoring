# Configuração do Sistema de Monitoramento

Este documento centraliza todas as informações de configuração do ambiente. Consulte-o ao adicionar novos alvos, alterar credenciais ou modificar o comportamento dos componentes.

---

## Acesso à VM de Monitoramento

| Campo     | Valor         |
| --------- | ------------- |
| IP        | IP_DA_VM |
| Usuário   | monitoramento |
| Senha     | SUA_SENHA_AQUI  |
| Porta SSH | 22            |
| Acesso    | PuTTY (SSH)   |

Após autenticar, o projeto está em:

```
~/monitoramento/
```

> **Atenção:** Não compartilhe estas credenciais fora da equipe de infraestrutura. Considere trocar a senha periodicamente.

---

## Estrutura dos Arquivos de Configuração

Todos os arquivos de configuração residem diretamente em `~/monitoramento/`, junto com o `docker-compose.yml`. Cada contêiner monta seu respectivo arquivo via volume bind.

```
~/monitoramento/
  docker-compose.yml       <- Orquestração de todos os contêineres
  prometheus.yml           <- Jobs de coleta de métricas
  snmp.yml                 <- Autenticações e módulos SNMP
  loki-config.yml          <- Configuração de armazenamento de logs
  blackbox.yml             <- Módulo de probe ICMP
  alertmanager.yml         <- Alertas (desabilitado, comentado no compose)
  n8n_data/                <- Dados persistentes do n8n
```

---

## 1. prometheus.yml Jobs de Coleta de Métricas

**Montado em:** `/etc/prometheus/prometheus.yml` (contêiner `prometheus`)

A função do Prometheus é fazer _scrape_ periódico de cada alvo e armazenar as métricas coletadas. Cada bloco `job_name` define um grupo de alvos com suas regras.

---

### Job: `prometheus` Servidores Windows

Coleta métricas de CPU, memória, disco e rede dos servidores Windows via `windows_exporter` (porta `9182`).

```yaml
- job_name: prometheus
  scrape_interval: 30s
  static_configs:
    - targets:
        - "IP_SERVIDOR_01:9182" # servidor-01
        - "IP_SERVIDOR_02:9182" # servidor-02
        - "IP_SERVIDOR_03:9182" # servidor-03
        - "IP_SERVIDOR_04:9182" # servidor-04
        - "IP_SERVIDOR_05:9182" # servidor-05
  relabel_configs:
    - source_labels: [__address__]
      regex: "IP_SERVIDOR_01:9182"
      target_label: instance
      replacement: servidor-01
    # ... demais servidores
```

| IP             | Nome Amigável |
| -------------- | ------------- |
| IP_SERVIDOR_01 | servidor-01        |
| IP_SERVIDOR_02  | servidor-02        |
| IP_SERVIDOR_03 | servidor-03        |
| IP_SERVIDOR_04 | servidor-04        |
| IP_SERVIDOR_05 | servidor-05        |

**Para adicionar um novo servidor:** inclua o IP na lista `targets` e adicione o bloco correspondente em `relabel_configs` com o nome desejado.

---

### Job: `fortigate` Firewall Seu Firewall

Coleta métricas do FortiGate via SNMP através do `snmp-exporter`. O intervalo é de **3 minutos** por ser um dispositivo mais pesado.

```yaml
- job_name: fortigate
  scrape_interval: 3m
  scrape_timeout: 3m
  metrics_path: /snmp
  params:
    module: [fortigate]
    auth: [public]
  static_configs:
    - targets:
        - IP_FORTIGATE
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: snmp-exporter:9116
```

| Parâmetro       | Valor               |
| --------------- | ------------------- |
| IP do FortiGate | IP_FORTIGATE        |
| Auth SNMP       | `public` (snmp.yml) |
| Módulo SNMP     | `fortigate`         |
| Scrape interval | 3 minutos           |

---

### Job: `blackbox-gateways` Disponibilidade de Redes (ICMP)

Verifica se os gateways de cada rede estão respondendo via ping (ICMP). Intervalo de **15 segundos**.

```yaml
- job_name: blackbox-gateways
  scrape_interval: 15s
  metrics_path: /probe
  params:
    module: [icmp]
  static_configs:
    - targets: [IP_FORTIGATE]
      labels:
        rede: rede-10
    - targets: [IP_GATEWAY_REDE_02]
      labels:
        rede: rede-11
    - targets: [IP_GATEWAY_REDE_03]
      labels:
        rede: rede-12
    - targets: [IP_GATEWAY_REDE_04]
      labels:
        rede: rede-13
    - targets: [IP_GATEWAY_REDE_05]
      labels:
        rede: rede-14
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox:9115
```

| Label   | IP Gateway   |
| ------- | ------------ |
| rede-10 | IP_FORTIGATE |
| rede-11 | IP_GATEWAY_REDE_02 |
| rede-12 | IP_GATEWAY_REDE_03 |
| rede-13 | IP_GATEWAY_REDE_04 |
| rede-14 | IP_GATEWAY_REDE_05 |

**Para adicionar uma nova rede:** inclua novo bloco `targets` com o IP do gateway e o label `rede` correspondente.

---

### Job: `postgres` Banco de Dados PostgreSQL

Coleta métricas do PostgreSQL remoto via `postgres_exporter`. O exporter conecta diretamente ao banco e expõe as métricas na porta `9187`.

```yaml
- job_name: postgres
  static_configs:
    - targets:
        - postgres_exporter:9187
```

A string de conexão está definida no `docker-compose.yml` (variável `DATA_SOURCE_NAME`). Veja a seção [docker-compose.yml](#5-docker-composeyml--variáveis-e-configurações-de-ambiente).

---

### Job: `snmp-hp-switch` Switches HPE Comware

Coleta métricas dos switches HPE via SNMP através do `snmp-exporter`. Intervalo de **60 segundos**.

```yaml
- job_name: snmp-hp-switch
  scrape_interval: 60s
  scrape_timeout: 30s
  metrics_path: /snmp
  params:
    module: [hp_switch]
    auth: [hp_auth]
  static_configs:
    - targets:
        - IP_SWITCH_01
      labels:
        instance: "Switch 01"
    - targets:
        - IP_SWITCH_02
      labels:
        instance: "Switch 02"
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [instance]
      target_label: instance
    - target_label: __address__
      replacement: snmp-exporter:9116
```

| Nome       | IP             |
| ---------- | -------------- |
| Switch 01 | IP_SWITCH_01 |
| Switch 02 | IP_SWITCH_02 |

---

## 2. snmp.yml Autenticações e Módulos SNMP

**Montado em:** `/etc/snmp_exporter/snmp.yml` (contêiner `snmp-exporter`)

Define as credenciais SNMP (`auths`) e quais OIDs serão coletados por dispositivo (`modules`).

### Autenticações

```yaml
auths:
  public:
    version: 2
    community: public # Usada pelo FortiGate

  hp_auth:
    version: 2
    community: COMMUNITY_DO_SEU_SWITCH # Usada pelos switches HPE
```

| Nome Auth | Community | Dispositivo   |
| --------- | --------- | ------------- |
| public    | public    | Seu Firewall |
| hp_auth   | COMMUNITY_DO_SEU_SWITCH   | Switches HPE  |

> Para alterar a community de um dispositivo, modifique o valor `community` aqui **e** atualize a configuração SNMP no próprio dispositivo.

### Módulos Disponíveis

| Módulo      | Dispositivo   | OIDs Principais                          |
| ----------- | ------------- | ---------------------------------------- |
| `fortigate` | Seu Firewall | CPU, memória, sessões ativas, interfaces |
| `hp_switch` | Switches HPE  | Portas, tráfego, status de interfaces    |

---

## 3. loki-config.yml Armazenamento de Logs

**Montado em:** `/etc/loki/local-config.yaml` (contêiner `loki`)

```yaml
auth_enabled: false # Sem autenticação (rede interna)

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

limits_config:
  retention_period: 720h # 30 dias de retenção de logs
```

### Parâmetros relevantes

| Parâmetro          | Valor | Descrição                                   |
| ------------------ | ----- | ------------------------------------------- |
| `auth_enabled`     | false | Sem autenticação adequado para rede interna |
| `retention_period` | 720h  | Logs mantidos por 30 dias                   |
| `http_listen_port` | 3100  | Porta de recebimento (push do Promtail)     |
| `path_prefix`      | /loki | Diretório base no volume Docker             |

**Para aumentar a retenção:** altere `retention_period` em horas (ex.: `1440h` = 60 dias) e reinicie o contêiner Loki.

**Volume de dados:** o volume `loki-data` mapeia `/loki` do contêiner para armazenamento persistente no host Docker.

---

## 4. blackbox.yml Probe ICMP

**Montado em:** `/etc/blackbox_exporter/config.yml` (contêiner `blackbox`)

```yaml
modules:
  icmp:
    prober: icmp
    timeout: 5s
```

Arquivo simples. Define o módulo `icmp` com timeout de 5 segundos por probe. Utilizado pelo job `blackbox-gateways` no Prometheus para verificar disponibilidade dos gateways.

**Para adicionar um novo tipo de probe** (ex.: HTTP), adicione um novo módulo aqui. Consulte a documentação do Blackbox Exporter para módulos disponíveis.

---

## 5. docker-compose.yml Variáveis e Configurações de Ambiente

**Localização:** `~/monitoramento/docker-compose.yml`

### Variáveis de ambiente sensíveis

#### postgres_exporter String de conexão

```yaml
environment:
  DATA_SOURCE_NAME: "postgresql://USUARIO_BANCO:SENHA_BANCO@IP_BANCO_REMOTO:PORTA_BANCO/NOME_BANCO?schema=SEU_SCHEMA"
```

| Campo    | Valor          |
| -------- | -------------- |
| Usuário  | consulta       |
| Senha    | vvs@2025@      |
| Host     | IP_BANCO_REMOTO |
| Porta    | 2407           |
| Database | CPlus5         |
| Schema   | dbo            |

> A senha contém `@` codificado como `%40` na URL. Não altere a codificação ao editar.

#### n8n Chave de criptografia

```yaml
environment:
  - N8N_SECURE_COOKIE=false
  - N8N_ENCRYPTION_KEY=GERE-UM-UUID-ALEATORIO
```

| Variável             | Descrição                                                                                                                         |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `N8N_SECURE_COOKIE`  | `false` necessário sem HTTPS                                                                                                      |
| `N8N_ENCRYPTION_KEY` | Chave que protege credenciais salvas no n8n. **Não altere após o primeiro uso** isso invalidará todas as credenciais armazenadas. |

### Portas expostas

| Serviço           | Porta Host | Porta Contêiner |
| ----------------- | ---------- | --------------- |
| prometheus        | 9090       | 9090            |
| grafana           | 3081       | 3000            |
| snmp-exporter     | 9116       | 9116            |
| loki              | 3100       | 3100            |
| blackbox          | 9115       | 9115            |
| postgres_exporter | 9187       | 9187            |
| n8n               | 5678       | 5678            |

### Volumes persistentes

| Volume            | Usado por           | Conteúdo                        |
| ----------------- | ------------------- | ------------------------------- |
| `grafana-storage` | grafana             | Dashboards, datasources, config |
| `loki-data`       | loki                | Logs armazenados (chunks)       |
| `n8n_data`        | n8n                 | Fluxos e credenciais do n8n     |
| `portainer_data`  | portainer (externo) | Config do Portainer             |

---

## 6. Alertmanager (Desabilitado)

O Alertmanager está **comentado** no `docker-compose.yml`. O arquivo `alertmanager.yml` existe mas está inativo.

Para habilitar: descomente o bloco `alertmanager` no `docker-compose.yml` e execute:

```bash
docker compose up -d alertmanager
```

---

## 7. Aplicando Mudanças de Configuração

Após editar qualquer arquivo de configuração, é necessário recarregar ou reiniciar o contêiner correspondente:

### Prometheus (suporta reload sem reiniciar)

```bash
curl -X POST http://localhost:9090/-/reload
```

### Demais contêineres (reiniciar o serviço)

```bash
# Reiniciar um contêiner específico
docker compose restart prometheus
docker compose restart snmp-exporter
docker compose restart loki
docker compose restart blackbox

# Reiniciar tudo
docker compose restart
```

### Aplicar mudanças no docker-compose.yml

```bash
docker compose up -d
```

> O `up -d` recria apenas os contêineres cujas definições foram alteradas, sem derrubar os demais.

---

## Referências

| Documento                                                                           | Conteúdo                                    |
| ----------------------------------------------------------------------------------- | ------------------------------------------- |
| [docs/instalacao.md](instalacao.md)                                                 | Instalação do Docker e primeira execução    |
| [docs/arquitetura.md](arquitetura.md)                                               | Visão geral da arquitetura e fluxo de dados |
| [docs/info/switches/switch-hpe.md](info/switches/switch-hpe.md)                     | Config SNMP nos switches HPE                |
| [docs/info/firewall/fortigate.md](info/firewall/fortigate.md)                       | Config SNMP no FortiGate                    |
| [docs/info/servers/windows-server-status.md](info/servers/windows-server-status.md) | windows_exporter nos servidores             |
| [docs/info/logs/logs-ServerEvent.md](info/logs/logs-ServerEvent.md)                 | Promtail + Loki nos servidores              |
