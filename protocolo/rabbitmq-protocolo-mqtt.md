# RabbitMQ: Protocolo MQTT - Conceitos Fundamentais

## O que é MQTT?

O **MQTT** (Message Queuing Telemetry Transport) é um protocolo de mensageria **leve e eficiente**, projetado especificamente para dispositivos IoT (Internet of Things) e redes com **largura de banda limitada**.

## 🎯 **Analogia do Mundo Real**

Imagine um **sistema de telemetria de carros**:
- **Sensores do carro** = Dispositivos IoT
- **Central de monitoramento** = Broker MQTT (RabbitMQ)
- **Dados do GPS/velocidade** = Mensagens leves
- **Sinal de celular fraco** = Rede com pouca largura de banda
- **Bateria do carro** = Recursos limitados

O MQTT é como um sistema que consegue enviar informações essenciais mesmo quando o sinal está fraco e a bateria está baixa.

**Imagem sugerida:** Carros conectados enviando dados para uma central via torres de celular, com indicadores de sinal fraco e economia de bateria.

## 📊 **Características Principais**

### **1. Protocolo Leve**
- **Overhead mínimo**: Headers pequenos (2 bytes mínimo)
- **Payloads compactos**: Ideal para dados simples
- **Eficiência de rede**: Reduz uso de largura de banda
- **Economia de energia**: Prolonga vida útil de dispositivos

### **2. Qualidade de Serviço (QoS)**
- **QoS 0**: At most once (no máximo uma vez)
- **QoS 1**: At least once (pelo menos uma vez)
- **QoS 2**: Exactly once (exatamente uma vez)

### **3. Conectividade**
- **Porta sem TLS**: 1883
- **Porta com TLS**: 8883
- **WebSocket**: Porta 15675 (no RabbitMQ)

**Imagem sugerida:** Diagrama mostrando diferentes dispositivos IoT (sensores, câmeras, termostatos) conectados ao RabbitMQ via MQTT.

## 🏗️ **Arquitetura MQTT**

### **Modelo Publish/Subscribe:**

#### **Publisher (Publicador)**
- **Função**: Envia dados dos sensores
- **Características**: Dispositivos IoT, aplicações móveis
- **Exemplo**: Sensor de temperatura enviando dados

#### **Subscriber (Assinante)**
- **Função**: Recebe dados específicos
- **Características**: Aplicações de monitoramento, dashboards
- **Exemplo**: App que monitora temperatura de casa

#### **Topic (Tópico)**
- **Função**: Canal de comunicação temático
- **Hierarquia**: Estrutura em árvore com "/" como separador
- **Exemplo**: "casa/sala/temperatura", "carro/motor/rpm"

#### **Broker (Corretor)**
- **Função**: RabbitMQ atuando como intermediário
- **Responsabilidades**: Gerenciar subscriptions, entregar mensagens

**Imagem sugerida:** Diagrama de rede mostrando múltiplos dispositivos publicando em topics e aplicações fazendo subscribe aos topics de interesse.

## 🔧 **Sistema de Topics**

### **Estrutura Hierárquica:**
```
smartcity/
├── traffic/
│   ├── zone1/speed
│   ├── zone1/count
│   └── zone2/speed
├── environment/
│   ├── air/quality
│   ├── noise/level
│   └── temperature
└── energy/
    ├── solar/production
    └── consumption/residential
```

### **Wildcards para Subscription:**

#### **Single Level (+)**
- **"smartcity/traffic/+/speed"** captura:
  - ✅ smartcity/traffic/zone1/speed
  - ✅ smartcity/traffic/zone2/speed
  - ❌ smartcity/traffic/zone1/count (não é speed)

#### **Multi Level (#)**
- **"smartcity/environment/#"** captura:
  - ✅ smartcity/environment/air/quality
  - ✅ smartcity/environment/noise/level
  - ✅ smartcity/environment/temperature

**Imagem sugerida:** Árvore hierárquica de topics com diferentes cores para mostrar quais mensagens cada wildcard capturaria.

## ⚡ **Qualidade de Serviço (QoS)**

### **QoS 0 - At Most Once**
- **Garantia**: Pode não entregar ou entregar uma vez
- **Performance**: Mais rápido, menor overhead
- **Uso**: Dados não-críticos, telemetria frequente
- **Exemplo**: Leitura de temperatura a cada segundo

