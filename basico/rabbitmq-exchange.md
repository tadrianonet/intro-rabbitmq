# RabbitMQ: Exchange (Roteador) - Guia Detalhado

## O que é um Exchange?

O **Exchange** é o componente central do RabbitMQ responsável por **rotear mensagens** para as queues apropriadas. É o "cerebro" do sistema que decide onde cada mensagem deve ir baseado em regras de roteamento.

## 🎯 **Analogia do Mundo Real**

Imagine uma **central de distribuição postal**:
- **Remetente** = Producer
- **Carta** = Mensagem
- **Central de Triagem** = Exchange
- **Endereço/CEP** = Routing Key
- **Caixa Postal** = Queue
- **Destinatário** = Consumer

O exchange é como o funcionário que olha o endereço da carta e decide para qual setor de entrega direcioná-la.

## 📊 **Diagrama Conceitual**

```
Producer ──→ [Exchange] ──┬──→ Queue A ──→ Consumer A
                          ├──→ Queue B ──→ Consumer B  
                          └──→ Queue C ──→ Consumer C

               ↑
        Routing Rules
     (binding + routing key)
```

**Sugestão de Imagem:** Diagrama mostrando um exchange central recebendo mensagens e distribuindo para múltiplas queues baseado em regras.

## 🔧 **Tipos de Exchange**

### 1. **Direct Exchange**

Roteia mensagens baseado na **routing key exata**.

```java
@Bean
public DirectExchange orderExchange() {
    return ExchangeBuilder.directExchange("orders.direct")
                         .durable(true)
                         .build();
}

@Bean
public Binding orderProcessingBinding() {
    return BindingBuilder.bind(orderProcessingQueue())
                        .to(orderExchange())
                        .with("order.created");  // Routing key exata
}

@Bean
public Binding orderCancelBinding() {
    return BindingBuilder.bind(orderCancelQueue())
                        .to(orderExchange())
                        .with("order.cancelled"); // Routing key exata
}
```

**Exemplo de uso:**
```java
// Mensagem vai para orderProcessingQueue
rabbitTemplate.convertAndSend("orders.direct", "order.created", order);

// Mensagem vai para orderCancelQueue  
rabbitTemplate.convertAndSend("orders.direct", "order.cancelled", order);
```

**Diagrama Direct Exchange:**
```
Producer ──→ [Direct Exchange] ──┬─ "order.created" ──→ Queue Processing
                                 ├─ "order.cancelled" ──→ Queue Cancel
                                 └─ "order.updated" ──→ Queue Updates
```

### 2. **Topic Exchange**

Permite roteamento com **padrões** usando wildcards.

**Wildcards:**
- `*` = Substitui **exatamente uma palavra**
- `#` = Substitui **zero ou mais palavras**

```java
@Bean
public TopicExchange notificationExchange() {
    return ExchangeBuilder.topicExchange("notifications.topic")
                         .durable(true)
                         .build();
}

@Bean
public Binding emailNotificationBinding() {
    return BindingBuilder.bind(emailQueue())
                        .to(notificationExchange())
                        .with("notification.*.email");  // user.email, order.email
}

@Bean
public Binding smsNotificationBinding() {
    return BindingBuilder.bind(smsQueue())
                        .to(notificationExchange())
                        .with("notification.*.sms");    // user.sms, order.sms
}

@Bean
public Binding criticalNotificationBinding() {
    return BindingBuilder.bind(criticalQueue())
                        .to(notificationExchange())
                        .with("notification.critical.#"); // Qualquer notificação crítica
}
```

**Exemplos de roteamento:**
```java
// → email queue (match: notification.*.email)
rabbitTemplate.convertAndSend("notifications.topic", "notification.user.email", message);

// → sms queue (match: notification.*.sms)  
rabbitTemplate.convertAndSend("notifications.topic", "notification.order.sms", message);

// → critical queue (match: notification.critical.#)
rabbitTemplate.convertAndSend("notifications.topic", "notification.critical.system.alert", message);

// → critical queue E email queue (multiple matches)
rabbitTemplate.convertAndSend("notifications.topic", "notification.critical.email", message);
```

