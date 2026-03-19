# Monitoramento de Logs  Windows Event Log (Application, System, Security)

## Objetivo

Este documento descreve o procedimento completo para:

- Subir o **Loki** na VM de monitoramento (armazenamento centralizado de logs)
- Instalar e configurar o **Promtail** em cada servidor Windows (agente coletor)
- Configurar o **Loki como datasource** no Grafana
- Consultar os logs centralizados no Grafana

---

## Como Funciona (Visão Geral)

A stack **Loki + Promtail** centraliza os logs de todos os servidores em um único lugar, permitindo buscar e filtrar eventos de todos os servidores simultaneamente pelo Grafana  sem precisar acessar cada servidor individualmente.

```
Windows Event Log  --->  Promtail (agente no servidor)  --(HTTP POST)-->  Loki (VM)  --->  Grafana
```

| Componente | Onde roda | O que faz |
|------------|-----------|-----------|
| **Promtail** | Cada servidor Windows | Lê o Event Log local e envia para o Loki |
| **Loki** | VM de monitoramento (Docker) | Recebe, indexa e armazena os logs de todos os servidores |
| **Grafana** | VM de monitoramento (Docker) | Interface para buscar e visualizar os logs |

### O que é o Event Log do Windows?

O Windows registra automaticamente eventos do sistema em três canais principais:

| Canal | Conteúdo |
|-------|----------|
| **Application** | Eventos de aplicações instaladas (erros de software, serviços) |
| **System** | Eventos do sistema operacional (drivers, hardware, inicialização) |
| **Security** | Eventos de autenticação (logins, falhas de acesso, criação de usuários) |

O Promtail lê esses canais e os encaminha ao Loki com labels que permitem filtrar por servidor (`computer`), canal (`log_type`) e nível (`level`).

---

## Fluxo Detalhado

```
PASSO 1: Evento acontece no Windows

Um serviço falha no servidor-01
         
         
Windows grava no Event Log (Application/System/Security)

PASSO 2: Promtail detecta o novo log

Promtail monitora o Event Log continuamente
         
         
Detecta nova entrada e adiciona labels:
  - job: windows_eventlog
  - log_type: Application | System | Security
  - computer: servidor-01
  - level: 1 (Critical) | 2 (Error) | 3 (Warning) | 4 (Information)
         
         
Envia para Loki via HTTP POST  http://IP_DA_VM:3100/loki/api/v1/push

PASSO 3: Loki recebe e armazena

Indexa as labels (busca rápida)
         
         
Comprime e persiste o conteúdo do log

PASSO 4: Consulta no Grafana

Grafana > Explore > Loki
Query: {log_type="Application", computer="servidor-01", level="2"}
         
         
Loki retorna os logs filtrados  Grafana exibe formatado
```

---

## Pré-requisitos

| Item | Descrição |
|------|-----------|
| VM de monitoramento | Loki rodando via Docker Compose |
| Acesso ao servidor Windows | Conta de Administrador |
| Rede liberada | TCP 9080 (Promtail) e TCP 3100 (Loki) |

---

## 1. Subir o Loki na VM de Monitoramento

O Loki já está declarado no `docker-compose.yml` do projeto. Se ainda não estiver rodando, siga os passos abaixo.

### 1.1 Verificar o arquivo de configuração

**Localização no projeto:** `config/loki/loki-config.yml`

O arquivo já está configurado com:

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

limits_config:
  retention_period: 720h   # logs retidos por 30 dias
```

> **`retention_period: 720h`**  Os logs são mantidos por 30 dias (720 horas). Após esse período são removidos automaticamente.

### 1.2 Trecho do Docker Compose

```yaml
loki:
  image: grafana/loki:2.9.0
  container_name: loki
  ports:
    - "3100:3100"
  volumes:
    - ./loki-config.yml:/etc/loki/local-config.yaml
    - loki-data:/loki
  command: -config.file=/etc/loki/local-config.yaml
  restart: unless-stopped

volumes:
  loki-data:
