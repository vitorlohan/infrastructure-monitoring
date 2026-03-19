# Infrastructure Monitoring Stack

> Template open-source para montar um sistema completo de monitoramento de infraestrutura usando ferramentas gratuitas com Docker.
> Baseado em um projeto real implantado em ambiente corporativo.

---

## Sobre Este Template

Este repositório é um **template pronto para uso**. Clone, preencha os placeholders com os dados do seu ambiente e coloque em produção.

Tudo roda em **Docker Compose** sobre uma VM Linux (Ubuntu Server), sem necessidade de licenças pagas ou agentes proprietários.

### O que este sistema monitora

| Componente                  | Protocolo          | Ferramenta              |
|-----------------------------|--------------------|-------------------------|
| Servidores Windows          | HTTP pull          | windows_exporter        |
| Switches de rede (SNMP)     | SNMP v2c           | SNMP Exporter           |
| Firewall                    | SNMP v2c           | SNMP Exporter           |
| Banco de dados PostgreSQL   | HTTP pull          | postgres_exporter       |
| Disponibilidade de redes    | ICMP               | Blackbox Exporter       |
| Logs de eventos Windows     | HTTP push          | Promtail + Loki         |

---

## Stack Utilizada

| Ferramenta              | Função                                    | Porta |
|-------------------------|-------------------------------------------|-------|
| **Prometheus**          | Coleta e armazenamento de métricas        | 9090  |
| **Grafana**             | Dashboards e visualização                 | 3081  |
| **Loki**                | Armazenamento centralizado de logs        | 3100  |
| **SNMP Exporter**       | Tradução SNMP  métricas Prometheus       | 9116  |
| **Blackbox Exporter**   | Monitoramento de disponibilidade (ICMP)   | 9115  |
| **Postgres Exporter**   | Métricas do banco PostgreSQL              | 9187  |
| **n8n**                 | Automação de notificações e alertas       | 5678  |

---

## Como Usar Este Template

### 1. Pré-requisitos

- VM com **Ubuntu Server 22.04+**
- **Docker** + **Docker Compose v2** instalados
- Acesso SSH à VM

> Consulte [docs/instalacao.md](docs/instalacao.md) para o guia completo de instalação do Docker.

### 2. Clone o repositório

```bash
git clone https://github.com/SEU-USUARIO/infrastructure-monitoring-stack.git
cd infrastructure-monitoring-stack
```

### 3. Substitua os placeholders

Todos os valores específicos de ambiente foram marcados com **PLACEHOLDERS EM MAIÚSCULAS**. Localize e substitua pelos dados da sua infraestrutura:

| Placeholder                 | Arquivo(s)                        | Descrição                                    |
|-----------------------------|-----------------------------------|----------------------------------------------|
| `IP_DA_VM`                  | Todos os docs                     | IP da VM onde o Docker está rodando          |
| `SEU_USUARIO_DA_VM`         | docs/configuracao.md              | Usuário SSH da VM                            |
| `SUA_SENHA_AQUI`            | docs/configuracao.md              | Senha SSH da VM                              |
| `IP_SERVIDOR_01` a `05`     | config/prometheus/prometheus.yml  | IPs dos servidores Windows                   |
| `servidor-01` a `05`        | config/prometheus/prometheus.yml  | Nomes amigáveis dos servidores               |
| `IP_FORTIGATE`              | config/prometheus/prometheus.yml  | IP do firewall (também usado como gateway)   |
| `IP_SWITCH_01`              | config/prometheus/prometheus.yml  | IP do switch 01                              |
| `IP_SWITCH_02`              | config/prometheus/prometheus.yml  | IP do switch 02                              |
| `IP_GATEWAY_REDE_02` a `05` | config/prometheus/prometheus.yml  | Gateways das demais redes                    |
| `IP_BANCO_REMOTO`           | docker-compose.yml                | IP do servidor PostgreSQL                    |
| `PORTA_BANCO`               | docker-compose.yml                | Porta do PostgreSQL (padrão: 5432)           |
| `NOME_BANCO`                | docker-compose.yml                | Nome do banco de dados                       |
| `SEU_SCHEMA`                | docker-compose.yml                | Schema do banco                              |
| `USUARIO_BANCO`             | docker-compose.yml                | Usuário de leitura do PostgreSQL             |
| `SENHA_BANCO`               | docker-compose.yml                | Senha do usuário PostgreSQL                  |
| `COMMUNITY_DO_SEU_SWITCH`   | config/snmp/snmp.yml              | Community SNMP dos switches HPE              |
| `GERE-UM-UUID-ALEATORIO`    | docker-compose.yml                | Chave de criptografia do n8n (gere um UUID)  |

> **Dica:** Use `Ctrl+Shift+H` no VS Code (ou `grep -r "PLACEHOLDER"`) para localizar todos os placeholders de uma vez.

### 4. Suba a stack

```bash
docker compose up -d
```

### 5. Acesse os serviços

| Serviço    | URL                          | Credenciais padrão |
|------------|------------------------------|--------------------|
| Grafana    | http://IP_DA_VM:3081         | admin / admin      |
| Prometheus | http://IP_DA_VM:9090         |                   |
| n8n        | http://IP_DA_VM:5678         | Crie na 1ª vez     |

---

## Estrutura do Repositório

```
infrastructure-monitoring-stack/

 docker-compose.yml              # Orquestração de todos os contêineres

 config/
    prometheus/prometheus.yml   # Jobs de coleta (edite com seus IPs)
    snmp/snmp.yml               # Autenticações e módulos SNMP
    loki/loki-config.yml        # Retenção e armazenamento de logs
    blackbox/blackbox.yml       # Módulo de probe ICMP

 dashboards/                     # JSONs prontos para importar no Grafana
    servers/
    network/
    firewall/
    database/
    logs/

 docs/
     instalacao.md               # Instalação do Docker + primeira execucao
     configuracao.md             # Referência completa de configuração
     arquitetura.md              # Arquitetura e diagrama do sistema
     info/
         servers/windows-server-status.md
         network/switch-hpe.md
         firewall/fortigate.md
         logs/logs-ServerEvent.md
```

---

## Documentação Completa

| Documento | Descrição |
|-----------|-----------|
| [Instalação](docs/instalacao.md) | Como instalar Docker na VM e subir a stack do zero |
| [Configuração](docs/configuracao.md) | Todos os arquivos de configuração explicados |
| [Arquitetura](docs/arquitetura.md) | Diagrama e fluxo de dados entre os componentes |
| [Servidores Windows](docs/info/servers/windows-server-status.md) | Instalar windows_exporter e configurar coleta |
| [Switches HPE](docs/info/network/switch-hpe.md) | Configurar SNMP v2c nos switches e integrar ao stack |
| [Firewall](docs/info/firewall/fortigate.md) | Configurar SNMP no FortiGate e coletar métricas |
| [Logs de Eventos](docs/info/logs/logs-ServerEvent.md) | Instalar Promtail e centralizar logs no Loki |

---

## Comandos Úteis

```bash
# Ver status dos contêineres
docker compose ps

# Ver logs de um serviço
docker compose logs -f prometheus

# Reiniciar um serviço após alterar configuração
docker compose restart prometheus

# Recarregar config do Prometheus sem reiniciar
curl -X POST http://localhost:9090/-/reload

# Subir stack ou aplicar mudanças no compose
docker compose up -d
```

---

## Contribuindo

Encontrou algum problema ou quer sugerir melhorias? Abra uma **issue** ou envie um **pull request**. Toda contribuição é bem-vinda.

---

> Projeto baseado em stack 100% open-source. Nenhuma licença paga necessária.