**Diagrama Topic Exchange:**
```
Producer ──→ [Topic Exchange] ──┬─ "notification.*.email" ──→ Email Queue
                                ├─ "notification.*.sms" ──→ SMS Queue  
                                ├─ "notification.critical.#" ──→ Critical Queue
                                └─ "log.#" ──→ Log Queue
```

### 3. **Fanout Exchange**

Envia mensagens para **todas** as queues conectadas (broadcast).

```java
@Bean
public FanoutExchange broadcastExchange() {
    return ExchangeBuilder.fanoutExchange("events.fanout")
                         .durable(true)
                         .build();
}

@Bean
public Binding auditBinding() {
    return BindingBuilder.bind(auditQueue())
                        .to(broadcastExchange());
}

@Bean
public Binding analyticsBinding() {
    return BindingBuilder.bind(analyticsQueue())
                        .to(broadcastExchange());
}

@Bean
public Binding cacheBinding() {
    return BindingBuilder.bind(cacheQueue())
                        .to(broadcastExchange());
}
```

**Exemplo de uso:**
```java
// Mensagem vai para TODAS as queues (audit, analytics, cache)
UserEvent event = new UserEvent("user.created", user);
rabbitTemplate.convertAndSend("events.fanout", "", event);
// Routing key é ignorada no fanout
```

**Diagrama Fanout Exchange:**
```
Producer ──→ [Fanout Exchange] ──┬──→ Audit Queue
                                 ├──→ Analytics Queue
                                 ├──→ Cache Queue
                                 └──→ Notification Queue
                     (Todas recebem a mensagem)
```

### 4. **Headers Exchange**

Roteia baseado nos **headers** da mensagem, não na routing key.

```java
@Bean
public HeadersExchange headersExchange() {
    return ExchangeBuilder.headersExchange("routing.headers")
                         .durable(true)
                         .build();
}

@Bean
public Binding priorityBinding() {
    Map<String, Object> headers = new HashMap<>();
    headers.put("priority", "high");
    headers.put("type", "order");
    
    return BindingBuilder.bind(priorityQueue())
                        .to(headersExchange())
                        .whereAll(headers).match(); // TODOS os headers devem bater
}

@Bean
public Binding typeBinding() {
    Map<String, Object> headers = new HashMap<>();
    headers.put("type", "notification");
    
    return BindingBuilder.bind(notificationQueue())
                        .to(headersExchange())
                        .whereAny(headers).match(); // QUALQUER header deve bater
}
```

**Exemplo de uso:**
```java
public void sendPriorityOrder(Order order) {
    MessageProperties props = new MessageProperties();
    props.setHeader("priority", "high");
    props.setHeader("type", "order");
    props.setHeader("customer-tier", "premium");
    
    Message message = new Message(serialize(order), props);
    rabbitTemplate.send("routing.headers", "", message);
    // → Vai para priorityQueue (priority=high AND type=order)
}

public void sendNotification(String notification) {
    MessageProperties props = new MessageProperties();
    props.setHeader("type", "notification");
    props.setHeader("urgency", "normal");
    
    Message message = new Message(notification.getBytes(), props);
    rabbitTemplate.send("routing.headers", "", message);
    // → Vai para notificationQueue (type=notification)
}
```

## 🔄 **Exchanges Avançados**

### Consistent Hash Exchange
Para distribuição baseada em hash (load balancing).

```java
@Bean
public CustomExchange hashExchange() {
    Map<String, Object> args = new HashMap<>();
    args.put("hash-header", "user_id");  // Header para hash
    
    return new CustomExchange("user.hash", "x-consistent-hash", true, false, args);
}

// As mensagens serão distribuídas baseado no hash do user_id
public void sendUserEvent(UserEvent event) {
    MessageProperties props = new MessageProperties();
    props.setHeader("user_id", event.getUserId());
    
    Message message = new Message(serialize(event), props);
    rabbitTemplate.send("user.hash", "", message);
}
```

### Delayed Exchange
Para mensagens com delay programado.

```java
@Bean
public CustomExchange delayedExchange() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-delayed-type", "direct");
    
    return new CustomExchange("orders.delayed", "x-delayed-message", true, false, args);
}

// Enviar mensagem com delay de 5 minutos
public void scheduleOrderReminder(Order order) {
    rabbitTemplate.convertAndSend("orders.delayed", "reminder", order, 
        message -> {
            message.getMessageProperties().setDelay(300000); // 5 min
            return message;
        });
}
```

