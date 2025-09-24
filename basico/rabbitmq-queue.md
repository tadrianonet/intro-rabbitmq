# RabbitMQ: Queue (Fila) - Conceitos Fundamentais

## O que é uma Queue?

A **Queue** (Fila) é a estrutura de dados fundamental do RabbitMQ que **armazena mensagens** até que sejam consumidas. Funciona como um buffer FIFO (First In, First Out) entre producers e consumers.

## 🎯 **Analogia do Mundo Real**

Imagine uma **fila de banco**:
- **Pessoas chegando** = Mensagens sendo adicionadas
- **Fila de espera** = Queue
- **Caixa do banco** = Consumer
- **Ordem de chegada** = FIFO (primeiro a chegar, primeiro a ser atendido)

**Imagem sugerida:** Uma fila de pessoas em um banco, com números indicando a ordem de chegada e setas mostrando o fluxo de entrada e saída.

## 📊 **Diagrama Visual da Queue**

```
Producer 1 ──┐
              ├──→ [Exchange] ──→ [Queue: msg1|msg2|msg3|msg4] ──→ Consumer
Producer 2 ──┘                           ↑                           ↓
                                    HEAD (próxima                  TAIL (última
                                   mensagem a ser                mensagem
                                    consumida)                   adicionada)
```

**Imagem sugerida:** Diagrama mostrando uma fila com mensagens organizadas como vagões de trem, indicando claramente HEAD e TAIL, com cores diferentes para entrada e saída.

## 🔧 **Tipos de Queues**

### 1. **Classic Queues (Padrão)**
**Características:**
- **Uma única thread** para processar mensagens
- **Melhor para throughput moderado**
- **Compatível** com todas as versões
- **Uso comum** em aplicações tradicionais

### 2. **Quorum Queues (Recomendado para Produção)**
**Características:**
- **Alta disponibilidade** através de replicação
- **Melhor consistência** de dados
- **Ideal para dados críticos**
- **Tolerante a falhas** de nós individuais

### 3. **Stream Queues (Para High Throughput)**
**Características:**
- **Retenção** baseada em tempo
- **Múltiplos consumers** podem ler as mesmas mensagens
- **Ideal para logs** e eventos
- **Comportamento similar** ao Apache Kafka

**Imagem sugerida:** Três diagramas lado a lado mostrando a diferença visual entre os tipos de queue: Classic (fila simples), Quorum (fila replicada em múltiplos nós), Stream (fila persistente com múltiplos readers).

## ⚙️ **Propriedades Importantes das Queues**

### 1. **Durabilidade**
- **Queues Duráveis**: Sobrevivem a restart do broker
- **Queues Temporárias**: Perdidas quando broker reinicia
- **Escolha**: Baseada na criticidade dos dados

### 2. **Exclusividade**
- **Queue Exclusiva**: Apenas uma conexão pode usar
- **Queue Compartilhada**: Múltiplos consumers podem acessar
- **Uso**: Sessões temporárias vs processamento distribuído

### 3. **Auto-delete**
- **Definição**: Queue é deletada automaticamente quando não há consumers
- **Uso**: Ideal para queues temporárias
- **Cuidado**: Pode causar perda de mensagens se mal configurada

**Imagem sugerida:** Infográfico mostrando as três propriedades como interruptores ON/OFF com suas consequências visuais.

## 📏 **Configurações de Controle**

### 1. **Limite de Mensagens**
- **x-max-length**: Número máximo de mensagens na queue
- **Estratégias de Overflow**:
  - **reject-publish**: Rejeita novas mensagens
  - **drop-head**: Remove mensagens mais antigas
  - **reject-publish-dlx**: Envia para Dead Letter Exchange

### 2. **Time-To-Live (TTL)**
- **TTL de Mensagem**: Tempo máximo que uma mensagem fica na queue
- **TTL de Queue**: Tempo máximo que a queue fica vazia antes de ser deletada
- **Uso**: Controle de recursos e limpeza automática

### 3. **Dead Letter Exchange (DLX)**
- **Conceito**: Para onde vão mensagens "mortas" (rejeitadas, expiradas, etc.)
- **Routing Key**: Como rotear mensagens para DLX
- **Retry Limit**: Quantas tentativas antes de enviar para DLX

**Imagem sugerida:** Diagrama de fluxo mostrando uma mensagem passando por diferentes estágios: Queue Principal → TTL Expiry → DLX → Queue de Falhas.

## 📊 **Estados e Ciclo de Vida da Queue**

### Estados Possíveis:
1. **Empty**: Sem mensagens, aguardando
2. **Ready**: Com mensagens aguardando consumers
3. **Unacked**: Mensagens entregues mas não confirmadas
4. **Running**: Processando mensagens ativamente
5. **Idle**: Temporariamente inativa

