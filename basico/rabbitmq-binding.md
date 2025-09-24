# RabbitMQ: Binding (Ligação) - Guia Detalhado

## O que é um Binding?

O **Binding** (Ligação) é a **regra de conexão** entre um Exchange e uma Queue que define **quando e como** uma mensagem deve ser roteada. É o elemento que torna o roteamento possível no RabbitMQ.

## 🎯 **Analogia do Mundo Real**

Imagine um **sistema de entrega de correspondência**:
- **Central de Triagem** = Exchange
- **Caixa Postal** = Queue
- **Regra de Entrega** = Binding
- **Endereço/CEP** = Routing Key
- **Tipo de Correspondência** = Headers/Arguments

O binding é como uma **instrução específica** para o funcionário da triagem: "Toda carta com CEP 01234-567 vai para a Caixa Postal A", "Cartas urgentes vão para a Caixa Postal Expressa".

## 📊 **Diagrama Visual**

```
                    Binding Rules
Exchange ────────────────────────────────→ Queue
   ↑                    ↓                    ↓
Routing Key      [Pattern/Condition]    Message Stored
"order.created"    "order.*"           [msg1|msg2|msg3]
```

**Sugestão de Imagem:** Diagrama mostrando múltiplos bindings conectando um exchange a diferentes queues, cada um com suas regras específicas.

## 🔧 **Anatomia de um Binding**

Um binding é composto por:

1. **Exchange**: Onde a mensagem chega
2. **Queue**: Para onde a mensagem vai
3. **Routing Key/Pattern**: Regra de correspondência
4. **Arguments**: Parâmetros adicionais (opcional)

```java
// Estrutura básica de um binding
Binding binding = BindingBuilder
    .bind(queue)                    // Destino (Queue)
    .to(exchange)                   // Origem (Exchange)  
    .with("routing.key.pattern")    // Regra (Routing Key/Pattern)
    .and(arguments);                // Argumentos opcionais
```

## 🎯 **Tipos de Binding por Exchange**

### 1. **Direct Exchange Bindings**

Roteamento baseado em **correspondência exata**.

```java
@Configuration
public class DirectBindingConfig {
    
    @Bean
    public DirectExchange orderExchange() {
        return ExchangeBuilder.directExchange("orders.direct").durable(true).build();
    }
    
    @Bean
    public Queue orderCreatedQueue() {
        return QueueBuilder.durable("orders.created").build();
    }
    
    @Bean
    public Queue orderUpdatedQueue() {
        return QueueBuilder.durable("orders.updated").build();
    }
    
    @Bean
    public Queue orderCancelledQueue() {
        return QueueBuilder.durable("orders.cancelled").build();
    }
    
    // Binding específico para criação de pedidos
    @Bean
    public Binding orderCreatedBinding() {
        return BindingBuilder.bind(orderCreatedQueue())
                            .to(orderExchange())
                            .with("order.created");  // Routing key exata
    }
    
    // Binding específico para atualização de pedidos
    @Bean
    public Binding orderUpdatedBinding() {
        return BindingBuilder.bind(orderUpdatedQueue())
                            .to(orderExchange())
                            .with("order.updated");  // Routing key exata
    }
    
    // Binding específico para cancelamento de pedidos
    @Bean
    public Binding orderCancelledBinding() {
        return BindingBuilder.bind(orderCancelledQueue())
                            .to(orderExchange())
                            .with("order.cancelled"); // Routing key exata
    }
}

// Uso
@Service
public class OrderEventPublisher {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void publishOrderCreated(Order order) {
        // Vai para orderCreatedQueue via binding "order.created"
        rabbitTemplate.convertAndSend("orders.direct", "order.created", order);
    }
    
    public void publishOrderUpdated(Order order) {
        // Vai para orderUpdatedQueue via binding "order.updated"  
        rabbitTemplate.convertAndSend("orders.direct", "order.updated", order);
    }
}
```

### 2. **Topic Exchange Bindings**

Roteamento baseado em **padrões com wildcards**.

