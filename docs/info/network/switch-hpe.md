# Monitoramento de Switch HPE (Comware)

## Objetivo

Este documento descreve o procedimento completo para:

- Configurar SNMP v2c nos switches HPE (firmware Comware)
- Integrar os switches com o SNMP Exporter
- Coletar métricas via Prometheus
- Visualizar os dados no Grafana

---

## Como Funciona (Visão Geral)

Antes de qualquer configuração, é importante entender o fluxo de dados do monitoramento:

```
Switch HPE  <--(UDP 161 / SNMP)--  SNMP Exporter  <--(HTTP /metrics)--  Prometheus  -->  Grafana
```

1. **Switch HPE** — expõe dados de rede via protocolo SNMP (CPU, memória, interfaces, temperatura, PoE, etc.)
2. **SNMP Exporter** — é um serviço intermediário que recebe requisições do Prometheus, consulta o switch via SNMP e traduz os dados para o formato que o Prometheus entende
3. **Prometheus** — coleta as métricas do SNMP Exporter em intervalos regulares e armazena no banco de dados de séries temporais
4. **Grafana** — lê os dados do Prometheus e exibe nos dashboards visuais

> O SNMP Exporter **não** se comunica diretamente com o switch de forma contínua — ele só consulta quando o Prometheus faz uma requisição.

---

## Pré-requisitos

Antes de começar, verifique se os itens abaixo estão disponíveis:

| Item              | Descrição                                                         |
| ----------------- | ----------------------------------------------------------------- |
| Acesso ao switch  | Via console físico ou SSH                                         |
| Firewall liberado | UDP 161 aberto entre o servidor de monitoramento e o IP do switch |
| Stack em execução | Docker Compose rodando (`docker compose up -d`)                   |

---

## 1. Configuração SNMP no Switch HPE (Comware)

Esta seção descreve como habilitar o SNMP no switch para que ele responda às consultas do SNMP Exporter.

> **O que é SNMP?** Simple Network Management Protocol — protocolo padrão para monitoramento de dispositivos de rede. A versão v2c utiliza uma _community string_ como senha de leitura.

### 1.1 Acessar o switch

Conecte ao switch via **SSH** ou via **cabo console** diretamente na porta console do equipamento.

### 1.2 Verificar o IP do switch

Após logar no switch, verifique o IP configurado nas interfaces:

```
display ip interface brief
```

Anote o IP de gerenciamento — ele será o `target` configurado no Prometheus mais adiante.

### 1.3 Entrar no modo de configuração global

```
system-view
```

O prompt mudará para `[NomeDoSwitch]`, indicando que você está no modo de configuração.

### 1.4 Habilitar o agente SNMP

```
snmp-agent
```

### 1.5 Criar a community de leitura

```
snmp-agent community read COMMUNITY_DO_SEU_SWITCH
```

> **Sobre a community:** `COMMUNITY_DO_SEU_SWITCH` é a "senha" de leitura SNMP. Apenas o servidor com essa string poderá consultar o switch. **Nunca use "public"** — é inseguro e padrão de fábrica.

### 1.6 Definir a versão SNMP

```
snmp-agent sys-info version v2c
```

### 1.7 Salvar a configuração

```
save
```

O switch pedirá confirmações. Responda:

- `Y` → confirmar nome do arquivo
- `ENTER` → aceitar nome padrão
- `Y` → confirmar gravação

### 1.8 Remover uma community (quando necessário)

Caso precise remover ou trocar a community:

```
undo snmp-agent community read nomedacommunity
```

Substitua `nomedacommunity` pelo nome da community que deseja remover.

---

## 2. Configuração do SNMP Exporter (Docker)

O SNMP Exporter roda como container Docker e atua como tradutor entre o Prometheus e os switches via SNMP.

### 2.1 Trecho do Docker Compose

O serviço já está declarado no `docker-compose.yml` do projeto:

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

**O que cada parte faz:**

| Campo                     | Descrição                                                                |
| ------------------------- | ------------------------------------------------------------------------ |
| `image`                   | Imagem oficial do SNMP Exporter no Docker Hub                            |
| `ports: 9116`             | Porta HTTP onde o Prometheus faz as requisições                          |
| `volumes`                 | Monta o arquivo de configuração local dentro do container                |
| `restart: unless-stopped` | Reinicia automaticamente em caso de falha (exceto se parado manualmente) |

