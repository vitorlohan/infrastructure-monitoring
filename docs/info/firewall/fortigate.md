# Monitoramento de Firewall FortiGate (80F)

## Objetivo

Este documento descreve o procedimento completo para:

- Configurar SNMP v2c no firewall Seu Firewall via interface web
- Integrar o FortiGate com o SNMP Exporter
- Coletar métricas via Prometheus
- Visualizar os dados no Grafana

---

## Como Funciona (Visão Geral)

```
FortiGate  <--(UDP 161 / SNMP)--  SNMP Exporter  <--(HTTP /metrics)--  Prometheus  -->  Grafana
```

1. **FortiGate** expõe dados via protocolo SNMP (CPU, memória, sessões ativas e interfaces)
2. **SNMP Exporter** recebe requisições do Prometheus, consulta o firewall via SNMP e traduz os dados para o formato Prometheus
3. **Prometheus** coleta as métricas a cada 3 minutos e armazena no banco de dados de séries temporais
4. **Grafana** lê os dados do Prometheus e exibe nos dashboards visuais

> O FortiGate utiliza MIBs proprietárias da Fortinet (OID base: `1.3.6.1.4.1.12356`) para expor métricas específicas do equipamento, além das MIBs padrão de rede (RFC 1213 / IF-MIB).

---

## Pré-requisitos

| Item                | Descrição                                                            |
| ------------------- | -------------------------------------------------------------------- |
| Acesso ao FortiGate | Acesso à interface web de administração                              |
| Firewall liberado   | UDP 161 aberto entre o servidor de monitoramento e o IP do FortiGate |
| Stack em execução   | Docker Compose rodando (`docker compose up -d`)                      |

---

## 1. Configuração SNMP no FortiGate

A configuração do SNMP no FortiGate é feita pela **interface web de administração**, diferente dos switches HPE que usam CLI.

### 1.1 Acessar o painel de administração

Acesse o FortiGate pelo navegador:

```
https://IP_DO_FORTIGATE
```

Faça login com as credenciais de administrador.

### 1.2 Navegar até as configurações de SNMP

No menu lateral, acesse:

```
System  SNMP
```

### 1.3 Habilitar SNMP v2c

1. Clique no botão para habilitar o **SNMP Agent**
2. Clique em **Create New** dentro da seção **SNMP v1/v2c**
3. Configure os campos:

| Campo              | Valor                           | Descrição                                                                 |
| ------------------ | ------------------------------- | ------------------------------------------------------------------------- |
| **Community Name** | `public`                        | Senha de leitura SNMP deve corresponder ao configurado no `snmp.yml`      |
| **Hosts**          | IP do servidor de monitoramento | Restringe quem pode consultar o SNMP                                      |
| **Queries**        | Enabled, porta `161`            | Habilita respostas a consultas SNMP                                       |
| **Traps**          | Opcional                        | Notificações proativas do equipamento (não utilizado neste monitoramento) |

4. Clique em **OK** para salvar

> **Sobre a community `public`:** No FortiGate este valor é usado por compatibilidade e o acesso está restrito ao IP do servidor configurado em **Hosts**. Consulte a seção de Boas Práticas para reforço de segurança.

### 1.4 Verificar a configuração via CLI (opcional)

Para confirmar via CLI do FortiGate:

```
get system snmp community
```

O retorno deve mostrar a community configurada com o IP de host permitido.

---

## 2. Configuração do SNMP Exporter (Docker)

O SNMP Exporter é o mesmo serviço usado para os switches ele suporta múltiplos módulos no mesmo arquivo de configuração.

### 2.1 Trecho do Docker Compose

```yaml
snmp-exporter:
  image: prom/snmp-exporter:latest
  container_name: snmp-exporter
  ports:
    - "9116:9116"
  volumes:
    - ./config/snmp/snmp.yml:/etc/snmp_exporter/snmp.yml
  restart: unless-stopped
```

O mesmo container atende tanto o FortiGate quanto os switches HP. A diferença está no **módulo** informado na requisição.

### 2.2 Arquivo de configuração: `snmp.yml`

**Localização no projeto:** `config/snmp/snmp.yml`

O FortiGate utiliza a autenticação `public` (definida na seção `auths`) e o módulo `fortigate` (definido na seção `modules`).