```java
@Configuration
public class TopicBindingConfig {
    
    @Bean
    public TopicExchange notificationExchange() {
        return ExchangeBuilder.topicExchange("notifications.topic").durable(true).build();
    }
    
    // Queues por tipo de notificação
    @Bean public Queue emailQueue() { return QueueBuilder.durable("notifications.email").build(); }
    @Bean public Queue smsQueue() { return QueueBuilder.durable("notifications.sms").build(); }
    @Bean public Queue pushQueue() { return QueueBuilder.durable("notifications.push").build(); }
    
    // Queues por prioridade
    @Bean public Queue criticalQueue() { return QueueBuilder.durable("notifications.critical").build(); }
    @Bean public Queue normalQueue() { return QueueBuilder.durable("notifications.normal").build(); }
    
    // Queues por contexto
    @Bean public Queue userQueue() { return QueueBuilder.durable("notifications.user").build(); }
    @Bean public Queue orderQueue() { return QueueBuilder.durable("notifications.order").build(); }
    
    // =============== BINDINGS POR CANAL ===============
    
    @Bean
    public Binding emailNotificationBinding() {
        return BindingBuilder.bind(emailQueue())
                            .to(notificationExchange())
                            .with("notification.*.email");  // notification.{qualquer_coisa}.email
    }
    
    @Bean
    public Binding smsNotificationBinding() {
        return BindingBuilder.bind(smsQueue())
                            .to(notificationExchange())
                            .with("notification.*.sms");    // notification.{qualquer_coisa}.sms
    }
    
    @Bean
    public Binding pushNotificationBinding() {
        return BindingBuilder.bind(pushQueue())
                            .to(notificationExchange())
                            .with("notification.*.push");   // notification.{qualquer_coisa}.push
    }
    
    // =============== BINDINGS POR PRIORIDADE ===============
    
    @Bean
    public Binding criticalNotificationBinding() {
        return BindingBuilder.bind(criticalQueue())
                            .to(notificationExchange())
                            .with("notification.critical.#"); // notification.critical.{zero_ou_mais_palavras}
    }
    
    @Bean
    public Binding normalNotificationBinding() {
        return BindingBuilder.bind(normalQueue())
                            .to(notificationExchange())
                            .with("notification.normal.#");   // notification.normal.{zero_ou_mais_palavras}
    }
    
    // =============== BINDINGS POR CONTEXTO ===============
    
    @Bean
    public Binding userNotificationBinding() {
        return BindingBuilder.bind(userQueue())
                            .to(notificationExchange())
                            .with("notification.user.#");     // notification.user.{zero_ou_mais_palavras}
    }
    
    @Bean
    public Binding orderNotificationBinding() {
        return BindingBuilder.bind(orderQueue())
                            .to(notificationExchange())
                            .with("notification.order.#");    // notification.order.{zero_ou_mais_palavras}
    }
}

// Exemplos de roteamento
@Service  
public class NotificationService {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendCriticalUserEmail(String userId, String message) {
        // Routing key: "notification.critical.user.email"
        // Vai para: criticalQueue, userQueue, emailQueue (múltiplos matches!)
        rabbitTemplate.convertAndSend("notifications.topic", 
                                     "notification.critical.user.email", 
                                     new Notification(userId, message));
    }
    
    public void sendNormalOrderSms(String orderId, String message) {
        // Routing key: "notification.normal.order.sms"  
        // Vai para: normalQueue, orderQueue, smsQueue
        rabbitTemplate.convertAndSend("notifications.topic",
                                     "notification.normal.order.sms",
                                     new Notification(orderId, message));
    }
}
```

### 3. **Headers Exchange Bindings**

Roteamento baseado em **headers da mensagem**.

