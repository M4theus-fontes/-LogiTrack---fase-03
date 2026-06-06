# SAD — Software Architecture Document
## LogiTrack · Fase 3 — Cloud & Microsserviços

| Campo | Valor |
|---|---|
| **Versão** | 3.0 |
| **Data** | 2026-06-05 |
| **Projeto** | LogiTrack — Sistema de Otimização de Rotas e Entregas |
| **Disciplina** | Arquitetura de Software 2026.1 |
| **Professor** | Carlos Roberto Gomes Júnior |
| **Instituição** | UniEVANGÉLICA |

---

## 1. Introdução

### 1.1 Propósito

Este documento descreve a arquitetura de software do LogiTrack na Fase 3 do Mini Projeto Arquiteto Decisor. Registra as decisões estruturais, os componentes do sistema, as interações entre eles e os trade-offs que fundamentaram cada escolha arquitetural.

### 1.2 Escopo

O SAD cobre a evolução do sistema desde a arquitetura N-Tier refatorada na Fase 1 e a documentação ADR hexagonal da Fase 2, até a arquitetura de microsserviços em nuvem (GCP) da Fase 3.

### 1.3 Evolução Arquitetural

| Fase | Modelo | Principal Mudança |
|------|--------|-------------------|
| Fase 1 | N-Tier → Hexagonal | Desacoplamento domínio/infraestrutura; eliminação de dependência direta do banco no domínio |
| Fase 2 | Hexagonal com ADR | Documentação formal de decisões; modelo C4 nível 2; análise de trade-offs |
| Fase 3 | Microsserviços em GCP | Decomposição por domínio; containerização; comunicação assíncrona; cloud-native |

---

## 2. Contexto do Sistema

### 2.1 Problema de Negócio

Empresas de logística enfrentam três desafios críticos que o LogiTrack resolve:

- **Ineficiência de rota:** sem otimização em tempo real, motoristas percorrem caminhos subótimos, aumentando custo de combustível e tempo de entrega.
- **Falta de visibilidade:** clientes e gestores não sabem onde está o veículo nem quando a entrega chegará.
- **Fragilidade sistêmica:** sistemas legados monolíticos não escalam em picos (Black Friday, Natal) e não toleram falhas de APIs externas.

### 2.2 Stakeholders

| Stakeholder | Interesse Arquitetural |
|---|---|
| Motorista | App mobile confiável, rota sempre disponível mesmo offline |
| Gestor de Frota | Dashboard em tempo real, alertas de atraso |
| Cliente Final | Rastreio de entrega e previsão de chegada |
| Equipe de Dev | Deployabilidade independente por serviço |
| Operações | Observabilidade, alertas e recuperação automática de falhas |

---

## 3. Visão de Containers (C4 Nível 2)

O diagrama completo está no `README.md`. A tabela abaixo complementa com responsabilidades e tecnologias:

| Container | Tecnologia | Responsabilidade |
|---|---|---|
| API Gateway | Cloud Endpoints / Kong | Autenticação JWT, rate limiting, roteamento por versão, timeout global |
| Svc. Rotas | Python / OR-Tools / Cloud Run | Cálculo e otimização de rotas; integração Google Maps; cache em Redis |
| Svc. Frota | Node.js / Cloud Run | CRUD de veículos, motoristas e alocação de carga |
| Svc. Entregas | Java / Spring Boot / Cloud Run | Ciclo de vida das entregas; publicação de eventos de domínio |
| Svc. Rastreio | Go / GKE | Ingestão de posições GPS via WebSocket; alta frequência de escrita |
| Svc. Notificações | Node.js / Cloud Run | Consumo de eventos; envio de SMS/e-mail/push |
| Dashboard Web | React / Cloud Run | Interface do gestor; SPA servida via CDN |
| App Mobile | Flutter | Interface do motorista; recebe rota, confirma entregas, envia GPS |
| DB Rotas | Cloud SQL (PostgreSQL) | Histórico de rotas e métricas |
| DB Frota | Cloud SQL (PostgreSQL) | Cadastro de veículos e motoristas |
| DB Entregas | Cloud SQL (PostgreSQL) | Pedidos, status, histórico |
| DB Rastreio | Firestore | Posições GPS em tempo real (NoSQL, alta escrita) |
| Pub/Sub | Google Cloud Pub/Sub | Barramento de eventos assíncronos |
| Redis | Memorystore (Redis) | Cache de rotas + blocklist JTI |
| Keycloak | GKE | Authorization Server; emissão de JWT; Client Credentials M2M |

---

## 4. Decisões Arquiteturais Chave

### 4.1 Database per Service

Cada microsserviço possui e controla seu próprio banco de dados, sem compartilhamento de esquema. Essa decisão elimina acoplamento de dados estrutural e garante que uma migration em um serviço não afete os demais.

**Consequência:** operações cross-service (ex: registrar entrega E notificar lojista) não podem usar transações ACID únicas. O padrão **Saga coreografada** via Pub/Sub resolve esse gap: cada serviço publica um evento de conclusão que o próximo consome, com **compensating transactions** em caso de falha.