```yaml
auths:
  public:
    version: 2
    community: public

modules:
  fortigate:
    timeout: 20s
    retries: 2
    walk:
      # ===== Sistema =====
      - 1.3.6.1.2.1.1.5 # sysName     nome do dispositivo
      - 1.3.6.1.2.1.1.3 # sysUpTime   tempo ativo

      # ===== CPU / Memória / Sessões =====
      - 1.3.6.1.4.1.12356.101.4.1.3 # fgSysCpuUsage   uso de CPU (%)
      - 1.3.6.1.4.1.12356.101.4.1.4 # fgSysMemUsage   uso de memória (%)
      - 1.3.6.1.4.1.12356.101.4.1.8 # fgSysSesCount   sessões ativas

      # ===== Interfaces =====
      - 1.3.6.1.2.1.2.2.1.2 # ifDescr         nome da interface
      - 1.3.6.1.2.1.2.2.1.5 # ifSpeed         velocidade
      - 1.3.6.1.2.1.2.2.1.8 # ifOperStatus    status operacional
      - 1.3.6.1.2.1.31.1.1.1.6 # ifHCInOctets   bytes recebidos
      - 1.3.6.1.2.1.31.1.1.1.10 # ifHCOutOctets  bytes enviados

    metrics:
      # ==========================================
      #  CPU
      # ==========================================
      - name: fgSysCpuUsage
        oid: 1.3.6.1.4.1.12356.101.4.1.3
        type: gauge
        help: Current CPU usage (percentage)

      # ==========================================
      #  MEMÓRIA
      # ==========================================
      - name: fgSysMemUsage
        oid: 1.3.6.1.4.1.12356.101.4.1.4
        type: gauge
        help: Current memory usage (percentage)

      # ==========================================
      #  SESSÕES ATIVAS
      # ==========================================
      - name: fgSysSesCount
        oid: 1.3.6.1.4.1.12356.101.4.1.8
        type: gauge
        help: Number of active sessions

      # ==========================================
      #  SISTEMA
      # ==========================================
      - name: sysName
        oid: 1.3.6.1.2.1.1.5
        type: DisplayString
        help: System name

      - name: sysUpTime
        oid: 1.3.6.1.2.1.1.3
        type: gauge
        help: System uptime in hundredths of seconds

      # ==========================================
      #  INTERFACES
      # ==========================================
      - name: ifDescr
        oid: 1.3.6.1.2.1.2.2.1.2
        type: DisplayString
        help: Interface description/name
        indexes:
          - labelname: ifIndex
            type: gauge

      - name: ifSpeed
        oid: 1.3.6.1.2.1.2.2.1.5
        type: gauge
        help: Interface speed in bits per second
        indexes:
          - labelname: ifIndex
            type: gauge

      - name: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
        help: Interface operational status (1=up, 2=down)
        indexes:
          - labelname: ifIndex
            type: gauge

      - name: ifHCInOctets
        oid: 1.3.6.1.2.1.31.1.1.1.6
        type: counter
        help: Total bytes received on interface
        indexes:
          - labelname: ifIndex
            type: gauge

      - name: ifHCOutOctets
        oid: 1.3.6.1.2.1.31.1.1.1.10
        type: counter
        help: Total bytes sent on interface
        indexes:
          - labelname: ifIndex
            type: gauge
```

---

## 3. Configuração do Prometheus

**Localização no projeto:** `config/prometheus/prometheus.yml`

### 3.1 Job de coleta do FortiGate

```yaml
- job_name: fortigate
  scrape_interval: 3m # coleta a cada 3 minutos
  scrape_timeout: 3m # aguarda ate 3 minutos por resposta
  metrics_path: /snmp # endpoint do SNMP Exporter
  params:
    module: [fortigate] # modulo definido no snmp.yml
    auth: [public] # autenticacao definida no snmp.yml

  static_configs:
    - targets:
        - IP_FORTIGATE # IP do FortiGate

  relabel_configs:
    # Envia o IP do FortiGate como parametro "target" para o SNMP Exporter
    - source_labels: [__address__]
      target_label: __param_target

    # Usa o IP como label "instance"
    - source_labels: [__param_target]
      target_label: instance

    # Redireciona a coleta para o SNMP Exporter (nao vai direto ao firewall)
    - target_label: __address__
      replacement: snmp-exporter:9116
```

