# RabbitMQ: Consumer (Consumidor) - Conceitos Fundamentais

## O que é um Consumer?

O **Consumer** (Consumidor) é a aplicação ou componente responsável por **receber e processar mensagens** das queues do RabbitMQ. É o ponto final do fluxo de mensagens, onde a lógica de negócio é executada.

## 🎯 **Analogia do Mundo Real**

Imagine um **call center**:
- **Fila de chamadas** = Queue
- **Atendente** = Consumer
- **Atender chamada** = Processar mensagem
- **Finalizar atendimento** = Acknowledgment
- **Repassar para supervisor** = Dead Letter Queue

**Imagem sugerida:** Call center com atendentes (consumers) pegando chamadas de uma fila central, com indicadores visuais de sucesso/falha no atendimento.

## 📊 **Diagrama de Funcionamento**

```
[Queue: msg1|msg2|msg3] ──→ [Consumer] ──→ [Processa Mensagem] 
                              ↓
                         [ACK/NACK/REJECT]
                              ↓
                    [Mensagem removida da queue]
                              OU
                    [Mensagem volta para queue/DLX]
```

**Imagem sugerida:** Fluxograma colorido mostrando o caminho da mensagem através do consumer, com diferentes cores para sucesso (verde) e falha (vermelho).

## 🔧 **Tipos de Consumer**

### 1. **Push Consumer (Padrão)**
O RabbitMQ **"empurra"** mensagens para o consumer.

```java
@RabbitListener(queues = "orders.processing")
public void processOrder(Order order) {
    log.info("Processando pedido: {}", order.getId());
    orderService.processar(order);
    log.info("Pedido processado com sucesso: {}", order.getId());
}
```

### 2. **Pull Consumer (Síncrono)**
O consumer **"puxa"** mensagens quando está pronto.

```java
@Service
public class PullConsumerService {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    @Scheduled(fixedDelay = 5000) // A cada 5 segundos
    public void pullMessages() {
        Message message = rabbitTemplate.receive("orders.processing", 1000); // Timeout 1s
        
        if (message != null) {
            String content = new String(message.getBody());
            processMessage(content);
        }
    }
    
    private void processMessage(String content) {
        // Processar mensagem
        log.info("Mensagem processada: {}", content);
    }
}
```

## ⚙️ **Acknowledgments (ACK)**

### Manual Acknowledgment
```java
@RabbitListener(queues = "orders.processing", 
               ackMode = "MANUAL")
public void processOrderManual(Order order, 
                              @Header Map<String, Object> headers,
                              Channel channel,
                              @Header("amqp_deliveryTag") long deliveryTag) {
    try {
        // Processar pedido
        orderService.processar(order);
        
        // ✅ Confirmar processamento bem-sucedido
        channel.basicAck(deliveryTag, false);
        log.info("Pedido processado e confirmado: {}", order.getId());
        
    } catch (BusinessException e) {
        try {
            // ❌ Rejeitar e reenviar para queue
            channel.basicNack(deliveryTag, false, true);
            log.warn("Erro de negócio, reenviando: {}", e.getMessage());
        } catch (IOException ioException) {
            log.error("Erro ao fazer NACK: {}", ioException.getMessage());
        }
        
    } catch (Exception e) {
        try {
            // ❌ Rejeitar e enviar para DLX
            channel.basicNack(deliveryTag, false, false);
            log.error("Erro crítico, enviando para DLX: {}", e.getMessage());
        } catch (IOException ioException) {
            log.error("Erro ao fazer NACK: {}", ioException.getMessage());
        }
    }
}
```

### Auto Acknowledgment (Padrão)
```java
@RabbitListener(queues = "notifications.email")
public void sendEmail(EmailRequest request) {
    // ACK automático após retorno do método
    emailService.send(request);
    // Se exception ocorrer, NACK automático
}
```

## 🚀 **Configurações de Performance**

### Prefetch Count
```java
@Bean
public SimpleRabbitListenerContainerFactory containerFactory() {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    
    // Quantas mensagens não confirmadas cada consumer pode ter
    factory.setPrefetchCount(10);  // ✅ Balanceia performance vs memory
    
    // Número de consumers concurrent
    factory.setConcurrentConsumers(2);     // Mínimo
    factory.setMaxConcurrentConsumers(10); // Máximo
    
    return factory;
}
```