### 4.2 Autenticação e Autorização

- **Usuários externos (motorista, gestor, cliente):** JWT emitido pelo Keycloak com `exp = 15 min` + Refresh Token em HttpOnly cookie.
- **Comunicação M2M interna:** OAuth2 Client Credentials — cada microsserviço possui `client_id` e `client_secret` próprios no Keycloak.
- **Verificação de autorização por objeto (BOLA):** cada microsserviço extrai o claim `sub` do JWT e verifica se o recurso solicitado pertence ao sujeito — implementado na camada de Application Service, não no Gateway.

### 4.3 Estratégia de Versionamento de API

APIs públicas seguem versionamento semântico na URL (`/v1/`, `/v2/`). Versões são mantidas por mínimo de 6 meses após deprecação, com `Sunset Header` nas respostas da versão antiga. O API Gateway gerencia o roteamento entre versões.

---

## 5. Visão de Implantação (GCP)

```
┌─────────────────────────────────────────────────────────┐
│                    Google Cloud Platform                │
│                                                         │
│  ┌──────────────┐    ┌─────────────────────────────┐   │
│  │  Cloud CDN   │    │        Cloud Run             │   │
│  │  (Frontend)  │    │  svc-rotas   svc-frota       │   │
│  └──────────────┘    │  svc-entregas svc-notif      │   │
│                      └─────────────────────────────┘   │
│  ┌──────────────┐    ┌─────────────────────────────┐   │
│  │   Cloud      │    │    GKE Cluster               │   │
│  │  Endpoints   │    │  svc-rastreio  keycloak      │   │
│  │ (API Gateway)│    └─────────────────────────────┘   │
│  └──────────────┘                                       │
│  ┌──────────────┐    ┌──────────────┐  ┌────────────┐  │
│  │  Cloud SQL   │    │  Firestore   │  │  Pub/Sub   │  │
│  │ (3 instâncias│    │  (GPS data)  │  │  (eventos) │  │
│  │  isoladas)   │    └──────────────┘  └────────────┘  │
│  └──────────────┘                                       │
│  ┌──────────────┐    ┌──────────────┐                  │
│  │ Memorystore  │    │   Secret     │                  │
│  │   (Redis)    │    │   Manager    │                  │
│  └──────────────┘    └──────────────┘                  │
└─────────────────────────────────────────────────────────┘
```

---

## 6. Atributos de Qualidade e Como São Endereçados

| Atributo | Requisito | Mecanismo Arquitetural |
|---|---|---|
| **Performance** | Resposta de rota < 2s | Cache Redis de rotas recentes; OR-Tools otimizado; Cloud Run scale-out |
| **Escalabilidade** | Suportar 10x carga em picos | Cloud Run scale-to-zero/N; GKE HPA para rastreio |
| **Resiliência** | Disponível mesmo com Google Maps fora | Circuit Breaker com fallback Redis; Pub/Sub retém eventos offline |
| **Confiabilidade** | Entrega de notificações garantida | Pub/Sub at-least-once + idempotência no consumidor + DLQ |
| **Segurança** | Acesso isolado por lojista/motorista | JWT 15 min + BOLA check + Bulkhead + rate limiting no Gateway |
| **Manutenibilidade** | Deploy independente por serviço | Database per service + CI/CD por repositório |
| **Observabilidade** | Diagnóstico de falhas em produção | Cloud Monitoring + Cloud Logging + traces distribuídos (Cloud Trace) |

---

## 7. Riscos Arquiteturais e Mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Indisponibilidade da Google Maps API | Média | Alto | Circuit Breaker + fallback de rota em cache |
| Cold start do Cloud Run em pico repentino | Alta | Médio | Configurar instância mínima = 1 para serviços críticos (Rotas, Entregas) |
| Lock-in com GCP (Pub/Sub, Firestore) | Baixa | Alto | Adapters abstraem detalhes de infra (hexagonal da Fase 2); migração possível |
| Saga inconsistente (evento perdido) | Baixa | Alto | Dead Letter Queue + alertas + compensating transactions documentadas |
| Proliferação de versões de API | Média | Médio | Política de deprecação 6 meses + contract tests no CI/CD |

---

## 8. Referências

- MARTIN, Robert C. *Clean Architecture*. Prentice Hall, 2017.
- NEWMAN, Sam. *Building Microservices*. 2ª ed. O'Reilly, 2021.
- PRESSMAN, Roger S. *Engenharia de Software*. 7ª ed. McGraw-Hill, 2011.
- NYGARD, Michael T. *Release It!*. 2ª ed. Pragmatic Bookshelf, 2018.
- RICHARDSON, Chris. *Microservices Patterns*. Manning, 2018.
- HOHPE, Gregor; WOOLF, Bobby. *Enterprise Integration Patterns*. Addison-Wesley, 2003.
- Google Cloud Architecture Framework. https://cloud.google.com/architecture/framework
