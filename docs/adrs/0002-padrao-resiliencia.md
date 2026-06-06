# ADR 0002 — Padrões de Resiliência

| Campo | Valor |
|---|---|
| **ID** | 0002 |
| **Título** | Adoção de API Gateway + Circuit Breaker + Bulkhead para tolerância a falhas |
| **Status** | Aceita |
| **Data** | 2026-06-05 |
| **Projeto** | LogiTrack — Fase 3 |

---

## 1. Contexto — Conflito de Forças

O LogiTrack integra sistemas externos críticos — **Google Maps Platform** para otimização de rotas e um **ERP externo** para sincronização de pedidos — além de depender de comunicação entre microsserviços internos. Esse contexto cria riscos de falha em cascata: se a API do Google Maps ficar indisponível, o Serviço de Rotas pode acumular threads bloqueadas, esgotando recursos e derrubando serviços adjacentes (Frota, Entregas) que compartilham a mesma infraestrutura.

O incidente de segurança documentado na auditoria PayRoute (Aula 17) reforça que sistemas distribuídos sem mecanismos de isolamento de falha transformam **falhas pontuais em falhas sistêmicas**.

As forças em conflito são:

- **Disponibilidade vs. Consistência:** o sistema deve continuar operando (exibindo rotas em cache ou degradadas) mesmo quando a API de mapas falha, mas isso implica servir dados potencialmente desatualizados.
- **Latência vs. Segurança:** timeouts curtos protegem threads mas podem rejeitar requisições legítimas que simplesmente demoram mais sob carga.
- **Complexidade vs. Resiliência:** cada padrão de resiliência adiciona código, configuração e pontos de observabilidade — o custo arquitetural deve ser proporcional ao risco protegido.

---

## 2. Decisão

Adotar **três padrões de resiliência em camadas**, conforme recomendado por Nygard (2018) em *Release It!*:

### 2.1 API Gateway — Primeira Linha de Defesa

O **Cloud Endpoints / Kong** atuará como API Gateway com as seguintes responsabilidades de resiliência:

- **Rate Limiting:** máximo de 100 req/s por cliente autenticado — protege os microsserviços de sobrecarga por abuso ou bug em cliente.
- **Timeout global:** 30 segundos por requisição — requisições que ultrapassam esse limite são canceladas com `504 Gateway Timeout`, liberando threads.
- **Autenticação centralizada:** validação de JWT com verificação de JTI na blocklist Redis — impede que tokens comprometidos atinjam os microsserviços.
- **Versionamento de rota:** roteamento entre `/v1/` e `/v2/` sem alteração nos serviços downstream.

**Justificativa teórica:** Richardson (2018) em *Microservices Patterns* define o API Gateway como o ponto correto para aplicar concerns transversais (cross-cutting concerns) como autenticação, logging e rate limiting, evitando que cada microsserviço reimplemente essas funcionalidades — o que seria uma violação do princípio DRY e fonte de inconsistências.

### 2.2 Circuit Breaker — Isolamento de Falhas Externas

O padrão **Circuit Breaker** será implementado no **Serviço de Rotas** para proteger as chamadas à Google Maps Platform:

```
Estado FECHADO → Chamadas normais à Google Maps API
       ↓ (5 falhas consecutivas em 10s)
Estado ABERTO → Retorna rota em cache (Redis) ou erro controlado 503
       ↓ (após 30s de espera)
Estado SEMI-ABERTO → Testa 1 chamada real; se OK → volta para FECHADO
```

**Configuração:**
- Biblioteca: `resilience4j` (Java) / `pybreaker` (Python)
- Threshold: 5 falhas consecutivas ou 50% de taxa de erro em janela de 10s
- Tempo em aberto: 30 segundos
- Fallback: retornar última rota calculada armazenada no Redis (TTL 5 min)

**Justificativa teórica:** Nygard (2018, cap. 5) descreve o Circuit Breaker como o padrão fundamental para sistemas distribuídos que integram dependências externas não confiáveis. Sem ele, threads bloqueadas em chamadas à API de mapas esgotam o pool de conexões do Serviço de Rotas, causando falha em cascata para os demais serviços — o fenômeno que Nygard denomina *integration point failure*.

### 2.3 Bulkhead — Isolamento de Recursos por Contexto