### Consumers Específicos por Performance
```java
// Consumer rápido - Muitas mensagens simultâneas
@RabbitListener(queues = "fast.processing", 
               containerFactory = "fastContainerFactory")
public void fastProcessing(String message) {
    // Processamento rápido
}

@Bean
public SimpleRabbitListenerContainerFactory fastContainerFactory() {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setPrefetchCount(50);         // Muitas mensagens
    factory.setConcurrentConsumers(5);
    factory.setMaxConcurrentConsumers(20);
    return factory;
}

// Consumer lento - Poucas mensagens por vez
@RabbitListener(queues = "slow.processing", 
               containerFactory = "slowContainerFactory")
public void slowProcessing(String message) {
    // Processamento lento (ex: API externa)
}

@Bean
public SimpleRabbitListenerContainerFactory slowContainerFactory() {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setPrefetchCount(1);          // Uma mensagem por vez
    factory.setConcurrentConsumers(2);
    factory.setMaxConcurrentConsumers(5);
    return factory;
}
```

## 💻 **Exemplos Práticos**

### Java/Spring Boot Completo
```java
@Component
@Slf4j
public class OrderConsumer {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private NotificationService notificationService;
    
    // Consumer principal de pedidos
    @RabbitListener(queues = "orders.processing")
    public void processOrder(Order order,
                           @Header Map<String, Object> headers) {
        
        String correlationId = (String) headers.get("correlation-id");
        MDC.put("correlationId", correlationId);
        
        try {
            log.info("Iniciando processamento do pedido: {}", order.getId());
            
            // Validar pedido
            orderService.validate(order);
            
            // Processar pagamento
            PaymentResult result = paymentService.processPayment(order);
            
            if (result.isSuccess()) {
                // Confirmar pedido
                orderService.confirm(order);
                
                // Enviar notificação
                notificationService.sendOrderConfirmation(order);
                
                log.info("Pedido processado com sucesso: {}", order.getId());
            } else {
                throw new PaymentException("Falha no pagamento: " + result.getError());
            }
            
        } catch (ValidationException e) {
            log.warn("Pedido inválido: {}", e.getMessage());
            // Não reprocessar, enviar para queue de pedidos inválidos
            throw new AmqpRejectAndDontRequeueException("Pedido inválido", e);
            
        } catch (PaymentException e) {
            log.error("Erro no pagamento: {}", e.getMessage());
            // Reprocessar depois (retry com backoff)
            throw e;
            
        } catch (Exception e) {
            log.error("Erro inesperado ao processar pedido: {}", e.getMessage(), e);
            throw new AmqpRejectAndDontRequeueException("Erro crítico", e);
        } finally {
            MDC.clear();
        }
    }
    
    // Consumer para reprocessamento com delay
    @RabbitListener(queues = "orders.retry")
    public void retryOrder(Order order,
                          @Header Map<String, Object> headers) {
        
        int retryCount = (int) headers.getOrDefault("x-retry-count", 0);
        
        if (retryCount >= 3) {
            log.error("Máximo de tentativas excedido para pedido: {}", order.getId());
            // Enviar para queue de falhas
            return;
        }
        
        try {
            processOrder(order, headers);
        } catch (Exception e) {
            // Incrementar contador e reenviar com delay maior
            headers.put("x-retry-count", retryCount + 1);
            int delay = (retryCount + 1) * 30000; // 30s, 60s, 90s
            
            rabbitTemplate.convertAndSend("orders.retry.delayed", order, 
                                        message -> {
                                            message.getMessageProperties().setDelay(delay);
                                            return message;
                                        });
        }
    }
}
```