## 💻 **Configuração Completa - Exemplo E-commerce**

```java
@Configuration
@EnableRabbit
public class EcommerceExchangeConfig {
    
    // =================== DIRECT EXCHANGES ===================
    
    @Bean
    public DirectExchange orderExchange() {
        return ExchangeBuilder.directExchange("ecommerce.orders")
                             .durable(true)
                             .build();
    }
    
    @Bean
    public DirectExchange paymentExchange() {
        return ExchangeBuilder.directExchange("ecommerce.payments")
                             .durable(true)
                             .build();
    }
    
    // =================== TOPIC EXCHANGE ===================
    
    @Bean
    public TopicExchange notificationExchange() {
        return ExchangeBuilder.topicExchange("ecommerce.notifications")
                             .durable(true)
                             .build();
    }
    
    // =================== FANOUT EXCHANGE ===================
    
    @Bean
    public FanoutExchange eventExchange() {
        return ExchangeBuilder.fanoutExchange("ecommerce.events")
                             .durable(true)
                             .build();
    }
    
    // =================== QUEUES ===================
    
    @Bean public Queue orderProcessingQueue() { return QueueBuilder.durable("orders.processing").build(); }
    @Bean public Queue orderCancelQueue() { return QueueBuilder.durable("orders.cancel").build(); }
    @Bean public Queue paymentProcessingQueue() { return QueueBuilder.durable("payments.processing").build(); }
    @Bean public Queue paymentRefundQueue() { return QueueBuilder.durable("payments.refund").build(); }
    @Bean public Queue emailQueue() { return QueueBuilder.durable("notifications.email").build(); }
    @Bean public Queue smsQueue() { return QueueBuilder.durable("notifications.sms").build(); }
    @Bean public Queue pushQueue() { return QueueBuilder.durable("notifications.push").build(); }
    @Bean public Queue auditQueue() { return QueueBuilder.durable("audit.events").build(); }
    @Bean public Queue analyticsQueue() { return QueueBuilder.durable("analytics.events").build(); }
    @Bean public Queue inventoryQueue() { return QueueBuilder.durable("inventory.events").build(); }
    
    // =================== BINDINGS - DIRECT ===================
    
    @Bean
    public Binding orderCreatedBinding() {
        return BindingBuilder.bind(orderProcessingQueue())
                            .to(orderExchange())
                            .with("order.created");
    }
    
    @Bean
    public Binding orderCancelledBinding() {
        return BindingBuilder.bind(orderCancelQueue())
                            .to(orderExchange())
                            .with("order.cancelled");
    }
    
    @Bean
    public Binding paymentRequestBinding() {
        return BindingBuilder.bind(paymentProcessingQueue())
                            .to(paymentExchange())
                            .with("payment.process");
    }
    
    @Bean
    public Binding paymentRefundBinding() {
        return BindingBuilder.bind(paymentRefundQueue())
                            .to(paymentExchange())
                            .with("payment.refund");
    }
    
    // =================== BINDINGS - TOPIC ===================
    
    @Bean
    public Binding emailNotificationBinding() {
        return BindingBuilder.bind(emailQueue())
                            .to(notificationExchange())
                            .with("notification.*.email");
    }
    
    @Bean
    public Binding smsNotificationBinding() {
        return BindingBuilder.bind(smsQueue())
                            .to(notificationExchange())
                            .with("notification.*.sms");
    }
    
    @Bean
    public Binding pushNotificationBinding() {
        return BindingBuilder.bind(pushQueue())
                            .to(notificationExchange())
                            .with("notification.*.push");
    }
    
    // =================== BINDINGS - FANOUT ===================
    
    @Bean
    public Binding auditEventBinding() {
        return BindingBuilder.bind(auditQueue()).to(eventExchange());
    }
    
    @Bean
    public Binding analyticsEventBinding() {
        return BindingBuilder.bind(analyticsQueue()).to(eventExchange());
    }
    
    @Bean
    public Binding inventoryEventBinding() {
        return BindingBuilder.bind(inventoryQueue()).to(eventExchange());
    }
}
```