```

### 1.3 Subir o container

Na VM de monitoramento, dentro da pasta do projeto:

```bash
docker compose up -d loki
```

### 1.4 Verificar se o Loki está rodando

```bash
# Verificar status do container
docker compose ps loki

# Ver os últimos logs do Loki
docker logs loki --tail 20
```

O Loki está pronto quando os logs mostrarem `msg="Loki started"` e o container aparecer com status `Up`.

Teste também via HTTP:

```bash
curl http://localhost:3100/ready
```

Deve retornar: `ready`

---

## 2. Instalar o Promtail nos Servidores Windows

Repita este procedimento em **cada servidor Windows** que terá logs monitorados (servidor-01, servidor-05, servidor-04, servidor-03, servidor-02).

### 2.1 Criar a pasta de instalação

Abra o **PowerShell como Administrador** e execute:

```powershell
New-Item -ItemType Directory -Path "C:\Promtail" -Force
Set-Location "C:\Promtail"
```

### 2.2 Baixar o executável do Promtail

```powershell
# Forçar TLS 1.2 (necessário em alguns Windows Server)
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

# Baixar Promtail para Windows (versão compatível com o Loki 2.9.0)
Invoke-WebRequest `
  -Uri "https://github.com/grafana/loki/releases/download/v2.9.0/promtail-windows-amd64.exe.zip" `
  -OutFile "C:\Promtail\promtail.zip"
```

### 2.3 Extrair e renomear

```powershell
# Extrair o arquivo
Expand-Archive -Path "promtail.zip" -DestinationPath "C:\Promtail" -Force

# Renomear para facilitar o uso
Rename-Item "C:\Promtail\promtail-windows-amd64.exe" "promtail.exe"
```

### 2.4 Criar o arquivo de configuração

```powershell
New-Item -Path "C:\Promtail\promtail-config.yml" -ItemType File -Force
```

Abra o arquivo `C:\Promtail\promtail-config.yml` com o editor de texto e cole o conteúdo abaixo.

> **Importante:** substitua o IP `IP_DA_VM` pelo IP real da VM de monitoramento, caso seja diferente.

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: C:\Promtail\positions.yaml   # controla o que já foi enviado ao Loki

clients:
  - url: http://IP_DA_VM:3100/loki/api/v1/push

scrape_configs:

  # ==========================================
  #  APPLICATION  logs de aplicações
  # ==========================================
  - job_name: windows_application
    windows_events:
      use_incoming_timestamp: true
      bookmark_path: C:\Promtail\bookmark_app.xml
      eventlog_name: Application
      xpath_query: '*'
      labels:
        job: windows_eventlog
        log_type: Application
    pipeline_stages:
      - json:
          expressions:
            level: level
            computer: computer
            source: source
            eventId: eventId
            channel: channel
            message: message
            timeCreated: timeCreated
      - template:
          source: levelText
          template: '{{ if eq .level "1" }}Critical{{ else if eq .level "2" }}Error{{ else if eq .level "3" }}Warning{{ else if eq .level "4" }}Information{{ else }}Unknown{{ end }}'
      - template:
          source: log_line
          template: '{"levelText":"{{ .levelText }}","level":"{{ .level }}","computer":"{{ .computer }}","source":"{{ .source }}","eventId":"{{ .eventId }}","channel":"{{ .channel }}","message":"{{ Replace .message "\"" "\\\"" -1 }}","timeCreated":"{{ .timeCreated }}"}'
      - output:
          source: log_line
      - labels:
          computer: computer
          level: level

  # ==========================================
  #  SYSTEM  logs do sistema operacional
  # ==========================================
  - job_name: windows_system
    windows_events:
      use_incoming_timestamp: true
      bookmark_path: C:\Promtail\bookmark_sys.xml
      eventlog_name: System
      xpath_query: '*'
      labels:
        job: windows_eventlog
        log_type: System
    pipeline_stages:
      - json:
          expressions:
            level: level
            computer: computer
            source: source
            eventId: eventId
            channel: channel
            message: message
            timeCreated: timeCreated
      - template:
          source: levelText
          template: '{{ if eq .level "1" }}Critical{{ else if eq .level "2" }}Error{{ else if eq .level "3" }}Warning{{ else if eq .level "4" }}Information{{ else }}Unknown{{ end }}'
      - template:
          source: log_line
          template: '{"levelText":"{{ .levelText }}","level":"{{ .level }}","computer":"{{ .computer }}","source":"{{ .source }}","eventId":"{{ .eventId }}","channel":"{{ .channel }}","message":"{{ Replace .message "\"" "\\\"" -1 }}","timeCreated":"{{ .timeCreated }}"}'
      - output:
          source: log_line
      - labels:
          computer: computer
          level: level

  # ==========================================
  #  SECURITY  apenas eventos críticos de segurança
  # ==========================================
  - job_name: windows_security
    windows_events:
      use_incoming_timestamp: true
      bookmark_path: C:\Promtail\bookmark_sec.xml
      eventlog_name: Security
      # Filtra apenas os Event IDs relevantes de segurança
      xpath_query: '*[System[(EventID=4624 or EventID=4625 or EventID=4634 or EventID=4648 or EventID=4720 or EventID=4726 or EventID=4740)]]'
      labels:
        job: windows_eventlog
        log_type: Security
    pipeline_stages:
      - json:
          expressions:
            level: level
            computer: computer
            source: source
            eventId: eventId
            channel: channel
            message: message
            timeCreated: timeCreated
      - template:
          source: levelText
          template: '{{ if eq .level "1" }}Critical{{ else if eq .level "2" }}Error{{ else if eq .level "3" }}Warning{{ else if eq .level "4" }}Information{{ else }}Unknown{{ end }}'
      - template:
          source: log_line
          template: '{"levelText":"{{ .levelText }}","level":"{{ .level }}","computer":"{{ .computer }}","source":"{{ .source }}","eventId":"{{ .eventId }}","channel":"{{ .channel }}","message":"{{ Replace .message "\"" "\\\"" -1 }}","timeCreated":"{{ .timeCreated }}"}'
      - output:
          source: log_line
      - labels:
          computer: computer
          level: level
```