### Python (Celery-style)
```python
import pika
import json
import logging
from typing import Dict, Any
from dataclasses import dataclass

@dataclass
class OrderProcessor:
    def __init__(self, connection_params: dict):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(**connection_params)
        )
        self.channel = self.connection.channel()
        self.setup_queues()
    
    def setup_queues(self):
        # Declarar queues
        self.channel.queue_declare(queue='orders.processing', durable=True)
        self.channel.queue_declare(queue='orders.failed', durable=True)
        
        # Configurar prefetch
        self.channel.basic_qos(prefetch_count=10)
    
    def process_order(self, ch, method, properties, body):
        """Processar pedido individual"""
        try:
            # Parse da mensagem
            order_data = json.loads(body.decode('utf-8'))
            correlation_id = properties.correlation_id
            
            logging.info(f"Processando pedido {order_data['id']} - {correlation_id}")
            
            # Validar pedido
            self._validate_order(order_data)
            
            # Processar pagamento
            payment_result = self._process_payment(order_data)
            
            if payment_result['success']:
                # Confirmar pedido
                self._confirm_order(order_data)
                
                # ACK - mensagem processada com sucesso
                ch.basic_ack(delivery_tag=method.delivery_tag)
                logging.info(f"Pedido {order_data['id']} processado com sucesso")
            else:
                raise PaymentException(payment_result['error'])
                
        except ValidationException as e:
            # Erro de validação - não reprocessar
            logging.warning(f"Pedido inválido: {e}")
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
            
        except PaymentException as e:
            # Erro de pagamento - reprocessar depois
            logging.error(f"Erro no pagamento: {e}")
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=True)
            
        except Exception as e:
            # Erro crítico - enviar para DLX
            logging.error(f"Erro crítico: {e}")
            ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
    
    def start_consuming(self):
        """Iniciar consumo de mensagens"""
        self.channel.basic_consume(
            queue='orders.processing',
            on_message_callback=self.process_order
        )
        
        logging.info("Aguardando mensagens. Para sair, pressione CTRL+C")
        
        try:
            self.channel.start_consuming()
        except KeyboardInterrupt:
            self.channel.stop_consuming()
            self.connection.close()
            logging.info("Consumer finalizado")
    
    def _validate_order(self, order_data: Dict[str, Any]):
        """Validar dados do pedido"""
        required_fields = ['id', 'customer_id', 'items', 'total']
        for field in required_fields:
            if field not in order_data:
                raise ValidationException(f"Campo obrigatório ausente: {field}")
    
    def _process_payment(self, order_data: Dict[str, Any]) -> Dict[str, Any]:
        """Processar pagamento"""
        # Simular processamento de pagamento
        # Em produção, chamaria API de pagamento
        return {'success': True, 'transaction_id': 'txn_123'}
    
    def _confirm_order(self, order_data: Dict[str, Any]):
        """Confirmar pedido"""
        # Atualizar status no banco de dados
        pass

class ValidationException(Exception):
    pass

class PaymentException(Exception):
    pass

# Usar o consumer
if __name__ == "__main__":
    processor = OrderProcessor({
        'host': 'localhost',
        'port': 5672,
        'virtual_host': '/',
        'credentials': pika.PlainCredentials('admin', 'password')
    })
    
    processor.start_consuming()
```