## 🚀 **Serviços de Negócio Usando Exchanges**

```java
@Service
@Slf4j
public class EcommerceEventService {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    // =================== ORDER EVENTS ===================
    
    public void orderCreated(Order order) {
        log.info("Publicando evento: pedido criado {}", order.getId());
        
        // Direct exchange - processamento específico
        rabbitTemplate.convertAndSend("ecommerce.orders", "order.created", order);
        
        // Fanout exchange - event sourcing
        OrderEvent event = new OrderEvent("ORDER_CREATED", order);
        rabbitTemplate.convertAndSend("ecommerce.events", "", event);
        
        // Topic exchange - notificações
        String routingKey = String.format("notification.order.%s", 
                                        order.getCustomer().getPreferredChannel());
        OrderNotification notification = new OrderNotification(order, "CREATED");
        rabbitTemplate.convertAndSend("ecommerce.notifications", routingKey, notification);
    }
    
    public void orderCancelled(Order order) {
        log.info("Publicando evento: pedido cancelado {}", order.getId());
        
        // Cancelamento
        rabbitTemplate.convertAndSend("ecommerce.orders", "order.cancelled", order);
        
        // Event sourcing
        OrderEvent event = new OrderEvent("ORDER_CANCELLED", order);
        rabbitTemplate.convertAndSend("ecommerce.events", "", event);
        
        // Notificação de cancelamento
        String routingKey = String.format("notification.order.%s", 
                                        order.getCustomer().getPreferredChannel());
        OrderNotification notification = new OrderNotification(order, "CANCELLED");
        rabbitTemplate.convertAndSend("ecommerce.notifications", routingKey, notification);
        
        // Estorno se necessário
        if (order.isPaid()) {
            PaymentRefund refund = new PaymentRefund(order);
            rabbitTemplate.convertAndSend("ecommerce.payments", "payment.refund", refund);
        }
    }
    
    // =================== PAYMENT EVENTS ===================
    
    public void paymentProcessed(Payment payment) {
        log.info("Publicando evento: pagamento processado {}", payment.getId());
        
        if (payment.isSuccessful()) {
            // Confirmar pedido
            OrderConfirmation confirmation = new OrderConfirmation(payment.getOrderId());
            rabbitTemplate.convertAndSend("ecommerce.orders", "order.confirmed", confirmation);
            
            // Notificar sucesso
            String routingKey = "notification.payment.email"; // Sempre email para pagamento
            PaymentNotification notification = new PaymentNotification(payment, "SUCCESS");
            rabbitTemplate.convertAndSend("ecommerce.notifications", routingKey, notification);
        } else {
            // Falha no pagamento
            OrderFailure failure = new OrderFailure(payment.getOrderId(), payment.getErrorMessage());
            rabbitTemplate.convertAndSend("ecommerce.orders", "order.payment.failed", failure);
            
            // Notificar falha
            String routingKey = "notification.payment.email";
            PaymentNotification notification = new PaymentNotification(payment, "FAILED");
            rabbitTemplate.convertAndSend("ecommerce.notifications", routingKey, notification);
        }
        
        // Event sourcing
        PaymentEvent event = new PaymentEvent("PAYMENT_PROCESSED", payment);
        rabbitTemplate.convertAndSend("ecommerce.events", "", event);
    }
    
    // =================== NOTIFICATION EVENTS ===================
    
    public void sendMarketingCampaign(MarketingCampaign campaign) {
        log.info("Enviando campanha de marketing: {}", campaign.getName());
        
        // Enviar para todos os canais
        for (Customer customer : campaign.getTargetCustomers()) {
            // Routing baseado na preferência do cliente
            String routingKey = String.format("notification.marketing.%s", 
                                             customer.getPreferredChannel());
            
            CampaignNotification notification = new CampaignNotification(campaign, customer);
            rabbitTemplate.convertAndSend("ecommerce.notifications", routingKey, notification);
        }
    }
}
```

## 📊 **Monitoramento de Exchanges**

