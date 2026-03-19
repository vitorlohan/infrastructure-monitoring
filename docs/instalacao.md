# Instalação do Sistema de Monitoramento

## Objetivo

Este documento descreve o procedimento completo para instalar o sistema de monitoramento do zero em uma VM Ubuntu, incluindo:

- Acesso à VM via PuTTY (SSH)
- Instalação do Docker e Docker Compose
- Estrutura do projeto e organização dos arquivos
- Subida de todos os containers da stack

---

## Stacks Utilizadas

| Ferramenta | Versão | Função |
|------------|--------|--------|
| **Ubuntu Server** | 22.04 LTS | Sistema operacional da VM de monitoramento |
| **Docker Engine** | 24+ | Gerenciamento dos containers |
| **Docker Compose** | v2 (plugin) | Orquestração de múltiplos containers |
| **Prometheus** | latest | Coleta e armazenamento de métricas |
| **Grafana** | latest | Visualização de métricas e logs |
| **Loki** | 2.9.0 | Armazenamento e indexação de logs |
| **SNMP Exporter** | latest | Tradução SNMP para métricas Prometheus |
| **Blackbox Exporter** | latest | Sonda de disponibilidade via ICMP |
| **Postgres Exporter** | latest | Métricas de banco PostgreSQL |
| **n8n** | latest | Automação de fluxos e notificações |

---

## 1. Acesso à VM via PuTTY

O acesso à VM de monitoramento é feito via **SSH** através do **PuTTY**  não via Remote Desktop (mstsc), pois a VM roda Ubuntu Server (sem interface gráfica).

### 1.1 Instalar o PuTTY

Baixe em: https://www.putty.org

### 1.2 Conectar à VM

1. Abra o **PuTTY**
2. Em **Host Name (or IP address)**, informe o IP da VM: `IP_DA_VM`
3. Confirme: porta `22`, tipo de conexão `SSH`
4. Clique em **Open**
5. Na primeira conexão, aceite o alerta de chave de segurança (**Accept**)
6. Informe o **usuário** e **senha** quando solicitado

> **Dica:** Salve a sessão em **Saved Sessions** com o nome `VM-Monitoramento` e clique em **Save** para não redigitar o IP nas próximas vezes.

---

## 2. Instalação do Docker no Ubuntu

Execute todos os comandos abaixo conectado à VM via PuTTY.

### 2.1 Atualizar o sistema

```bash
sudo apt update && sudo apt upgrade -y
```

### 2.2 Instalar dependências

```bash
sudo apt install -y ca-certificates curl gnupg lsb-release
```

### 2.3 Adicionar a chave GPG oficial do Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### 2.4 Adicionar o repositório do Docker

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 2.5 Instalar o Docker Engine e o plugin Compose

```bash
sudo apt update

sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

### 2.6 Verificar a instalação

```bash
docker --version
docker compose version
```

Saída esperada (versões podem variar):
```
Docker version 24.x.x, build xxxxxxx
Docker Compose version v2.x.x
```

### 2.7 Permitir uso do Docker sem `sudo` (opcional, recomendado)

Por padrão o Docker exige `sudo`. Para usar sem:

```bash
sudo usermod -aG docker $USER
```

Depois disso, **feche e reabra a sessão SSH** (ou rode `newgrp docker`) para a permissão ter efeito.

### 2.8 Habilitar o Docker para iniciar com o sistema

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Verificar se está rodando:

```bash
sudo systemctl status docker
```

---

## 3. Estrutura do Projeto

Crie a pasta do projeto e organize os arquivos de configuração como abaixo:

```bash
mkdir ~/monitoring-system
cd ~/monitoring-system
```

Estrutura esperada:

```
monitoring-system/
|-- docker-compose.yml          # orquestração de todos os containers
|-- config/
|   |-- prometheus/
|   |   `-- prometheus.yml      # jobs de coleta do Prometheus
|   |-- snmp/
|   |   `-- snmp.yml            # modulos e auths do SNMP Exporter
|   |-- loki/
|   |   `-- loki-config.yml     # configuracao do Loki
|   `-- blackbox/
|       `-- blackbox.yml        # modulos do Blackbox Exporter
`-- dashboards/                 # JSONs dos dashboards do Grafana
```

Crie os diretórios:

```bash
mkdir -p config/prometheus config/snmp config/loki config/blackbox dashboards
```

> Os arquivos de configuração (`.yml`) devem ser criados antes de subir os containers. Consulte `docs/configuracao.md` para o conteúdo de cada um.

---

## 4. Criar o `docker-compose.yml`

Na pasta `~/monitoring-system`, crie o arquivo:

```bash
nano docker-compose.yml
```

Cole o conteúdo abaixo:

```yaml
version: "3.8"