### Node.js (Worker Pool)
```javascript
const amqp = require('amqplib');
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

class OrderConsumer {
    constructor(connectionUrl, options = {}) {
        this.connectionUrl = connectionUrl;
        this.options = {
            prefetch: options.prefetch || 10,
            workerPoolSize: options.workerPoolSize || 4,
            ...options
        };
        this.workers = [];
        this.connection = null;
        this.channel = null;
    }
    
    async connect() {
        this.connection = await amqp.connect(this.connectionUrl);
        this.channel = await this.connection.createChannel();
        
        // Configurar prefetch
        await this.channel.prefetch(this.options.prefetch);
        
        // Declarar queues
        await this.channel.assertQueue('orders.processing', { durable: true });
        await this.channel.assertQueue('orders.failed', { durable: true });
        
        console.log('Consumer conectado ao RabbitMQ');
    }
    
    async startConsuming() {
        // Criar pool de workers
        await this.createWorkerPool();
        
        await this.channel.consume('orders.processing', async (message) => {
            if (message) {
                try {
                    const orderData = JSON.parse(message.content.toString());
                    
                    // Enviar para worker disponível
                    const result = await this.processInWorker(orderData);
                    
                    if (result.success) {
                        // ACK
                        this.channel.ack(message);
                        console.log(`Pedido ${orderData.id} processado com sucesso`);
                    } else {
                        // NACK - requeue baseado no tipo de erro
                        const requeue = result.errorType !== 'validation';
                        this.channel.nack(message, false, requeue);
                        console.error(`Erro ao processar pedido ${orderData.id}: ${result.error}`);
                    }
                    
                } catch (error) {
                    console.error('Erro ao processar mensagem:', error);
                    this.channel.nack(message, false, false); // Enviar para DLX
                }
            }
        });
        
        console.log('Aguardando mensagens...');
    }
    
    async createWorkerPool() {
        for (let i = 0; i < this.options.workerPoolSize; i++) {
            const worker = new Worker(__filename, {
                workerData: { isWorker: true }
            });
            
            this.workers.push({
                worker,
                busy: false
            });
        }
    }
    
    async processInWorker(orderData) {
        // Encontrar worker disponível
        const availableWorker = this.workers.find(w => !w.busy);
        
        if (!availableWorker) {
            throw new Error('Nenhum worker disponível');
        }
        
        availableWorker.busy = true;
        
        return new Promise((resolve, reject) => {
            availableWorker.worker.postMessage(orderData);
            
            availableWorker.worker.once('message', (result) => {
                availableWorker.busy = false;
                resolve(result);
            });
            
            availableWorker.worker.once('error', (error) => {
                availableWorker.busy = false;
                reject(error);
            });
        });
    }
    
    async close() {
        // Fechar workers
        for (const { worker } of this.workers) {
            await worker.terminate();
        }
        
        if (this.channel) await this.channel.close();
        if (this.connection) await this.connection.close();
    }
}

// Worker thread logic
if (!isMainThread && workerData.isWorker) {
    parentPort.on('message', async (orderData) => {
        try {
            // Simular processamento
            await new Promise(resolve => setTimeout(resolve, Math.random() * 1000));
            
            // Validar pedido
            if (!orderData.id || !orderData.customer_id) {
                parentPort.postMessage({
                    success: false,
                    errorType: 'validation',
                    error: 'Dados inválidos'
                });
                return;
            }
            
            // Processar pagamento
            const paymentSuccess = Math.random() > 0.1; // 90% sucesso
            
            if (!paymentSuccess) {
                parentPort.postMessage({
                    success: false,
                    errorType: 'payment',
                    error: 'Falha no pagamento'
                });
                return;
            }
            
            parentPort.postMessage({
                success: true,
                orderId: orderData.id
            });
            
        } catch (error) {
            parentPort.postMessage({
                success: false,
                errorType: 'system',
                error: error.message
            });
        }
    });
}

// Usar o consumer
if (isMainThread && !workerData) {
    const consumer = new OrderConsumer('amqp://localhost:5672', {
        prefetch: 20,
        workerPoolSize: 6
    });
    
    consumer.connect()
        .then(() => consumer.startConsuming())
        .catch(console.error);
    
    // Graceful shutdown
    process.on('SIGINT', async () => {
        console.log('Finalizando consumer...');
        await consumer.close();
        process.exit(0);
    });
}
```

## 🛡️ **Tratamento de Erros e Retry**

### Estratégia de Retry com Backoff
```java
@Component
public class RetryableConsumer {
    
    private static final int MAX_RETRIES = 3;
    
    @Retryable(
        value = {TransientException.class},
        maxAttempts = MAX_RETRIES,
        backoff = @Backoff(delay = 1000, multiplier = 2) // 1s, 2s, 4s
    )
    @RabbitListener(queues = "orders.processing")
    public void processOrder(Order order) {
        try {
            orderService.process(order);
        } catch (DatabaseException e) {
            log.warn("Erro temporário, tentando novamente: {}", e.getMessage());
            throw new TransientException("Erro temporário", e);
        } catch (ValidationException e) {
            log.error("Erro de validação, não retentando: {}", e.getMessage());
            throw new AmqpRejectAndDontRequeueException("Validação falhou", e);
        }
    }
    
    @Recover
    public void recover(TransientException ex, Order order) {
        log.error("Máximo de tentativas excedido para pedido: {}", order.getId());
        // Enviar para queue de falhas ou alertar equipe
        failedOrderService.handleFailedOrder(order, ex);
    }
}
```

