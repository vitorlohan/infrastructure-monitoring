# Monitoramento de Servidores Windows

## Objetivo

Este documento descreve o procedimento completo para:

- Instalar o `windows_exporter` nos servidores Windows
- Restringir o acesso via Firewall do Windows apenas ao servidor de monitoramento
- Configurar o Prometheus para coleta das métricas
- Validar o funcionamento no Prometheus e no Grafana

---

## Como Funciona (Visão Geral)

```
Windows Server  ---(HTTP :9182)---  Prometheus (IP_DA_VM)  --->  Grafana
```

Diferente do SNMP (usado nos switches e FortiGate), o monitoramento de servidores Windows usa o modelo **Pull direto via HTTP**:

1. **`windows_exporter`**  agente instalado em cada servidor Windows. Roda como serviço do Windows e expõe métricas do sistema em `http://IP:9182/metrics`
2. **Prometheus**  acessa diretamente cada servidor pela porta `9182` a cada 30 segundos e armazena as métricas
3. **Grafana**  lê os dados do Prometheus e exibe no dashboard de servidores Windows

> Não há intermediário (como o SNMP Exporter) neste fluxo  o Prometheus fala diretamente com o exporter instalado no servidor.

---

## Servidores Monitorados

| Instância | IP | Porta |
|-----------|-----|-------|
| servidor-01 | IP_SERVIDOR_01 | 9182 |
| servidor-02 | IP_SERVIDOR_02 | 9182 |
| servidor-03 | IP_SERVIDOR_03 | 9182 |
| servidor-04 | IP_SERVIDOR_04 | 9182 |
| servidor-05 | IP_SERVIDOR_05 | 9182 |

> Para adicionar um novo servidor, repita os passos abaixo e inclua o IP na configuração do Prometheus (Seção 3).

---

## Pré-requisitos

| Item | Descrição |
|------|-----------|
| Acesso ao servidor | Conta de Administrador local ou de domínio |
| Firewall do Windows | Acesso ao Windows Firewall com Segurança Avançada (`wf.msc`) |
| Rede liberada | Porta TCP 9182 acessível a partir de `IP_DA_VM` |

---

## 1. Instalação do `windows_exporter`

O `windows_exporter` é um agente open source que coleta métricas do sistema operacional Windows e as expõe em formato Prometheus.

### 1.1 Baixar o instalador

Acesse a página de releases do projeto no GitHub e baixe o arquivo `.msi` da versão mais recente:

```
https://github.com/prometheus-community/windows_exporter/releases
```

Escolha o arquivo com sufixo `amd64.msi` para servidores 64-bit (padrão nos servidores Windows modernos).

### 1.2 Instalar como serviço

1. Clique com o botão direito no arquivo `.msi` e selecione **Executar como administrador**
2. Siga o assistente de instalação. Durante a instalação:
   - Mantenha a **porta padrão: `9182`**
   - Mantenha marcada a opção de **instalar como serviço do Windows**
3. Conclua a instalação

Após a instalação, o serviço `windows_exporter` estará rodando automaticamente e iniciará junto com o Windows.

### 1.3 Verificar se o serviço está ativo

Abra o **PowerShell** ou **Prompt de Comando** e execute:

```powershell
Get-Service windows_exporter
```

O status deve aparecer como `Running`.

Alternativamente, abra o **Gerenciador de Serviços** (`services.msc`) e localize `windows_exporter`.

### 1.4 Testar o endpoint localmente

Abra o navegador **no próprio servidor** e acesse:

```
http://localhost:9182/metrics
```

Deve retornar uma página com centenas de linhas de métricas no formato Prometheus, como:

```
windows_cpu_time_total{...}
windows_logical_disk_free_bytes{...}
windows_os_physical_memory_free_bytes{...}
```

Se a página carregar, a instalação foi bem-sucedida.

---

## 2. Liberação no Firewall do Windows

Por padrão, o `windows_exporter` instala uma regra de firewall que permite acesso de **qualquer IP** à porta `9182`. É obrigatório restringir esse acesso **somente ao servidor de monitoramento** (`IP_DA_VM`).