services:

  #  Prometheus 
  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./config/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    restart: unless-stopped

  #  Grafana 
  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3081:3000"      # acesso externo via porta 3081
    restart: unless-stopped
    volumes:
      - grafana-storage:/var/lib/grafana

  #  SNMP Exporter 
  snmp-exporter:
    image: prom/snmp-exporter:latest
    container_name: snmp-exporter
    ports:
      - "9116:9116"
    volumes:
      - ./config/snmp/snmp.yml:/etc/snmp_exporter/snmp.yml
    restart: unless-stopped

  #  Loki 
  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./config/loki/loki-config.yml:/etc/loki/local-config.yaml
      - loki-data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped

  #  Blackbox Exporter 
  blackbox:
    image: prom/blackbox-exporter:latest
    container_name: blackbox
    volumes:
      - ./config/blackbox/blackbox.yml:/etc/blackbox_exporter/config.yml
    ports:
      - "9115:9115"
    restart: unless-stopped
    cap_add:
      - NET_RAW    # necessario para enviar pacotes ICMP (ping)

  #  Postgres Exporter 
  postgres_exporter:
    image: prometheuscommunity/postgres-exporter
    container_name: postgres_exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://usuario:senha@IP:PORTA/BANCO?schema=dbo"
    ports:
      - "9187:9187"

  #  n8n 
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    ports:
      - "5678:5678"
    volumes:
      - ./n8n_data:/home/node/.n8n
    restart: unless-stopped
    environment:
      - N8N_SECURE_COOKIE=false

volumes:
  grafana-storage:
  loki-data:
  n8n_data:
```

Salve com `Ctrl+O`, `Enter`, `Ctrl+X`.

> **DATA_SOURCE_NAME:** substitua `usuario`, `senha`, `IP`, `PORTA` e `BANCO` pelas credenciais reais do banco PostgreSQL monitorado. Consulte o arquivo de produção para os valores em uso.

---

## 5. Subir os Containers

### 5.1 Primeira execução

```bash
cd ~/monitoring-system

docker compose up -d
```

O Docker irá baixar todas as imagens automaticamente e iniciar os containers em segundo plano (`-d` = detached). Na primeira execução pode levar alguns minutos dependendo da velocidade da rede.

### 5.2 Verificar se todos os containers estão rodando

```bash
docker compose ps
```

Todos os serviços devem estar com status `Up`:

```
NAME               STATUS          PORTS
prometheus         Up              0.0.0.0:9090->9090/tcp
grafana            Up              0.0.0.0:3081->3000/tcp
snmp-exporter      Up              0.0.0.0:9116->9116/tcp
loki               Up              0.0.0.0:3100->3100/tcp
blackbox           Up              0.0.0.0:9115->9115/tcp
postgres_exporter  Up              0.0.0.0:9187->9187/tcp
n8n                Up              0.0.0.0:5678->5678/tcp
```

### 5.3 Verificar logs de um container específico

Se algum container não subir, verifique os logs:

```bash
docker logs NOME_DO_CONTAINER --tail 30
```

Exemplos:

```bash
docker logs prometheus --tail 30
docker logs loki --tail 30
docker logs grafana --tail 30
```

---

## 6. Acessar os Serviços

Após a stack subir, os serviços ficam disponíveis nos seguintes endereços (substituindo pelo IP da VM):

| Serviço | Endereço | Usuário padrão |
|---------|----------|----------------|
| **Grafana** | `http://IP_DA_VM:3081` | `admin` / `admin` |
| **Prometheus** | `http://IP_DA_VM:9090` |  |
| **n8n** | `http://IP_DA_VM:5678` | Criado no primeiro acesso |

> **Grafana:** na primeira vez que acessar, o sistema pedirá para trocar a senha padrão `admin`. Faça isso imediatamente antes de qualquer outra configuração.

---

## 7. Comandos Úteis do Dia a Dia

```bash
# Ver status de todos os containers
docker compose ps

# Reiniciar um container específico
docker compose restart prometheus

# Parar toda a stack
docker compose down

# Subir a stack novamente
docker compose up -d

# Ver logs em tempo real
docker compose logs -f

# Ver logs de um container específico
docker logs grafana --tail 50 -f

# Atualizar as imagens e recriar os containers
docker compose pull
docker compose up -d
```

---

## 8. Próximos Passos

Após a instalação, configure os arquivos antes de usar o sistema:

| Arquivo | O que configurar |
|---------|-----------------|
| `config/prometheus/prometheus.yml` | IPs dos servidores, switches e FortiGate |
| `config/snmp/snmp.yml` | Modules e autenticações SNMP |
| `config/loki/loki-config.yml` | Ja configurado  retenção de 30 dias |
| `config/blackbox/blackbox.yml` | Ja configurado  modulo ICMP |

Consulte `docs/configuracao.md` para o detalhe de cada arquivo de configuração.