### Circuit Breaker Pattern
```java
@Component
public class ResilientConsumer {
    
    @CircuitBreaker(name = "payment-service", fallbackMethod = "fallbackPayment")
    @RabbitListener(queues = "payments.processing")
    public void processPayment(PaymentRequest request) {
        // Chama serviço externo que pode falhar
        externalPaymentService.processPayment(request);
    }
    
    public void fallbackPayment(PaymentRequest request, Exception ex) {
        log.warn("Circuit breaker ativo, enviando para reprocessamento: {}", ex.getMessage());
        
        // Reenviar para queue com delay
        rabbitTemplate.convertAndSend("payments.delayed", request,
            message -> {
                message.getMessageProperties().setDelay(30000); // 30s delay
                return message;
            });
    }
}
```

## 📊 **Monitoramento de Consumers**

### Métricas Customizadas
```java
@Component
@Slf4j
public class MonitoredConsumer {
    
    private final MeterRegistry meterRegistry;
    private final Counter processedMessages;
    private final Counter errorMessages;
    private final Timer processingTime;
    
    public MonitoredConsumer(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.processedMessages = Counter.builder("rabbitmq.consumer.processed")
                                       .description("Mensagens processadas com sucesso")
                                       .register(meterRegistry);
        this.errorMessages = Counter.builder("rabbitmq.consumer.errors")
                                   .description("Mensagens com erro")
                                   .register(meterRegistry);
        this.processingTime = Timer.builder("rabbitmq.consumer.processing.time")
                                  .description("Tempo de processamento")
                                  .register(meterRegistry);
    }
    
    @RabbitListener(queues = "orders.processing")
    public void processOrder(Order order) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        try {
            orderService.process(order);
            processedMessages.increment();
            log.info("Pedido processado: {}", order.getId());
            
        } catch (Exception e) {
            errorMessages.increment();
            log.error("Erro ao processar pedido: {}", e.getMessage());
            throw e;
        } finally {
            sample.stop(processingTime);
        }
    }
}
```

### Health Check
```java
@Component
public class ConsumerHealthIndicator implements HealthIndicator {
    
    @Autowired
    private RabbitAdmin rabbitAdmin;
    
    @Override
    public Health health() {
        try {
            // Verificar conexão
            Properties props = rabbitAdmin.getQueueProperties("orders.processing");
            
            if (props == null) {
                return Health.down()
                           .withDetail("queue", "Queue não encontrada")
                           .build();
            }
            
            int messageCount = (int) props.get("QUEUE_MESSAGE_COUNT");
            int consumerCount = (int) props.get("QUEUE_CONSUMER_COUNT");
            
            Health.Builder health = Health.up()
                                        .withDetail("messageCount", messageCount)
                                        .withDetail("consumerCount", consumerCount);
            
            // Alertar se muitas mensagens pendentes
            if (messageCount > 1000) {
                health.withDetail("warning", "Muitas mensagens pendentes");
            }
            
            // Alertar se não há consumers
            if (consumerCount == 0) {
                return health.down()
                           .withDetail("error", "Nenhum consumer ativo")
                           .build();
            }
            
            return health.build();
            
        } catch (Exception e) {
            return Health.down()
                       .withDetail("error", e.getMessage())
                       .build();
        }
    }
}
```

## 📚 **Resumo**

O **Consumer** é responsável pelo processamento efetivo das mensagens. Pontos importantes:

✅ **Acknowledgments** adequados para garantir confiabilidade  
✅ **Configuração de performance** (prefetch, concorrência)  
✅ **Tratamento de erros** robusto com retry e circuit breaker  
✅ **Monitoramento** contínuo de métricas e health  
✅ **Escalabilidade** horizontal para alta demanda  

Um consumer bem implementado garante que o sistema processe mensagens de forma eficiente e confiável.

---

**Próximo:** Exchange (Roteador) - Como as mensagens são direcionadas para as queues certas