```java
@Configuration
public class HeadersBindingConfig {
    
    @Bean
    public HeadersExchange routingExchange() {
        return ExchangeBuilder.headersExchange("routing.headers").durable(true).build();
    }
    
    @Bean public Queue priorityQueue() { return QueueBuilder.durable("processing.priority").build(); }
    @Bean public Queue normalQueue() { return QueueBuilder.durable("processing.normal").build(); }
    @Bean public Queue bulkQueue() { return QueueBuilder.durable("processing.bulk").build(); }
    
    // Binding para mensagens prioritárias (TODOS os headers devem bater)
    @Bean
    public Binding priorityBinding() {
        Map<String, Object> headers = new HashMap<>();
        headers.put("priority", "high");
        headers.put("type", "order");
        headers.put("customer-tier", "premium");
        
        return BindingBuilder.bind(priorityQueue())
                            .to(routingExchange())
                            .whereAll(headers).match();  // ALL headers devem bater
    }
    
    // Binding para mensagens normais (QUALQUER header pode bater)
    @Bean
    public Binding normalBinding() {
        Map<String, Object> headers = new HashMap<>();
        headers.put("priority", "normal");
        headers.put("type", "notification");
        
        return BindingBuilder.bind(normalQueue())
                            .to(routingExchange())
                            .whereAny(headers).match();  // ANY header pode bater
    }
    
    // Binding para processamento em lote
    @Bean
    public Binding bulkBinding() {
        Map<String, Object> headers = new HashMap<>();
        headers.put("processing-mode", "batch");
        
        return BindingBuilder.bind(bulkQueue())
                            .to(routingExchange())
                            .whereAll(headers).match();
    }
}

// Uso com headers
@Service
public class HeaderBasedRoutingService {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendPriorityOrder(Order order) {
        MessageProperties props = new MessageProperties();
        props.setHeader("priority", "high");
        props.setHeader("type", "order");
        props.setHeader("customer-tier", "premium");
        props.setHeader("created-at", System.currentTimeMillis());
        
        Message message = new Message(serialize(order), props);
        // Vai para priorityQueue (todos os headers batem)
        rabbitTemplate.send("routing.headers", "", message);
    }
    
    public void sendNotification(String notification) {
        MessageProperties props = new MessageProperties();
        props.setHeader("type", "notification");
        props.setHeader("urgency", "low");
        
        Message message = new Message(notification.getBytes(), props);
        // Vai para normalQueue (header "type" bate)
        rabbitTemplate.send("routing.headers", "", message);
    }
}
```

### 4. **Fanout Exchange Bindings**

Para fanout, o binding é **sempre simples** (sem routing key).

```java
@Configuration
public class FanoutBindingConfig {
    
    @Bean
    public FanoutExchange eventExchange() {
        return ExchangeBuilder.fanoutExchange("system.events").durable(true).build();
    }
    
    @Bean public Queue auditQueue() { return QueueBuilder.durable("audit.events").build(); }
    @Bean public Queue analyticsQueue() { return QueueBuilder.durable("analytics.events").build(); }
    @Bean public Queue cacheQueue() { return QueueBuilder.durable("cache.invalidation").build(); }
    @Bean public Queue notificationQueue() { return QueueBuilder.durable("event.notifications").build(); }
    
    // Bindings simples para fanout (sem routing key)
    @Bean
    public Binding auditEventBinding() {
        return BindingBuilder.bind(auditQueue()).to(eventExchange());
    }
    
    @Bean
    public Binding analyticsEventBinding() {
        return BindingBuilder.bind(analyticsQueue()).to(eventExchange());
    }
    
    @Bean
    public Binding cacheEventBinding() {
        return BindingBuilder.bind(cacheQueue()).to(eventExchange());
    }
    
    @Bean
    public Binding notificationEventBinding() {
        return BindingBuilder.bind(notificationQueue()).to(eventExchange());
    }
}
```

## 🔄 **Bindings Dinâmicos**

### Criação Programática de Bindings
```java
@Service
public class DynamicBindingService {
    
    @Autowired
    private RabbitAdmin rabbitAdmin;
    
    public void createUserSpecificBinding(String userId) {
        // Criar queue específica para o usuário
        Queue userQueue = QueueBuilder.durable("notifications.user." + userId)
                                     .autoDelete()  // Remove quando usuário sai
                                     .build();
        
        rabbitAdmin.declareQueue(userQueue);
        
        // Criar binding específico
        TopicExchange exchange = new TopicExchange("notifications.topic");
        String routingPattern = "notification.user." + userId + ".#";
        
        Binding binding = BindingBuilder.bind(userQueue)
                                       .to(exchange)
                                       .with(routingPattern);
        
        rabbitAdmin.declareBinding(binding);
        
        log.info("Binding criado para usuário {}: {}", userId, routingPattern);
    }
    
    public void removeUserBinding(String userId) {
        Queue userQueue = new Queue("notifications.user." + userId);
        TopicExchange exchange = new TopicExchange("notifications.topic");
        String routingPattern = "notification.user." + userId + ".#";
        
        Binding binding = BindingBuilder.bind(userQueue)
                                       .to(exchange)
                                       .with(routingPattern);
        
        rabbitAdmin.removeBinding(binding);
        rabbitAdmin.deleteQueue("notifications.user." + userId);
        
        log.info("Binding removido para usuário {}", userId);
    }
}
```