```java
@RestController
@RequestMapping("/admin/exchanges")
public class ExchangeMonitoringController {
    
    @Autowired
    private RabbitAdmin rabbitAdmin;
    
    @GetMapping("/stats")
    public Map<String, ExchangeStats> getExchangeStats() {
        Map<String, ExchangeStats> stats = new HashMap<>();
        
        List<String> exchanges = Arrays.asList(
            "ecommerce.orders",
            "ecommerce.payments", 
            "ecommerce.notifications",
            "ecommerce.events"
        );
        
        for (String exchangeName : exchanges) {
            try {
                // Obter informações do exchange via Management API
                ExchangeStats exchangeStats = getExchangeInfo(exchangeName);
                stats.put(exchangeName, exchangeStats);
            } catch (Exception e) {
                log.error("Erro ao obter stats do exchange {}: {}", exchangeName, e.getMessage());
            }
        }
        
        return stats;
    }
    
    private ExchangeStats getExchangeInfo(String exchangeName) {
        // Implementação via HTTP client para Management API
        // GET http://localhost:15672/api/exchanges/%2F/{exchangeName}
        return ExchangeStats.builder()
                          .name(exchangeName)
                          .type("direct") // ou topic, fanout, headers
                          .messageStats(getMessageStats(exchangeName))
                          .build();
    }
}

@Data
@Builder
public class ExchangeStats {
    private String name;
    private String type;
    private boolean durable;
    private boolean autoDelete;
    private MessageStats messageStats;
}

@Data
@Builder
public class MessageStats {
    private long publishedTotal;
    private double publishedRate;
    private long publishedInDetails;
    private double publishedInRate;
    private long publishedOutDetails;
    private double publishedOutRate;
}
```

## 🛡️ **Boas Práticas**

### 1. **Nomeclatura Consistente**
```java
// ✅ Bom - Padrão consistente
"ecommerce.orders"           // Exchange
"order.created"              // Routing key  
"orders.processing"          // Queue

// ❌ Ruim - Inconsistente
"OrderExchange"              // PascalCase
"order_created"              // snake_case
"processing-orders"          // kebab-case
```

### 2. **Exchanges Duráveis para Produção**
```java
// ✅ Sempre durable em produção
@Bean
public DirectExchange productionExchange() {
    return ExchangeBuilder.directExchange("orders.production")
                         .durable(true)    // ✅ Sobrevive a restart
                         .build();
}

// ❌ Apenas para desenvolvimento/teste
@Bean  
public DirectExchange tempExchange() {
    return ExchangeBuilder.directExchange("orders.temp")
                         .durable(false)   // ❌ Perdido em restart
                         .build();
}
```

### 3. **Separação por Contexto de Negócio**
```java
// ✅ Separar por domínio
@Bean public DirectExchange orderExchange() { ... }      // Pedidos
@Bean public DirectExchange paymentExchange() { ... }    // Pagamentos  
@Bean public DirectExchange inventoryExchange() { ... }  // Estoque
@Bean public FanoutExchange auditExchange() { ... }      // Auditoria

// ❌ Exchange genérico para tudo
@Bean public DirectExchange allPurposeExchange() { ... } // Muito genérico
```

### 4. **Dead Letter Exchange**
```java
@Bean
public DirectExchange mainExchange() {
    return ExchangeBuilder.directExchange("orders.main").durable(true).build();
}

@Bean
public DirectExchange deadLetterExchange() {
    return ExchangeBuilder.directExchange("orders.dlx").durable(true).build();
}

@Bean
public Queue mainQueue() {
    return QueueBuilder.durable("orders.processing")
                      .withArgument("x-dead-letter-exchange", "orders.dlx")
                      .withArgument("x-dead-letter-routing-key", "failed")
                      .build();
}
```

## 📚 **Resumo**

O **Exchange** é o componente que define como as mensagens são roteadas no RabbitMQ:

✅ **Direct Exchange**: Roteamento exato por routing key  
✅ **Topic Exchange**: Roteamento por padrões com wildcards  
✅ **Fanout Exchange**: Broadcast para todas as queues  
✅ **Headers Exchange**: Roteamento por headers da mensagem  
✅ **Exchanges Customizados**: Para necessidades específicas  

A escolha do tipo correto de exchange é fundamental para uma arquitetura eficiente e flexível.

---

**Próximo:** Binding (Ligação) - Como conectar exchanges às queues com regras específicas