### **QoS 1 - At Least Once**
- **Garantia**: Entrega pelo menos uma vez (pode duplicar)
- **Performance**: Balanceado
- **Uso**: Dados importantes que podem ser duplicados
- **Exemplo**: Alertas de segurança

### **QoS 2 - Exactly Once**
- **Garantia**: Entrega exatamente uma vez
- **Performance**: Mais lento, maior overhead
- **Uso**: Dados críticos que não podem duplicar
- **Exemplo**: Comandos de ligar/desligar equipamentos

**Imagem sugerida:** Gráfico de barras comparando velocidade vs confiabilidade para cada nível de QoS.

## 🔐 **Recursos de Conectividade**

### **Keep Alive**
- **Propósito**: Manter conexão ativa em redes instáveis
- **Funcionamento**: Ping periódico entre cliente e broker
- **Configuração**: Intervalo ajustável baseado na rede
- **Benefício**: Detecta desconexões rapidamente

### **Last Will and Testament (LWT)**
- **Propósito**: Notificar quando dispositivo desconecta inesperadamente
- **Configuração**: Mensagem automática enviada em caso de falha
- **Uso**: Alertas de dispositivos offline
- **Exemplo**: "device/sensor01/status" → "offline"

### **Retained Messages**
- **Propósito**: Manter última mensagem para novos subscribers
- **Funcionamento**: Broker armazena última mensagem por topic
- **Uso**: Status atual de dispositivos
- **Exemplo**: Estado atual de lâmpadas inteligentes

### **Clean Session**
- **True**: Não manter state entre conexões
- **False**: Preservar subscriptions e mensagens pendentes
- **Uso**: Baseado na natureza do dispositivo (móvel vs fixo)

**Imagem sugerida:** Timeline mostrando como diferentes recursos funcionam durante conexão, desconexão e reconexão de um dispositivo.

## 🌐 **Casos de Uso Práticos**

### **1. Smart Home (Casa Inteligente)**
**Topics Examples:**
- casa/sala/luz/estado
- casa/cozinha/temperatura
- casa/seguranca/sensor/movimento
- casa/jardim/irrigacao/comando

**Scenarios:**
- **Automação**: Ligar luzes baseado em movimento
- **Monitoramento**: Temperatura e umidade
- **Segurança**: Sensores de porta/janela
- **Economia**: Controle de energia

### **2. Industrial IoT**
**Topics Examples:**
- fabrica/linha1/maquina/temperatura
- fabrica/linha1/producao/contador
- fabrica/manutencao/alerta
- fabrica/qualidade/medida

**Scenarios:**
- **Monitoramento**: Temperatura e vibração de máquinas
- **Produção**: Contadores de peças produzidas
- **Manutenção**: Alertas preventivos
- **Qualidade**: Medições em tempo real

### **3. Agricultura Smart**
**Topics Examples:**
- campo/setor1/solo/umidade
- campo/setor1/clima/temperatura
- campo/irrigacao/sistema/estado
- campo/pragas/detector/alerta

**Scenarios:**
- **Irrigação**: Controle baseado em umidade do solo
- **Clima**: Monitoramento meteorológico
- **Pragas**: Detecção automática
- **Produtividade**: Análise de dados históricos

### **4. Smart City (Cidade Inteligente)**
**Topics Examples:**
- cidade/transito/semaforo/estado
- cidade/ar/qualidade/indice
- cidade/energia/consumo/residencial
- cidade/residuos/container/nivel

**Scenarios:**
- **Trânsito**: Otimização de semáforos
- **Ambiente**: Monitoramento da qualidade do ar
- **Energia**: Gestão inteligente do consumo
- **Limpeza**: Coleta de lixo otimizada

**Imagem sugerida:** Mapa de cidade mostrando diferentes sistemas conectados via MQTT (semáforos, sensores de ar, medidores de energia, etc.).

## 🔄 **Integração MQTT + AMQP**

### **Bridge Pattern:**
- **MQTT** para comunicação com dispositivos IoT
- **AMQP** para processamento interno da aplicação
- **RabbitMQ** faz a ponte entre os dois protocolos

### **Fluxo Híbrido:**
```
Dispositivos IoT ─MQTT─→ RabbitMQ ─AMQP─→ Aplicações Enterprise
                           ↓
                    Exchange/Queue
                    Transformação
```