### Bindings Baseados em Configuração
```java
@Configuration
public class ConfigurableBindingConfig {
    
    @Value("${app.routing.patterns}")
    private List<String> routingPatterns;
    
    @Value("${app.queues.enabled}")
    private List<String> enabledQueues;
    
    @Bean
    public TopicExchange configurableExchange() {
        return ExchangeBuilder.topicExchange("configurable.routing").durable(true).build();
    }
    
    @Bean
    public List<Binding> configurableBindings() {
        List<Binding> bindings = new ArrayList<>();
        TopicExchange exchange = configurableExchange();
        
        for (String queueName : enabledQueues) {
            Queue queue = QueueBuilder.durable(queueName).build();
            
            for (String pattern : routingPatterns) {
                if (pattern.contains(queueName.replace(".", "_"))) {
                    Binding binding = BindingBuilder.bind(queue)
                                                   .to(exchange)
                                                   .with(pattern);
                    bindings.add(binding);
                }
            }
        }
        
        return bindings;
    }
}
```

## 📊 **Monitoramento de Bindings**

```java
@RestController
@RequestMapping("/admin/bindings")
public class BindingMonitoringController {
    
    @Autowired
    private RabbitAdmin rabbitAdmin;
    
    @GetMapping("/{exchangeName}")
    public List<BindingInfo> getExchangeBindings(@PathVariable String exchangeName) {
        // Obter bindings via Management API
        return getBindingsFromManagementApi(exchangeName);
    }
    
    @PostMapping("/test")
    public BindingTestResult testBinding(@RequestBody BindingTestRequest request) {
        // Testar se uma routing key matcharia um binding
        return testRoutingKeyMatch(request.getExchangeName(), 
                                  request.getRoutingKey(),
                                  request.getHeaders());
    }
    
    @GetMapping("/stats")
    public Map<String, BindingStats> getBindingStats() {
        Map<String, BindingStats> stats = new HashMap<>();
        
        // Obter estatísticas de cada binding
        List<String> exchanges = Arrays.asList(
            "orders.direct", 
            "notifications.topic", 
            "system.events"
        );
        
        for (String exchange : exchanges) {
            List<BindingInfo> bindings = getBindingsFromManagementApi(exchange);
            
            for (BindingInfo binding : bindings) {
                BindingStats bindingStats = BindingStats.builder()
                    .exchangeName(binding.getSource())
                    .queueName(binding.getDestination())
                    .routingKey(binding.getRoutingKey())
                    .messageCount(getQueueMessageCount(binding.getDestination()))
                    .consumerCount(getQueueConsumerCount(binding.getDestination()))
                    .build();
                    
                String key = String.format("%s -> %s", 
                                         binding.getSource(), 
                                         binding.getDestination());
                stats.put(key, bindingStats);
            }
        }
        
        return stats;
    }
    
    private List<BindingInfo> getBindingsFromManagementApi(String exchangeName) {
        // Implementação via HTTP client para Management API
        // GET http://localhost:15672/api/exchanges/%2F/{exchangeName}/bindings/source
        return Collections.emptyList(); // Placeholder
    }
}

@Data
@Builder
public class BindingInfo {
    private String source;          // Exchange name
    private String destination;     // Queue name  
    private String destinationType; // "queue" ou "exchange"
    private String routingKey;      // Routing key/pattern
    private Map<String, Object> arguments;
}

@Data
@Builder
public class BindingStats {
    private String exchangeName;
    private String queueName;
    private String routingKey;
    private int messageCount;
    private int consumerCount;
    private long messagesPublished;
    private double publishRate;
}
```

## 🚨 **Problemas Comuns com Bindings**

### 1. **Bindings Órfãos**
```java
// ❌ Problema: Queue deletada mas binding permanece
@Bean
public Binding orphanBinding() {
    Queue tempQueue = QueueBuilder.nonDurable("temp.queue").build();
    // Se queue for deletada, binding fica órfão
    return BindingBuilder.bind(tempQueue).to(exchange()).with("temp.key");
}

// ✅ Solução: Usar auto-delete ou gerenciar lifecycle
@Bean
public Binding managedBinding() {
    Queue managedQueue = QueueBuilder.durable("managed.queue")
                                    .autoDelete()  // Remove binding automaticamente
                                    .build();
    return BindingBuilder.bind(managedQueue).to(exchange()).with("managed.key");
}
```