**Event IDs monitorados no canal Security:**

| Event ID | Evento |
|----------|--------|
| `4624` | Login bem-sucedido |
| `4625` | Falha de login |
| `4634` | Logoff |
| `4648` | Tentativa de login com credenciais explícitas |
| `4720` | Conta de usuário criada |
| `4726` | Conta de usuário excluída |
| `4740` | Conta bloqueada por tentativas de login |

> O canal Security pode gerar milhares de eventos por hora. O filtro por `xpath_query` garante que apenas eventos relevantes de segurança sejam coletados, evitando sobrecarga no Loki.

---

## 3. Instalar o Promtail como Serviço Windows (NSSM)

O Promtail precisa rodar continuamente em segundo plano. Para isso, ele é instalado como **serviço do Windows** usando o **NSSM** (Non-Sucking Service Manager).

### 3.1 Baixar o NSSM

```powershell
Set-Location "C:\Promtail"

[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

Invoke-WebRequest `
  -Uri "https://nssm.cc/ci/nssm-2.24-101-g897c7ad.zip" `
  -OutFile "C:\Promtail\nssm.zip"

# Extrair
Expand-Archive -Path "nssm.zip" -DestinationPath "C:\Promtail" -Force

# Confirmar extração
Get-ChildItem "C:\Promtail\nssm-2.24-101-g897c7ad\win64\"
```

### 3.2 Criar e configurar o serviço

```powershell
$nssm = "C:\Promtail\nssm-2.24-101-g897c7ad\win64\nssm.exe"

# Criar o serviço apontando para o executável e config
& $nssm install Promtail "C:\Promtail\promtail.exe" "-config.file promtail-config.yml"

# Definir diretório de trabalho (o Promtail cria os arquivos bookmark/ positions aqui)
& $nssm set Promtail AppDirectory "C:\Promtail"

# Configurar para iniciar automaticamente com o Windows
& $nssm set Promtail Start SERVICE_AUTO_START

# Iniciar o serviço imediatamente
& $nssm start Promtail
```