### 2.2 Arquivo de configuração: `snmp.yml`

**Localização no projeto:** `config/snmp/snmp.yml`

Este arquivo define:

- **Autenticações** (`auths`) — credenciais SNMP por perfil
- **Módulos** (`modules`) — conjunto de OIDs a coletar por tipo de dispositivo

> **O que são OIDs?** Object Identifiers — são os "endereços" das informações dentro do switch. Por exemplo, `1.3.6.1.2.1.1.3` é o OID do uptime do sistema.

```yaml
auths:
  public:
  hp_auth:
    version: 2
    community: COMMUNITY_DO_SEU_SWITCH

modules:
  hp_switch:
    timeout: 25s
    retries: 2
    max_repetitions: 25
    walk:
      # ===== System =====
      - 1.3.6.1.2.1.1.1 # sysDescr    — descrição do sistema
      - 1.3.6.1.2.1.1.3 # sysUpTime   — tempo ligado
      - 1.3.6.1.2.1.1.5 # sysName     — nome do switch

      # ===== Interfaces =====
      - 1.3.6.1.2.1.2.2.1.1 # ifIndex     — índice da interface
      - 1.3.6.1.2.1.2.2.1.2 # ifDescr     — descrição da interface
      - 1.3.6.1.2.1.2.2.1.5 # ifSpeed     — velocidade
      - 1.3.6.1.2.1.2.2.1.7 # ifAdminStatus — estado administrativo
      - 1.3.6.1.2.1.2.2.1.8 # ifOperStatus  — estado operacional
      - 1.3.6.1.2.1.31.1.1.1.1 # ifName     — nome da interface
      - 1.3.6.1.2.1.31.1.1.1.6 # ifHCInOctets  — bytes recebidos
      - 1.3.6.1.2.1.31.1.1.1.10 # ifHCOutOctets — bytes enviados

      # ===== Erros / Descartes =====
      - 1.3.6.1.2.1.2.2.1.13 # ifInDiscards  — pacotes descartados (entrada)
      - 1.3.6.1.2.1.2.2.1.14 # ifInErrors    — erros de entrada
      - 1.3.6.1.2.1.2.2.1.19 # ifOutDiscards — pacotes descartados (saída)
      - 1.3.6.1.2.1.2.2.1.20 # ifOutErrors   — erros de saída

      # ===== CPU / Memória HPE Comware =====
      - 1.3.6.1.4.1.25506.2.6.1.1.1.1.6 # hh3cEntityExtCpuUsage    — uso de CPU (%)
      - 1.3.6.1.4.1.25506.2.6.1.1.1.1.8 # hh3cEntityExtMemUsage    — uso de memória (%)

      # ===== Temperatura =====
      - 1.3.6.1.4.1.25506.2.6.1.1.1.1.12 # hh3cEntityExtTemperature — temperatura (ºC)

      # ===== Fan Speed =====
      - 1.3.6.1.4.1.25506.2.6.1.1.1.1.20 # hh3cEntityExtFanSpeed    — velocidade do fan (%)

      # ===== PSU Status =====
      - 1.3.6.1.4.1.25506.2.6.1.1.1.1.19 # hh3cEntityExtErrorStatus — status da fonte

      # ===== PoE =====
      - 1.3.6.1.2.1.105.1.3.1.1.2 # pethMainPseOperPower       — potencia total disponivel
      - 1.3.6.1.2.1.105.1.3.1.1.4 # pethMainPseConsumptionPower — potencia consumida
      - 1.3.6.1.2.1.105.1.1.1.6 # pethPsePortDetectionStatus  — status PoE por porta
      - 1.3.6.1.2.1.105.1.1.1.11 # pethPsePortPowerConsumption — consumo por porta (mW)

    metrics:
      # ==========================================
      #  SYSTEM
      # ==========================================
      - name: sysDescr
        oid: 1.3.6.1.2.1.1.1
        type: DisplayString
        help: A textual description of the entity

      - name: sysUpTime
        oid: 1.3.6.1.2.1.1.3
        type: gauge
        help: System uptime in hundredths of a second

      - name: sysName
        oid: 1.3.6.1.2.1.1.5
        type: DisplayString
        help: An administratively-assigned name for this managed node

      # ==========================================
      #  CPU
      # ==========================================
      - name: hh3cEntityExtCpuUsage
        oid: 1.3.6.1.4.1.25506.2.6.1.1.1.1.6
        type: gauge
        help: The CPU usage for this entity (percentage)
        indexes:
          - labelname: hh3cEntityExtPhysicalIndex
            type: gauge

      # ==========================================
      #  MEMORY
      # ==========================================
      - name: hh3cEntityExtMemUsage
        oid: 1.3.6.1.4.1.25506.2.6.1.1.1.1.8
        type: gauge
        help: The memory usage for this entity (percentage)
        indexes:
          - labelname: hh3cEntityExtPhysicalIndex
            type: gauge

      # ==========================================
      #  TEMPERATURE
      # ==========================================
      - name: hh3cEntityExtTemperature
        oid: 1.3.6.1.4.1.25506.2.6.1.1.1.1.12
        type: gauge
        help: The temperature for this entity (Celsius)
        indexes:
          - labelname: hh3cEntityExtPhysicalIndex
            type: gauge

      # ==========================================
      #  FAN SPEED
      # ==========================================
      - name: hh3cEntityExtFanSpeed
        oid: 1.3.6.1.4.1.25506.2.6.1.1.1.1.20
        type: gauge
        help: The fan speed for this entity (percentage)
        indexes:
          - labelname: hh3cEntityExtPhysicalIndex
            type: gauge

      # ==========================================
      #  PSU STATUS
      # ==========================================
      - name: hh3cEntityExtErrorStatus
        oid: 1.3.6.1.4.1.25506.2.6.1.1.1.1.19
        type: gauge
        help: Entity error status (2=normal)
        indexes:
          - labelname: hh3cEntityExtPhysicalIndex
            type: gauge

      # ==========================================
      #  PoE - POWER SOURCE
      # ==========================================
      - name: pethMainPseOperPower
        oid: 1.3.6.1.2.1.105.1.3.1.1.2
        type: gauge
        help: Total PoE power available (Watts)
        indexes:
          - labelname: pethMainPseGroupIndex
            type: gauge

      - name: pethMainPseConsumptionPower
        oid: 1.3.6.1.2.1.105.1.3.1.1.4
        type: gauge
        help: Total PoE power consumed (Watts)
        indexes:
          - labelname: pethMainPseGroupIndex
            type: gauge

      # ==========================================
      #  PoE - PER PORT
      # ==========================================
      - name: pethPsePortDetectionStatus
        oid: 1.3.6.1.2.1.105.1.1.1.6
        type: gauge
        help: "PoE port detection status (1=disabled, 2=searching, 3=deliveringPower, 4=fault)"
        indexes:
          - labelname: pethPsePortGroupIndex
            type: gauge
          - labelname: pethPsePortIndex
            type: gauge

      - name: pethPsePortPowerConsumption
        oid: 1.3.6.1.2.1.105.1.1.1.11
        type: gauge
        help: Power consumption on this port (milliwatts)
        indexes:
          - labelname: pethPsePortGroupIndex
            type: gauge
          - labelname: pethPsePortIndex
            type: gauge

      # ==========================================
      #  INTERFACES
      # ==========================================
      - name: ifName
        oid: 1.3.6.1.2.1.31.1.1.1.1
        type: DisplayString
        help: The textual name of the interface
        indexes:
          - labelname: ifIndex
            type: gauge

      - name: ifDescr
        oid: 1.3.6.1.2.1.2.2.1.2
        type: DisplayString
        help: A textual string containing information about the interface
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
        lookups:
          - labels: [ifIndex]
            labelname: ifDescr
            oid: 1.3.6.1.2.1.2.2.1.2
            type: DisplayString

      - name: ifAdminStatus
        oid: 1.3.6.1.2.1.2.2.1.7
        type: gauge
        help: "The desired state of the interface (1=up, 2=down, 3=testing)"
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifDescr
            oid: 1.3.6.1.2.1.2.2.1.2
            type: DisplayString

      - name: ifOperStatus
        oid: 1.3.6.1.2.1.2.2.1.8
        type: gauge
        help: "The current operational state of the interface (1=up, 2=down)"
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifDescr
            oid: 1.3.6.1.2.1.2.2.1.2
            type: DisplayString

      - name: ifHCInOctets
        oid: 1.3.6.1.2.1.31.1.1.1.6
        type: counter
        help: The total number of octets received on the interface
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifDescr
            oid: 1.3.6.1.2.1.2.2.1.2
            type: DisplayString

      - name: ifHCOutOctets
        oid: 1.3.6.1.2.1.31.1.1.1.10
        type: counter
        help: The total number of octets transmitted out of the interface
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifDescr
            oid: 1.3.6.1.2.1.2.2.1.2
            type: DisplayString

      # ==========================================
      #  ERROS / DESCARTES
      # ==========================================
      - name: ifInDiscards
        oid: 1.3.6.1.2.1.2.2.1.13
        type: counter
        help: The number of inbound packets discarded
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifDescr
            oid: 1.3.6.1.2.1.2.2.1.2
            type: DisplayString

      - name: ifInErrors
        oid: 1.3.6.1.2.1.2.2.1.14
        type: counter
        help: The number of inbound packets with errors
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifDescr
            oid: 1.3.6.1.2.1.2.2.1.2
            type: DisplayString

      - name: ifOutDiscards
        oid: 1.3.6.1.2.1.2.2.1.19
        type: counter
        help: The number of outbound packets discarded
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifDescr
            oid: 1.3.6.1.2.1.2.2.1.2
            type: DisplayString

      - name: ifOutErrors
        oid: 1.3.6.1.2.1.2.2.1.20
        type: counter
        help: The number of outbound packets with errors
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels: [ifIndex]
            labelname: ifDescr
            oid: 1.3.6.1.2.1.2.2.1.2
            type: DisplayString
```