### 2. **Routing Keys Conflitantes**
```java
// ❌ Problemático: Padrões que se sobrepõem
@Bean
public Binding broadBinding() {
    return BindingBuilder.bind(broadQueue()).to(topicExchange()).with("order.#");
}

@Bean  
public Binding specificBinding() {
    return BindingBuilder.bind(specificQueue()).to(topicExchange()).with("order.created");
}
// Mensagens "order.created" vão para AMBAS as queues!

// ✅ Melhor: Padrões exclusivos ou usar Direct Exchange
@Bean
public Binding newOrderBinding() {
    return BindingBuilder.bind(newOrderQueue()).to(topicExchange()).with("order.new.#");
}

@Bean
public Binding existingOrderBinding() {
    return BindingBuilder.bind(existingOrderQueue()).to(topicExchange()).with("order.existing.#");
}
```

### 3. **Performance com Muitos Bindings**
```java
// ❌ Muitos bindings no mesmo exchange pode degradar performance
@Configuration
public class TooManyBindingsConfig {
    
    @Bean
    public TopicExchange overloadedExchange() {
        return ExchangeBuilder.topicExchange("overloaded.topic").build();
    }
    
    // 100+ bindings no mesmo exchange...
    // Pode causar lentidão no roteamento
}

// ✅ Solução: Distribuir entre múltiplos exchanges
@Configuration
public class DistributedBindingsConfig {
    
    @Bean
    public TopicExchange ordersExchange() {
        return ExchangeBuilder.topicExchange("orders.topic").build();
    }
    
    @Bean
    public TopicExchange notificationsExchange() {
        return ExchangeBuilder.topicExchange("notifications.topic").build();
    }
    
    @Bean
    public TopicExchange analyticsExchange() {
        return ExchangeBuilder.topicExchange("analytics.topic").build();
    }
}
```

## 🛠️ **Ferramentas para Debugging de Bindings**

### Binding Debugger
```java
@Component
@Slf4j
public class BindingDebugger {
    
    public void debugRouting(String exchangeName, String routingKey, Map<String, Object> headers) {
        log.info("=== DEBUGGING ROUTING ===");
        log.info("Exchange: {}", exchangeName);
        log.info("Routing Key: {}", routingKey);
        log.info("Headers: {}", headers);
        
        List<BindingInfo> bindings = getBindingsForExchange(exchangeName);
        
        for (BindingInfo binding : bindings) {
            boolean matches = testBinding(binding, routingKey, headers);
            log.info("Binding {} -> {} ({}): {}", 
                    binding.getSource(), 
                    binding.getDestination(),
                    binding.getRoutingKey(),
                    matches ? "MATCH" : "NO MATCH");
        }
    }
    
    private boolean testBinding(BindingInfo binding, String routingKey, Map<String, Object> headers) {
        // Implementar lógica de teste baseado no tipo de exchange
        String exchangeType = getExchangeType(binding.getSource());
        
        switch (exchangeType) {
            case "direct":
                return binding.getRoutingKey().equals(routingKey);
                
            case "topic":
                return matchesTopicPattern(binding.getRoutingKey(), routingKey);
                
            case "headers":
                return matchesHeaders(binding.getArguments(), headers);
                
            case "fanout":
                return true; // Fanout sempre match
                
            default:
                return false;
        }
    }
    
    private boolean matchesTopicPattern(String pattern, String routingKey) {
        // Converter padrão topic para regex
        String regex = pattern.replace(".", "\\.")
                             .replace("*", "[^.]+")
                             .replace("#", ".*");
        return routingKey.matches(regex);
    }
}
```

## 📚 **Resumo**

O **Binding** é o elemento que conecta Exchanges às Queues com regras específicas:

✅ **Define roteamento**: Como mensagens chegam às queues certas  
✅ **Flexível**: Suporta padrões simples e complexos  
✅ **Configurável**: Pode ser criado estaticamente ou dinamicamente  
✅ **Monitorável**: Permite debugging e otimização  
✅ **Escalável**: Suporta arquiteturas complexas  

Bindings bem configurados são essenciais para um sistema de mensageria eficiente e confiável.

---

**Conclusão dos Conceitos Fundamentais**: Com Producer, Queue, Consumer, Exchange e Binding você tem todos os elementos para construir arquiteturas robustas de mensageria com RabbitMQ!