O padrão **Bulkhead** (anteparo de navio) será aplicado para separar pools de threads/conexões por contexto de uso:

| Pool | Serviço | Tamanho máximo | Propósito |
|------|---------|----------------|-----------|
| `pool-google-maps` | Svc. Rotas | 20 threads | Chamadas à API externa |
| `pool-db-rotas` | Svc. Rotas | 10 conexões | Acesso ao Cloud SQL |
| `pool-websocket` | Svc. Rastreio | 5.000 conexões | Conexões GPS dos motoristas |
| `pool-erp` | Svc. Entregas | 5 threads | Sincronização com ERP externo |

**Justificativa teórica:** Nygard (2018) usa a metáfora dos anteparos de navio: um navio sem anteparos afunda completamente se o casco é perfurado; com anteparos, apenas um compartimento é perdido. A mesma lógica se aplica a pools de threads — sem Bulkhead, a lentidão do ERP externo pode esgotar todas as threads do Serviço de Entregas, derrubando inclusive operações que não dependem do ERP.

---

## 3. Alternativas Consideradas e Rejeitadas

### 3.1 Retry com backoff exponencial como único mecanismo

**Rejeitado como solução isolada porque:** retries sem Circuit Breaker amplificam a carga sobre um sistema já degradado. Se a Google Maps API está falhando, 3 retries por requisição triplicam o número de chamadas em um momento crítico — o oposto do comportamento desejado. Retries são mantidos apenas dentro do estado SEMI-ABERTO do Circuit Breaker, com backoff exponencial.

### 3.2 Timeout sem Circuit Breaker

**Rejeitado como solução isolada porque:** timeouts evitam que threads fiquem bloqueadas indefinidamente, mas não previnem que o sistema continue tentando chamar um serviço que claramente está falhando. O Circuit Breaker complementa o timeout ao detectar o padrão de falha e abrir o circuito proativamente.

### 3.3 Service Mesh (Istio) para Circuit Breaker

**Rejeitado para esta fase porque:** Istio adiciona complexidade operacional significativa (sidecar proxy em cada pod, configuração de CRDs Kubernetes) desproporcional para a escala atual do LogiTrack. Newman (2021) recomenda começar com bibliotecas de resiliência no nível do serviço e evoluir para service mesh somente quando o número de serviços torna a gestão ponto-a-ponto inviável.

---

## 4. Trade-offs da Decisão

| Dimensão | Ganho | Custo |
|---|---|---|
| **Disponibilidade** | Sistema continua operando (degradado) mesmo com Google Maps indisponível | Rotas em cache podem estar desatualizadas — risco de qualidade de dado |
| **Latência** | Circuit Breaker aberto responde imediatamente com fallback | Latência de fallback (Redis) substitui latência de Maps (~50ms vs ~200ms) — ganho |
| **Complexidade** | Falhas são contidas no microsserviço de origem | Cada serviço precisa configurar e monitorar seu próprio Circuit Breaker |
| **Observabilidade** | Estados do Circuit Breaker expostos via métricas Prometheus | Equipe precisa configurar alertas para transições de estado (ABERTO → SEMI-ABERTO) |
| **Custo Redis** | Cache de fallback usa Memorystore já existente | TTL de cache precisa ser gerenciado — rotas expiradas devem ser invalidadas em mudanças de tráfego |

---

## 5. Consequências

- O `docker-compose.yml` local inclui container Redis para simular o Memorystore GCP.
- Dashboards de observabilidade (Cloud Monitoring) devem incluir painel com estado do Circuit Breaker e taxa de uso de cada Bulkhead pool.
- Testes de contrato incluem cenários de falha: Google Maps retornando 503 deve acionar fallback em < 100ms.

---

## 6. Referências

- NYGARD, Michael T. *Release It! Design and Deploy Production-Ready Software*. 2ª ed. Pragmatic Bookshelf, 2018. Cap. 5.
- NEWMAN, Sam. *Building Microservices*. 2ª ed. O'Reilly, 2021. Cap. 12.
- RICHARDSON, Chris. *Microservices Patterns*. Manning, 2018. Cap. 3.
- MARTIN, Robert C. *Clean Architecture*. Prentice Hall, 2017.