### 2.1 Abrir o Firewall com Segurança Avançada

Pressione `Win + R`, digite `wf.msc` e pressione `Enter`.

### 2.2 Localizar a regra do `windows_exporter`

No painel esquerdo, clique em **Regras de Entrada (Inbound Rules)**.

Use o campo de pesquisa ou role a lista até encontrar a regra criada pelo instalador, geralmente chamada `windows_exporter`.

### 2.3 Restringir o acesso por IP de origem

1. Clique com o botão direito na regra  **Propriedades**
2. Vá até a aba **Escopo (Scope)**
3. Na seção **Endereço IP Remoto (Remote IP Address)**:
   - Selecione **Estes endereços IP (These IP addresses)**
   - Clique em **Adicionar**
   - Informe: `IP_DA_VM`
   - Clique em **OK**
4. Certifique-se de que a seção **Endereço IP Local (Local IP Address)** está como **Qualquer endereço IP (Any IP address)**
5. Clique em **Aplicar** e **OK**

> **Por que isso é importante?** Sem essa restrição, qualquer máquina na rede que acesse `http://IP_DO_SERVIDOR:9182/metrics` verá todas as métricas do sistema operacional  incluindo informações sobre processos, discos e serviços. Restringir ao IP do Prometheus é a prática mínima de segurança.

### 2.4 Confirmar a regra

Para confirmar via PowerShell que a restrição foi aplicada:

```powershell
Get-NetFirewallRule -DisplayName "*exporter*" | Get-NetFirewallAddressFilter
```

O campo `RemoteAddress` deve mostrar `IP_DA_VM` e não `Any`.

---

## 3. Configuração do Prometheus

**Localização no projeto:** `config/prometheus/prometheus.yml`

### 3.1 Job de coleta dos servidores Windows

O job está nomeado como `prometheus` (nome que identifica o conjunto de exporters Windows no Prometheus).

```yaml
- job_name: prometheus
  scrape_interval: 30s
  static_configs:
    - targets:
        - "IP_SERVIDOR_01:9182"   # servidor-01
        - "IP_SERVIDOR_02:9182"    # servidor-02
        - "IP_SERVIDOR_03:9182"   # servidor-03
        - "IP_SERVIDOR_04:9182"   # servidor-04
        - "IP_SERVIDOR_05:9182"   # servidor-05

  relabel_configs:
    - source_labels: [__address__]
      regex: "IP_SERVIDOR_01:9182"
      target_label: instance
      replacement: servidor-01

    - source_labels: [__address__]
      regex: "IP_SERVIDOR_02:9182"
      target_label: instance
      replacement: servidor-02

    - source_labels: [__address__]
      regex: "IP_SERVIDOR_03:9182"
      target_label: instance
      replacement: servidor-03

    - source_labels: [__address__]
      regex: "IP_SERVIDOR_04:9182"
      target_label: instance
      replacement: servidor-04

    - source_labels: [__address__]
      regex: "IP_SERVIDOR_05:9182"
      target_label: instance
      replacement: servidor-05
```

### 3.2 Entendendo o `relabel_configs`

Sem o `relabel_configs`, o Prometheus usaria o IP + porta como identificador da instância nos dashboards (ex: `IP_SERVIDOR_01:9182`), o que dificulta a leitura.

O `relabel_configs` aqui faz uma coisa simples: **substitui o IP pelo nome amigável do servidor**. Assim, no Grafana aparece `servidor-01` em vez de `IP_SERVIDOR_01:9182`.

### 3.3 Adicionar um novo servidor Windows

1. **No novo servidor:** instale o `windows_exporter` e configure o firewall (Seções 1 e 2)
2. **No `prometheus.yml`:** adicione o IP em `targets` e uma entrada em `relabel_configs`:

