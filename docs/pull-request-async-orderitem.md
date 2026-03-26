# Processamento Assíncrono — Cálculo de Subtotal (OrderItem)

**Desenvolvedor:** Felipe Caldeira Akryghti  
**Issue:** Processamento Assíncrono com `@Async`  
**Entidade:** `OrderItem`  
**Data:** 24/03/2026

---

## 1. Descrição do Processo Assíncrono

O cálculo do **subtotal** de cada item de pedido (`subtotal = quantity × unitPrice`) foi extraído da thread principal e delegado a um serviço assíncrono.

Anteriormente, o cálculo era feito de forma síncrona dentro do `OrderItemController` antes da resposta ao cliente. Agora, o item é **persistido imediatamente sem subtotal**, a resposta é devolvida ao cliente, e o cálculo é executado em **background por uma thread separada**.

---

## 2. Estratégia Adotada

**`@Async` do Spring Framework**

- `@EnableAsync` habilitado na classe principal `RoupasApplication.java`
- Serviço dedicado `OrderItemAsyncService.java` no pacote `com.unilopers.roupas.async`
- Método `calculateSubtotalAsync()` anotado com `@Async`
- Logs com `@Slf4j` para rastreamento completo da execução assíncrona

---

## 3. Justificativa Técnica

Embora o cálculo `quantity × unitPrice` seja simples, a escolha por processamento assíncrono foi **arquitetural**:

- **Separação de responsabilidades:** a persistência (controller) é desacoplada do cálculo de negócio (service)
- **Possibilidade de evolução futura:** impostos, descontos progressivos, regras complexas por categoria
- **Simulação de sistemas distribuídos:** em sistemas reais, cálculos de preço são frequentemente delegados a microserviços
- **Redução de carga na thread principal:** em cenários com muitos itens criados simultaneamente, a thread HTTP fica livre mais rápido
- **Padrão de consistência eventual:** o cliente recebe resposta imediata e o dado é atualizado em background

---

## 4. Fluxo de Execução

```
POST /api/orderitem/create
  │
  ├─ Valida order.orderId e product.productId
  ├─ Busca Order e Product no banco
  ├─ Cria OrderItem (quantity + unitPrice, SEM subtotal)
  ├─ Persiste no banco (save)
  ├─ Retorna OrderItem ao cliente (HTTP 201) ← resposta imediata
  │
  └─ [THREAD ASSÍNCRONA]
       ├─ Log: "[ASYNC] Iniciando cálculo de subtotal para OrderItem {id}"
       ├─ Busca OrderItem no banco pelo ID
       ├─ Calcula: subtotal = quantity × unitPrice
       ├─ Atualiza OrderItem com subtotal
       ├─ Persiste no banco (save)
       └─ Log: "[ASYNC] Subtotal calculado para OrderItem {id}: {qty} x {price} = {subtotal}"
```

O mesmo fluxo se aplica ao `PUT /api/orderitem/update/{id}`.

---

## 5. Arquivos Alterados/Criados

| Arquivo | Ação | Descrição |
|---------|------|-----------|
| `RoupasApplication.java` | Modificado | Adicionado `@EnableAsync` |
| `async/OrderItemAsyncService.java` | **Criado** | Serviço com `@Async` para cálculo de subtotal |
| `controller/OrderItemController.java` | Modificado | Removido cálculo síncrono, injetado `OrderItemAsyncService`, delegação ao async no `create` e `update` |

---

## 6. Impactos Observados

### Positivos
- **Resposta mais rápida:** o cliente recebe o `OrderItem` imediatamente sem aguardar o cálculo
- **Desacoplamento:** lógica de negócio separada da lógica de persistência
- **Rastreabilidade:** logs `[ASYNC]` permitem acompanhar cada execução em background
- **Escalabilidade:** em cenários com criação massiva de itens, a thread HTTP é liberada mais rápido

### Consistência Eventual
- Na resposta imediata do `POST`, o campo `subtotal` retorna `null`
- Após milissegundos, o subtotal é calculado e persistido
- Consultas subsequentes (`GET /api/orderitem/{id}`) retornam o subtotal correto

---

## 9. Possíveis Falhas

| Cenário | Impacto | Mitigação |
|---------|---------|-----------|
| Falha ao buscar OrderItem no async | Subtotal fica null | Log `[WARN]` registrado para monitoramento |
| Exceção durante o cálculo | Subtotal não atualizado | Spring captura e loga a exceção |
| Consulta imediata após criação | Subtotal ainda null | Comportamento esperado (consistência eventual) |
| Thread pool esgotado | Cálculos enfileirados | Configurar `spring.task.execution.pool.*` se necessário |

---

## 10. Conclusão

A implementação do processamento assíncrono para o cálculo de subtotal do `OrderItem` demonstra na prática o padrão de **consistência eventual** e **separação de responsabilidades**. Embora o cálculo atual seja simples (`quantity × unitPrice`), a arquitetura está preparada para evoluir com regras de negócio mais complexas (descontos, impostos, promoções) sem impactar o tempo de resposta da API.

O uso de `@Async` com logs `@Slf4j` garante rastreabilidade completa, e o fluxo foi validado com testes de API confirmando que:
- A resposta imediata retorna `subtotal = null`
- Após o processamento assíncrono, o subtotal é calculado corretamente

---

## 11. Como Testar

### Pré-requisitos
- Servidor rodando em `localhost:8080`
- Pelo menos 1 usuário, 1 pedido e 1 produto cadastrados

### Passo a passo com curl

```bash
# 1) Login
curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/xml" \
  -d "<authRequest><email>admin@louisvittao.com</email><password>admin123</password></authRequest>"
# Copie o token da resposta

# 2) Criar OrderItem (substitua ORDER_ID, PRODUCT_ID e TOKEN)
curl -s -X POST http://localhost:8080/api/orderitem/create \
  -H "Content-Type: application/xml" \
  -H "Authorization: Bearer TOKEN" \
  -d "<OrderItem><order><orderId>ORDER_ID</orderId></order><product><productId>PRODUCT_ID</productId></product><quantity>5</quantity><unitPrice>29.90</unitPrice></OrderItem>"
# Observe: subtotal deve estar ausente/null na resposta

# 3) Consultar OrderItem (substitua ITEM_ID e TOKEN)
curl -s -X GET http://localhost:8080/api/orderitem/ITEM_ID \
  -H "Authorization: Bearer TOKEN"
# Agora subtotal deve ser 149.5 (5 × 29.90)
```

### Verificação nos logs do servidor

Procure no console do Spring Boot:
```
[ASYNC] Iniciando cálculo de subtotal para OrderItem <uuid>
[ASYNC] Subtotal calculado para OrderItem <uuid>: 5 x 29.9 = 149.5
```

---

Ref #12