---

## 3. Configuração do Prometheus

O Prometheus é responsável por disparar as coletas periodicamente. Ele chama o SNMP Exporter passando o IP do switch como parâmetro.

**Localização no projeto:** `config/prometheus/prometheus.yml`

### 3.1 Job de coleta dos switches HPE

```yaml
- job_name: snmp-hp-switch
  scrape_interval: 60s # coleta a cada 60 segundos
  scrape_timeout: 30s # aguarda ate 30s por resposta
  metrics_path: /snmp # endpoint do SNMP Exporter
  params:
    module: [hp_switch] # modulo definido no snmp.yml
    auth: [hp_auth] # autenticacao definida no snmp.yml

  static_configs:
    - targets:
        - IP_SWITCH_01 # IP do Switch 01
      labels:
        instance: "Switch 01"

    - targets:
        - IP_SWITCH_02 # IP do Switch 02
      labels:
        instance: "Switch 02"

  relabel_configs:
    # Envia o IP do switch como parametro "target" para o SNMP Exporter
    - source_labels: [__address__]
      target_label: __param_target

    # Mantem o label "instance" com o nome amigavel do switch
    - source_labels: [instance]
      target_label: instance

    # Redireciona a coleta para o SNMP Exporter (nao vai direto ao switch)
    - target_label: __address__
      replacement: snmp-exporter:9116
```