### **Vantagens da Integração:**
- **Melhor de ambos**: Leveza do MQTT + robustez do AMQP
- **Escalabilidade**: Suporte a milhões de dispositivos IoT
- **Flexibilidade**: Diferentes protocolos para diferentes necessidades
- **Manutenção**: Um único broker para todos os protocolos

**Imagem sugerida:** Diagrama mostrando dispositivos IoT usando MQTT conectados ao RabbitMQ, que então distribui as mensagens via AMQP para diferentes sistemas empresariais.

## 📊 **Monitoramento e Métricas**

### **Métricas Importantes:**

#### **Conectividade**
- **Dispositivos conectados**: Quantos dispositivos estão online
- **Taxa de desconexão**: Frequência de perdas de conexão
- **Keep alive timeouts**: Dispositivos que não respondem

#### **Mensagens**
- **Taxa de publicação**: Mensagens por segundo por topic
- **Distribuição de QoS**: Uso de cada nível de qualidade
- **Latência de entrega**: Tempo até entrega ao subscriber

#### **Topics**
- **Topics ativos**: Quantos topics têm tráfego
- **Subscribers por topic**: Quantos listeners por canal
- **Mensagens retained**: Quantas mensagens estão mantidas

### **Alertas Críticos:**
- **Dispositivos offline**: Sensores críticos desconectados
- **Qualidade degradada**: Muitas falhas de QoS 1/2
- **Overhead alto**: Muito uso de QoS 2
- **Topics órfãos**: Topics sem subscribers

**Imagem sugerida:** Dashboard IoT mostrando mapa de dispositivos conectados, gráficos de métricas em tempo real e alertas coloridos.

## 🛡️ **Segurança em MQTT**

### **Níveis de Segurança:**

#### **Network Level**
- **TLS/SSL**: Criptografia de transporte (porta 8883)
- **VPN**: Túneis seguros para dispositivos remotos
- **Firewall**: Controle de acesso por IP/porta

#### **Authentication**
- **Username/Password**: Credenciais básicas
- **Certificates**: Autenticação por certificados X.509
- **Token-based**: OAuth, JWT para dispositivos móveis

#### **Authorization**
- **Topic ACL**: Controle de acesso por topic
- **Read/Write permissions**: Permissões granulares
- **Device-specific**: Cada dispositivo apenas seus topics

### **Boas Práticas de Segurança:**
- **Sempre usar TLS** em produção
- **Credenciais únicas** por dispositivo
- **ACLs restritivas** por tipo de dispositivo
- **Monitoramento** de tentativas de acesso
- **Rotação periódica** de credenciais

**Imagem sugerida:** Layers de segurança como círculos concêntricos protegendo os dispositivos IoT no centro.

## 📚 **Resumo para Alunos**

### **Por que MQTT é Ideal para IoT:**

✅ **Protocolo leve** - Mínimo uso de recursos  
✅ **Baixo consumo** - Ideal para dispositivos com bateria  
✅ **Rede instável** - Funciona bem com conexões intermitentes  
✅ **Escalabilidade** - Suporte a milhões de dispositivos  
✅ **Flexibilidade** - QoS ajustável conforme necessidade  

### **Quando Usar MQTT:**
- **Dispositivos IoT** com recursos limitados
- **Redes com largura de banda limitada**
- **Telemetria** e sensoriamento em tempo real
- **Aplicações móveis** que precisam de eficiência
- **Sistemas distribuídos** geograficamente

### **Conceitos-Chave:**
- **Topics hierárquicos**: Organização em árvore
- **Publish/Subscribe**: Desacoplamento total
- **QoS levels**: Balancear performance vs confiabilidade
- **Wildcards**: Flexibilidade na subscription
- **Retained messages**: Estado atual disponível

### **Lembre-se:**
- MQTT é **complementar** ao AMQP, não substituto
- **Segurança** é fundamental em IoT
- **Monitoramento** de dispositivos é essencial
- **Planejamento de topics** facilita manutenção

**Imagem sugerida:** Infográfico resumindo os benefícios do MQTT para IoT, com ícones representando cada vantagem.

---

**Próximo Protocolo:** STOMP - Protocolo simples baseado em texto
