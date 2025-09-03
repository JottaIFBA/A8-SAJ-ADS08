■ Sistema de Pedidos e Pagamentos (Spring Boot
Microservices)
Este projeto apresenta um sistema de Pedidos e Sistema de Pedi desenvolvido em **Spring
Boot** e dividido em dois cenários: 1. Versão inicial (com falhas arquiteturais) 2. Versão refatorada
(com boas práticas: Feign + Resilience4j)
■ Estrutura do Código
src/main/java/com/example ■■■ orders ■ ■■■ OrderController.java #
Controller de pedidos (v1 e v2) ■ ■■■ PaymentClient.java # Feign Client
(apenas v2) ■■■ payments ■■■ PaymentController.java # Serviço de
pagamentos
■ Versão Inicial (com problemas)
OrderController:
@RestController @RequestMapping("/orders") public class OrderController {
private RestTemplate restTemplate = new RestTemplate(); @PostMapping public
String criarPedido(@RequestBody String pedido) { String respostaPagamento =
restTemplate.postForObject( "http://localhost:8082/payments", pedido,
String.class); return "Pedido criado! Resultado pagamento: " +
respostaPagamento; } }
PaymentController:
@RestController @RequestMapping("/payments") public class
PaymentController { private Map pedidos = new HashMap<>(); @PostMapping
public String processarPagamento(@RequestBody String pedido) {
pedidos.put(pedido, "Pago"); return "Pagamento confirmado para: " + pedido;
} }
Problemas identificados:
- Comunicação direta com RestTemplate (sem fallback) - Se o serviço de pagamento cair, o
sistema de pedidos também falha - Viola o princípio SRP (Single Responsibility Principle) no
PaymentController - Alto acoplamento entre serviços
■ Versão Refatorada (com boas práticas)
OrderController:
@RestController @RequestMapping("/orders") public class OrderController {
private final PaymentClient paymentClient; public
OrderController(PaymentClient paymentClient) { this.paymentClient =
paymentClient; } @PostMapping @Bulkhead(name = "orderService", type =
Bulkhead.Type.THREADPOOL) @CircuitBreaker(name = "orderService",
fallbackMethod = "fallbackPagamento") public String
criarPedido(@RequestBody String pedido) { return
paymentClient.processarPagamento(pedido); } public String
fallbackPagamento(String pedido, Throwable ex) { return "Pagamento em fila.
Tentaremos novamente. Erro: " + ex.getMessage(); } }
PaymentClient:
@FeignClient(name = "payment-service") public interface PaymentClient {
@PostMapping("/payments") String processarPagamento(@RequestBody String
pedido); }
Melhorias:
- Comunicação com Feign Client (baixo acoplamento) - CircuitBreaker e Bulkhead garantem
resiliência e fallback - Responsabilidades separadas (SOLID) - Mais fácil de manter e escalar
■ Como Executar1■■ Clonar o projeto git clone
https://github.com/seu-usuario/pedidos-pagamentos.git cd
pedidos-pagamentos 2■■ Subir os serviços ■ Serviço de Pagamentos (porta
8082) mvn spring-boot:run -Dspring-boot.run.arguments="--server.port=8082
--spring.application.name=payment-service" ■ Serviço de Pedidos (porta
8081) mvn spring-boot:run -Dspring-boot.run.arguments="--server.port=8081
--spring.application.name=order-service"
■ Testando a API
Criar um pedido: curl -X POST http://localhost:8081/orders -H
"Content-Type: application/json" -d "Pedido123" Resposta esperada
(normal): Pedido criado! Resultado pagamento: Pagamento confirmado para:
Pedido123 Resposta esperada (se o serviço de pagamento estiver offline):
Pagamento em fila. Tentaremos novamente. Erro:
■ Comparativo das Versões
Aspecto Versão Inicial ■ Versão Refatorada ■
Comunicação RestTemplate Feign Client
Tolerância a falhas Nenhuma Resilience4j
Acoplamento Alto Baixo
Princípios SOLID Violados Atendidos
Escalabilidade Difícil Facilitada
■ Próximos Passos
- Adicionar API Gateway - Usar Service Discovery (Eureka/Consul) - Incluir Observabilidade
