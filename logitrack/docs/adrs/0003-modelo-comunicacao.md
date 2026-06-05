# ADR 0003 — Modelo de Comunicação entre Microsserviços

| Campo | Valor |
|---|---|
| **ID** | 0003 |
| **Título** | Comunicação híbrida: REST síncrono para queries + Pub/Sub assíncrono para eventos |
| **Status** | Aceita |
| **Data** | 2026-06-05 |
| **Projeto** | LogiTrack — Fase 3 |

---

## 1. Contexto — Conflito de Forças

O LogiTrack possui dois perfis de comunicação fundamentalmente distintos:

1. **Consultas orientadas a resultado imediato:** um motorista solicita sua rota otimizada e precisa da resposta antes de partir — a interação é síncrona por natureza.
2. **Eventos de mudança de estado:** uma entrega é confirmada, desencadeando notificação ao cliente, atualização de KPIs no dashboard e sincronização com o ERP — essas ações podem ocorrer em paralelo e não precisam bloquear o fluxo do motorista.

Escolher um único modelo de comunicação para ambos os perfis cria trade-offs intransponíveis:

- **Síncrono puro (REST/gRPC):** acoplamento temporal entre serviços — se o Serviço de Notificações estiver lento, o Serviço de Entregas fica bloqueado aguardando confirmação, degradando a experiência do motorista.
- **Assíncrono puro (eventos):** adequado para reatividade, mas inapropriado para consultas que precisam de resposta imediata — implementar request-reply sobre mensageria adiciona complexidade desnecessária e latência extra.

---

## 2. Decisão

Adotar **modelo de comunicação híbrido**, com critério de escolha baseado no tipo de interação:

### 2.1 REST Síncrono — Consultas e Comandos com Resposta Imediata

Utilizado quando: o chamador precisa da resposta para continuar seu fluxo.

**Casos de uso no LogiTrack:**
- App Mobile → API Gateway → Serviço de Rotas: `GET /v1/rotas/otimizada?origem=...`
- Dashboard → API Gateway → Serviço de Frota: `GET /v1/veiculos/{id}/status`
- Serviço de Entregas → ERP Externo: `POST /pedidos/{id}/confirmar`

**Protocolo:** HTTP/1.1 com JSON para comunicação externa; HTTP/2 entre serviços internos para multiplexação de requisições.

**Justificativa teórica:** Richardson (2018) classifica REST como adequado para interações do tipo *request/response* onde o cliente precisa do resultado imediatamente para prosseguir. Newman (2021, cap. 4) complementa: "use synchronous calls where you need to know the result of a request before you can proceed." O custo — acoplamento temporal — é aceitável nesses casos porque a alternativa (request-reply assíncrono) seria mais complexa sem benefício real.

### 2.2 Google Cloud Pub/Sub — Eventos de Domínio (Assíncrono)

Utilizado quando: uma mudança de estado em um serviço precisa ser propagada para múltiplos consumidores sem acoplamento direto.

**Eventos publicados e consumidores:**

| Evento | Publicador | Consumidores |
|--------|------------|--------------|
| `EntregaConfirmada` | Svc. Entregas | Svc. Notificações, ERP (via adaptador) |
| `EntregaAtrasada` | Svc. Entregas | Svc. Notificações, Dashboard (via SSE) |
| `PosicaoAtualizada` | Svc. Rastreio | Svc. Rotas (reotimização), Dashboard |
| `VeiculoAlocado` | Svc. Frota | Svc. Entregas, Svc. Rotas |

**Configuração do Pub/Sub:**
- Modelo: **tópico por tipo de evento** (não por par produtor-consumidor)
- Entrega: **at-least-once** com deduplicação por `message_id` no consumidor
- Dead Letter Topic: eventos que falham 5 vezes são movidos para `topic-dlq` e geram alerta no Cloud Monitoring
- Retenção de mensagens: 7 dias

**Justificativa teórica:** Hohpe e Woolf (2003) em *Enterprise Integration Patterns* definem o padrão **Publish-Subscribe Channel** como a solução para desacoplar produtores de consumidores quando múltiplos serviços precisam reagir ao mesmo evento. Newman (2021) reforça que eventos de domínio comunicados via mensageria eliminam o acoplamento temporal — o Serviço de Entregas não precisa conhecer nem aguardar o Serviço de Notificações para confirmar uma entrega. Isso alinha-se ao princípio de baixo acoplamento (MARTIN, 2017) que permeia a arquitetura hexagonal adotada na Fase 2.

### 2.3 WebSocket — Streaming de Posição GPS