> **Por que `scrape_interval: 3m`?** O FortiGate é um equipamento crítico de rede. Um intervalo maior reduz a carga de processamento no firewall gerada pelas consultas SNMP, sem comprometer a visibilidade das métricas no Grafana.

---

## 4. Validação

### 4.1 Testar SNMP diretamente no FortiGate

Execute no servidor de monitoramento para confirmar que o firewall responde via SNMP:

```bash
snmpwalk -v2c -c public IP_FORTIGATE
```

Se retornar uma lista de OIDs e valores, o SNMP está funcionando corretamente.

Se o comando `snmpwalk` não estiver instalado:

```bash
# Debian/Ubuntu
sudo apt install snmp -y
```

### 4.2 Verificar se o SNMP Exporter está respondendo

```bash
curl "http://localhost:9116/snmp?module=fortigate&auth=public&target=IP_FORTIGATE"
```

O retorno deve ser uma lista de métricas no formato Prometheus (linhas com nomes como `fgSysCpuUsage`, `ifOperStatus`, etc.).

### 4.3 Verificar no Prometheus (interface web)

Acesse o Prometheus pelo navegador: `http://IP_DO_SERVIDOR:9090`

Na barra de busca, execute:

```
up{job="fortigate"}
```

| Valor retornado | Significado                                                  |
| --------------- | ------------------------------------------------------------ |
| `1`             | Coleta bem-sucedida FortiGate respondendo                    |
| `0`             | Falha na coleta verificar conectividade ou configuração SNMP |

Exemplos de queries para validar métricas individuais:

```
fgSysCpuUsage
fgSysMemUsage
fgSysSesCount
```

---

## 5. Boas Práticas de Segurança

| Prática                                           | Motivo                                                        |
| ------------------------------------------------- | ------------------------------------------------------------- |
| Restringir **Allowed Hosts** no FortiGate         | Somente o servidor de monitoramento deve consultar via SNMP   |
| Usar community exclusiva em vez de `public`       | Evita acesso não autorizado caso a porta UDP 161 seja exposta |
| Liberar UDP 161 **somente** para o IP do servidor | Reduz superfície de ataque no firewall                        |
| Monitorar acessos SNMP nos logs do FortiGate      | Detecta tentativas de leitura não autorizadas                 |

### Usar community personalizada (recomendado)

Para maior segurança, troque a community `public` por um valor exclusivo:

1. **No FortiGate:** altere o **Community Name** para o novo valor (ex: `snmp_fg`)
2. **No `snmp.yml`:** atualize a autenticação:

```yaml
auths:
  public:
    version: 2
    community: snmp_fg # novo valor
```

3. **No `prometheus.yml`:** se criou uma auth com nome diferente, atualize o campo `auth`:

```yaml
params:
  module: [fortigate]
  auth: [nome_da_auth]
```

4. Reinicie o SNMP Exporter: `docker compose restart snmp-exporter`

---

## 6. Troubleshooting

### SNMP não responde

- Confirme que o SNMP está habilitado: **System SNMP** no painel do FortiGate
- Verifique se o IP do servidor está listado em **Allowed Hosts** na community
- Confirme que a porta UDP 161 está liberada entre o servidor e o FortiGate (pode estar bloqueada por policy do próprio firewall)

### Verificar se a policy do FortiGate não bloqueia o SNMP

O próprio FortiGate pode ter uma policy bloqueando tráfego para ele mesmo. Verifique em:

```
Policy & Objects  Local In Policy
```

Deve existir uma regra permitindo UDP 161 do IP do servidor de monitoramento para a interface de gerenciamento.

### Prometheus mostra `up = 0`

- Verifique se o container `snmp-exporter` está rodando: `docker compose ps`
- Teste o acesso direto ao SNMP Exporter (ver Seção 4.2)
- Verifique os logs: `docker compose logs snmp-exporter`
- Confirme que o `scrape_timeout` não está menor que o tempo de resposta do equipamento (atualmente ambos em `3m`)

### Métricas aparecem no Prometheus mas não no Grafana

- Confirme que o datasource do Prometheus está configurado corretamente no Grafana
- Verifique se o dashboard importado é o correto (`dashboardsGrafana/firewall/fortigate.json`)
- O label `instance` no job é o próprio IP (`IP_FORTIGATE`) certifique-se de que os filtros do dashboard correspondem a esse valor