```yaml
    - targets:
        - "192.168.10.XXX:9182"   # novo servidor

  relabel_configs:
    # ... entradas existentes ...
    - source_labels: [__address__]
      regex: "192.168.10.XXX:9182"
      target_label: instance
      replacement: srv-XX
```

3. **Reinicie o Prometheus** para aplicar:

```bash
docker compose restart prometheus
```

---

## 4. Validação

### 4.1 Testar o endpoint do exporter

No servidor de monitoramento ou em qualquer máquina com acesso liberado, acesse via navegador ou curl:

```bash
curl http://IP_SERVIDOR_01:9182/metrics
```

Deve retornar métricas no formato Prometheus. Exemplos do que esperar:

```
# HELP windows_cpu_time_total Total CPU time
windows_cpu_time_total{core="0,0",mode="idle"} 12345.67

# HELP windows_os_physical_memory_free_bytes Physical memory free
windows_os_physical_memory_free_bytes 4.29496704e+09

# HELP windows_logical_disk_free_bytes Logical disk free space
windows_logical_disk_free_bytes{volume="C:"} 5.36870912e+10
```

### 4.2 Verificar no Prometheus (interface web)

Acesse: `http://IP_DA_VM:9090`

Na barra de busca, execute:

```
up{job="prometheus"}
```

| Valor | Servidor | Significado |
|-------|----------|-------------|
| `1` | Qualquer | Coleta bem-sucedida |
| `0` | Qualquer | Falha  servidor não acessível ou exporter parado |

Para ver o status de um servidor específico:

```
up{job="prometheus", instance="servidor-01"}
```

---

## 5. Métricas Coletadas Principais

| Categoria | Métrica | Descrição |
|-----------|---------|-----------|
| **CPU** | `windows_cpu_time_total` | Tempo de CPU por modo (idle, user, kernel) |
| **Memória** | `windows_os_physical_memory_free_bytes` | Memória física livre |
| **Disco** | `windows_logical_disk_free_bytes` | Espaço livre por volume |
| **Rede** | `windows_net_bytes_total` | Tráfego de rede por interface |
| **Serviços** | `windows_service_status` | Status dos serviços do Windows |

> O `windows_exporter` coleta dezenas de outras métricas além dessas. As principais são usadas no dashboard `dashboardsGrafana/servers/windows-server-status.json`.

---

## 6. Boas Práticas

| Prática | Motivo |
|---------|--------|
| Restringir porta 9182 ao IP `IP_DA_VM` | Evita exposição de métricas do sistema para toda a rede |
| **Nunca deixar `Any IP` liberado** | Qualquer máquina na rede poderia ler métricas sensíveis |
| Padronizar nomes de instâncias no Prometheus | Facilita filtros e alertas no Grafana |
| Documentar todos os servidores monitorados | Saber quais IPs estão no `prometheus.yml` evita confusão |
| Manter `scrape_interval: 30s` | Intervalo adequado  menor aumenta carga; maior reduz granularidade |

---

## 7. Troubleshooting

### Serviço `windows_exporter` não está rodando

Reinicie o serviço via PowerShell (como administrador):

```powershell
Restart-Service windows_exporter
```

Verifique se inicia automaticamente com o Windows:

```powershell
Set-Service -Name windows_exporter -StartupType Automatic
```

### Prometheus mostra `up = 0` para um servidor

1. Confirme que o serviço está rodando no servidor: `Get-Service windows_exporter`
2. Teste o endpoint diretamente do servidor de monitoramento: `curl http://IP:9182/metrics`
3. Se o curl falhar, verifique a regra de firewall no servidor Windows  o IP `IP_DA_VM` deve estar em **Remote IP Address**
4. Verifique os logs do Prometheus: `docker compose logs prometheus`

### Endpoint acessível mas métricas não aparecem no Grafana

- Confirme que o datasource do Prometheus está configurado no Grafana
- Verifique se o dashboard importado é o correto (`dashboardsGrafana/servers/windows-server-status.json`)
- Certifique-se de que o filtro de `instance` no dashboard usa o nome configurado no `relabel_configs` (ex: `servidor-01`), não o IP