### Transições de Estado:
```
Empty → Ready (mensagens chegam)
Ready → Running (consumer conecta)
Running → Unacked (mensagem enviada)
Unacked → Empty (ACK recebido)
Unacked → Ready (NACK recebido)
```

**Imagem sugerida:** Diagrama de estados como um grafo direcionado, com círculos representando estados e setas mostrando transições possíveis.

## 📈 **Métricas e Monitoramento**

### Métricas Essenciais:
- **Message Count**: Número de mensagens na queue
- **Consumer Count**: Quantos consumers estão conectados
- **Message Rate**: Taxa de entrada e saída de mensagens
- **Unacked Messages**: Mensagens enviadas mas não confirmadas

### Alertas Importantes:
- **Queue muito cheia**: Pode indicar consumer lento
- **Sem consumers**: Mensagens acumulando sem processamento
- **Taxa de erro alta**: Problemas na aplicação consumidora
- **Memory pressure**: Queue usando muita memória

**Imagem sugerida:** Dashboard estilo semáforo com indicadores verdes, amarelos e vermelhos para diferentes métricas, facilitando identificação rápida de problemas.

## 🚨 **Problemas Comuns e Soluções**

### 1. **Queue Bloat (Acúmulo de Mensagens)**
- **Sintomas**: Mensagens acumulando rapidamente
- **Causas**: Consumer muito lento, falha no processamento
- **Soluções**: Aumentar consumers, otimizar processamento, configurar limites

### 2. **Memory Pressure**
- **Sintomas**: Broker usando muita RAM
- **Causas**: Queues muito grandes, mensagens não persistidas
- **Soluções**: Configurar paging para disco, limitar tamanho das queues

### 3. **Consumer Lento**
- **Sintomas**: Baixa taxa de processamento
- **Causas**: Lógica complexa, chamadas externas lentas
- **Soluções**: Prefetch adequado, paralelização, cache

**Imagem sugerida:** Três cenários problemáticos mostrados como "antes e depois", ilustrando o problema e sua solução de forma visual.

## 🎯 **Padrões de Uso Avançados**

### 1. **Priority Queues**
- **Conceito**: Mensagens com prioridade são processadas primeiro
- **Configuração**: Define nível máximo de prioridade (0-255)
- **Uso**: Pedidos VIP, alertas críticos, processamento urgente

### 2. **Delayed Queues**
- **Conceito**: Mensagens ficam "dormindo" por um tempo definido
- **Implementação**: Via plugin x-delayed-message
- **Uso**: Lembrete de pagamento, retry com backoff, agendamento

### 3. **Work Queues (Distribuição de Carga)**
- **Conceito**: Múltiplos workers processam da mesma queue
- **Distribuição**: Round-robin entre consumers disponíveis
- **Uso**: Processamento de imagens, envio de emails, cálculos pesados

**Imagem sugerida:** Três diagramas separados mostrando cada padrão: Priority (mensagens com números de prioridade), Delayed (relógio indicando tempo), Work (múltiplos workers conectados à mesma queue).

## 🏗️ **Decisões de Design**

### Quando usar cada tipo:

**Classic Queue**:
- ✅ Aplicações simples
- ✅ Throughput moderado
- ✅ Compatibilidade com versões antigas

**Quorum Queue**:
- ✅ Dados críticos
- ✅ Alta disponibilidade necessária
- ✅ Consistência importante

**Stream Queue**:
- ✅ High throughput
- ✅ Retenção por tempo
- ✅ Múltiplos consumers para mesmos dados

**Imagem sugerida:** Árvore de decisão visual ajudando a escolher o tipo correto de queue baseado nos requisitos do projeto.

## 📚 **Resumo para Alunos**

### Pontos-Chave das Queues:

✅ **São o coração** do sistema de mensageria  
✅ **Armazenam mensagens** de forma ordenada (FIFO)  
✅ **Diferentes tipos** para diferentes necessidades  
✅ **Configurações flexíveis** para controle de recursos  
✅ **Monitoramento essencial** para manter performance  
✅ **Padrões avançados** para casos específicos  

### Conceitos Fundamentais:
- **FIFO**: Primeira mensagem a entrar é a primeira a sair
- **Durabilidade**: Configurar baseado na criticidade dos dados
- **TTL**: Controlar tempo de vida para evitar acúmulo
- **DLX**: Sempre ter estratégia para mensagens problemáticas
- **Monitoring**: Observar métricas continuamente

### Dicas Práticas:
- **Escolha o tipo certo** para sua necessidade
- **Configure limites** para evitar problemas de recursos
- **Monitore sempre** para detectar problemas cedo
- **Teste cenários de falha** antes de colocar em produção

**Imagem sugerida:** Mapa mental visual conectando todos os conceitos principais da Queue, facilitando a visão geral do componente.

---

**Próximo Conceito:** Consumer (Consumidor) - Como processar mensagens das queues