O Serviço de Rastreio utiliza **WebSocket bidirecional** para receber posições GPS dos apps mobile dos motoristas em alta frequência (1 atualização/3s por veículo):

- Conexão persistente: elimina overhead de handshake HTTP por atualização.
- Protocolo binário compactado (MessagePack) para minimizar tráfego de rede.
- Posições recebidas são gravadas no Firestore e publicadas no tópico `PosicaoAtualizada` do Pub/Sub.

---

## 3. Alternativas Consideradas e Rejeitadas

### 3.1 gRPC para toda comunicação interna

**Rejeitado porque:** embora gRPC ofereça melhor performance que REST/JSON (Protobuf binário, HTTP/2 nativo), o custo de manutenção de arquivos `.proto` compartilhados entre serviços em linguagens diferentes (Python, Java, Go, Node.js) cria acoplamento de esquema — qualquer mudança no proto exige recompilação e deploy coordenado de todos os consumidores. Newman (2021) chama isso de *stamp coupling*. Para o volume de tráfego atual do LogiTrack, o ganho de performance não justifica esse custo. gRPC é mantido apenas na comunicação interna do Serviço de Rastreio → Firestore.

### 3.2 Apache Kafka em vez de Google Cloud Pub/Sub

**Rejeitado porque:** Kafka exige cluster dedicado (mínimo 3 brokers para produção), gestão de partições, offsets e retenção — overhead operacional significativo para uma equipe individual. O Cloud Pub/Sub é um serviço gerenciado sem cluster para operar, com garantias de entrega equivalentes para o volume do LogiTrack. Newman (2021) recomenda soluções gerenciadas quando o diferencial de Kafka (replay de eventos, ordenação por partição) não é requisito do sistema.

### 3.3 Assíncrono puro com RabbitMQ

**Rejeitado porque:** implementar o padrão request-reply sobre RabbitMQ para suportar consultas síncronas (ex: cálculo de rota) adiciona complexidade sem benefício — correlation IDs, filas de resposta temporárias e timeouts precisariam ser implementados manualmente. REST é a solução idiomática e mais simples para esse padrão.

---

## 4. Trade-offs da Decisão

| Dimensão | Ganho | Custo |
|---|---|---|
| **Desacoplamento** | Produtores de eventos não conhecem consumidores — adicionar novo consumidor não requer alteração no publicador | Depuração é mais difícil: rastrear uma falha requer correlacionar logs entre serviços desacoplados |
| **Resiliência** | Consumidores podem estar offline — mensagens ficam no Pub/Sub até 7 dias | Consistência eventual: notificação ao cliente pode chegar segundos após a confirmação da entrega |
| **Latência (REST)** | Resposta imediata para o chamador | Acoplamento temporal — se Svc. Rotas estiver lento, o app mobile espera |
| **Latência (Pub/Sub)** | Publicador não espera consumidores | Latência adicional de 50–500ms entre evento e processamento pelo consumidor |
| **Complexidade** | Cada serviço tem responsabilidade clara (publicar OU consumir, não orquestrar) | Equipe precisa entender dois modelos de comunicação e suas ferramentas de observabilidade distintas |
| **Ordering** | Pub/Sub entrega at-least-once — idempotência deve ser implementada no consumidor | Eventos do mesmo pedido podem chegar fora de ordem — consumidores precisam de lógica de deduplicação |

---

## 5. Consequências

- O Serviço de Notificações deve ser **idempotente**: receber `EntregaConfirmada` duas vezes (at-least-once) não deve gerar notificação duplicada ao cliente — implementar cache de `message_id` processados por 24h.
- O diagrama de sequência completo de cada fluxo (ex: confirmação de entrega) deve documentar explicitamente quais etapas são síncronas e quais são assíncronas — disponível em `docs/diagrams/`.
- O `docker-compose.yml` local substitui Pub/Sub por **emulador local** (`google/cloud-sdk` com `gcloud beta emulators pubsub`).

---

## 6. Referências

- NEWMAN, Sam. *Building Microservices*. 2ª ed. O'Reilly, 2021. Cap. 4.
- RICHARDSON, Chris. *Microservices Patterns*. Manning, 2018. Cap. 3.
- HOHPE, Gregor; WOOLF, Bobby. *Enterprise Integration Patterns*. Addison-Wesley, 2003.
- MARTIN, Robert C. *Clean Architecture*. Prentice Hall, 2017.
- Google Cloud Pub/Sub Documentation. Disponível em: https://cloud.google.com/pubsub/docs