### 3.3 Verificar se o serviço está rodando

```powershell
Get-Service Promtail
```

Resultado esperado:

```
Status   Name      DisplayName
------   ----      -----------
Running  Promtail  Promtail
```

Verifique também se o processo está ativo:

```powershell
Get-Process promtail
```

### 3.4 Verificar se o Promtail está enviando logs

Aguarde cerca de 1 minuto e verifique no endpoint local do Promtail:

```
http://localhost:9080/targets
```

Abra no navegador do próprio servidor. Deve mostrar os 3 jobs (`windows_application`, `windows_system`, `windows_security`) com status `up`.

---

## 4. Configurar Loki como Datasource no Grafana

Este passo é feito **uma única vez** na VM de monitoramento.

1. Acesse o Grafana: `http://IP_DA_VM:3081`
2. No menu lateral, vá em **Connections  Data Sources**
3. Clique em **Add data source**
4. Selecione **Loki**
5. Em **URL**, informe: `http://loki:3100`
   > Use o nome do container (`loki`) e não o IP  os containers Docker se comunicam pelo nome na rede interna
6. Clique em **Save & Test**

Se a configuração estiver correta, aparecerá a mensagem `Data source successfully connected`.

---

## 5. Consultar Logs no Grafana

### 5.1 Usando o Explore

1. No menu lateral, clique em **Explore**
2. Selecione o datasource **Loki**
3. Use a linguagem **LogQL** para filtrar os logs

### 5.2 Exemplos de queries LogQL

```logql
# Todos os logs de erro do servidor-01
{job="windows_eventlog", computer="servidor-01", level="2"}

# Todos os erros críticos de todos os servidores
{job="windows_eventlog", level="1"}

# Logs de segurança (falhas de login) de qualquer servidor
{log_type="Security"} |= "4625"

# Logs do canal System do servidor-02
{job="windows_eventlog", log_type="System", computer="servidor-02"}

# Filtrar por texto dentro da mensagem
{job="windows_eventlog"} |= "Service Control Manager"
```

**Referência de níveis:**

| Valor do label `level` | Significado |
|------------------------|-------------|
| `1` | Critical |
| `2` | Error |
| `3` | Warning |
| `4` | Information |

### 5.3 Dashboard de logs

O dashboard de logs está em `dashboardsGrafana/logs/logs-ServerEvent.json`. Para importar:

1. No Grafana, vá em **Dashboards  Import**
2. Clique em **Upload JSON file**
3. Selecione o arquivo `logs-ServerEvent.json`
4. Confirme o datasource Loki e clique em **Import**

---

## 6. Troubleshooting

### Promtail não envia logs

- Verifique se o serviço está rodando: `Get-Service Promtail`
- Verifique os logs do Promtail em `C:\Promtail\` ou pelo Gerenciador de Serviços
- Confirme que o IP `IP_DA_VM` e a porta `3100` estão acessíveis do servidor:

```powershell
Test-NetConnection -ComputerName IP_DA_VM -Port 3100
```

O campo `TcpTestSucceeded` deve aparecer como `True`.

### Loki não recebe logs

- Verifique se o container está rodando: `docker compose ps loki`
- Verifique os logs do container: `docker logs loki --tail 30`
- Confirme que a porta `3100` está exposta: `docker compose ps` deve mostrar `0.0.0.0:3100->3100/tcp`

### Logs aparecem no Loki mas não no dashboard

- Confirme que o datasource está configurado com a URL `http://loki:3100` (nome do container, não IP)
- No Grafana Explore, teste uma query simples: `{job="windows_eventlog"}`  se retornar dados, o problema é no dashboard
- Verifique se o label `computer` no Promtail config corresponde ao nome que o servidor usa (o valor vem do próprio Windows Event Log)

### Reiniciar o Promtail após alterar a configuração

```powershell
Restart-Service Promtail
```