### 3.2 Entendendo o `relabel_configs`

O `relabel_configs` é necessário porque o Prometheus não fala SNMP — ele sempre usa HTTP. O fluxo funciona assim:

1. Prometheus tenta "raspar" o endereço `IP_SWITCH_01` (o switch)
2. O `relabel_configs` intercepta e **redireciona** a requisição para `snmp-exporter:9116`
3. Mas passa o IP original (`IP_SWITCH_01`) como parâmetro `?target=IP_SWITCH_01`
4. O SNMP Exporter recebe, consulta o switch via SNMP e retorna as métricas

Para adicionar um novo switch, basta incluir um novo bloco em `static_configs` com o IP e nome do equipamento.

---

## 4. Validação

Após configurar tudo, execute as validações abaixo para garantir que o monitoramento está funcionando.

### 4.1 Testar SNMP diretamente no switch

Execute na VM de monitoramento para confirmar que o switch responde via SNMP:

```bash
snmpwalk -v2c -c COMMUNITY_DO_SEU_SWITCH IP_DO_SWITCH
```

Substitua `IP_DO_SWITCH` pelo IP real (ex: `IP_SWITCH_01`). Se retornar uma lista de OIDs e valores, o SNMP está funcionando corretamente.

Se o comando `snmpwalk` não estiver instalado:

```bash
# Debian/Ubuntu
sudo apt install snmp -y
```

### 4.2 Verificar se o SNMP Exporter está respondendo

Acesse pelo navegador ou via curl na VM:

```bash
curl "http://localhost:9116/snmp?module=hp_switch&auth=hp_auth&target=IP_SWITCH_01"
```

O retorno deve ser uma lista de métricas no formato Prometheus (linhas começando com `#` e nomes de métricas).

### 4.3 Verificar no Prometheus (interface web)

Acesse o Prometheus pelo navegador: `http://IP_DA_VM:9090`

Na barra de busca, execute a query:

```
up{job="snmp-hp-switch"}
```

| Valor retornado | Significado                                                    |
| --------------- | -------------------------------------------------------------- |
| `1`             | Coleta bem-sucedida — switch respondendo                       |
| `0`             | Falha na coleta — verificar conectividade ou configuração SNMP |

---

## 5. Adicionando um Novo Switch ao Monitoramento

Para monitorar um switch HPE adicional, siga os passos:

1. **No switch novo:** Repita os passos da Seção 1 (habilitar SNMP, criar community, salvar)
2. **No `prometheus.yml`:** Adicione um novo bloco em `static_configs`:

```yaml
- targets:
    - 192.168.12.XXX # IP do novo switch
  labels:
    instance: "Switch HPX" # Nome identificavel
```

3. **Reinicie o Prometheus** para aplicar a nova configuração:

```bash
docker compose restart prometheus
```

4. **Valide** no Prometheus que o novo switch aparece com valor `1` na query `up{job="snmp-hp-switch"}`.

---

## 6. Boas Práticas de Segurança

| Prática                                     | Motivo                                                       |
| ------------------------------------------- | ------------------------------------------------------------ |
| Usar community exclusiva (ex: `COMMUNITY_DO_SEU_SWITCH`)    | Evita uso indevido por outros sistemas                       |
| **Nunca usar `public`**                     | Community padrão de fábrica, conhecida por qualquer atacante |
| Restringir SNMP por ACL no switch           | Permite que apenas o IP da VM consulte via SNMP              |
| Liberar UDP 161 **somente** para o IP da VM | Reduz superfície de ataque no firewall                       |
| Documentar todas as communities ativas      | Facilita auditoria e desativação quando necessário           |

### Restringir SNMP por ACL no switch (recomendado)

Para que apenas a VM de monitoramento consiga consultar o switch:

```
acl number 2000
 rule 0 permit source IP_DA_VM 0
 rule 5 deny

snmp-agent community read COMMUNITY_DO_SEU_SWITCH acl 2000
```

Substitua `IP_DA_VM` pelo IP real da VM onde o Prometheus/SNMP Exporter estão rodando.

---

## 7. Troubleshooting

### SNMP não responde

- Verifique se o serviço está ativo no switch: `display snmp-agent`
- Confirme que a community está correta: `display snmp-agent community`
- Verifique se a porta UDP 161 está liberada no firewall/VLAN entre a VM e o switch

### Prometheus mostra `up = 0`

- Verifique se o container `snmp-exporter` está rodando: `docker compose ps`
- Teste o acesso direto ao SNMP Exporter (ver Seção 5.2)
- Verifique os logs do SNMP Exporter: `docker compose logs snmp-exporter`

### Métricas aparecem no Prometheus mas não no Grafana

- Confirme que o datasource do Prometheus está configurado corretamente no Grafana
- Verifique se o dashboard importado é o correto (`dashboardsGrafana/network/switch-hpe.json`)
- Certifique-se de que o filtro de `instance` no dashboard corresponde ao label configurado no `prometheus.yml`
