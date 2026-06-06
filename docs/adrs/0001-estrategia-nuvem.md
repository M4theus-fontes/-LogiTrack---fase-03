# ADR 0001 — Estratégia de Nuvem e Escalabilidade

| Campo | Valor |
|---|---|
| **ID** | 0001 |
| **Título** | Adoção de GCP com Cloud Run (PaaS) e GKE (CaaS) para escalabilidade híbrida |
| **Status** | Aceita |
| **Data** | 2026-06-05 |
| **Projeto** | LogiTrack — Fase 3 |

---

## 1. Contexto — Conflito de Forças

O LogiTrack opera em um domínio de logística com **padrão de carga altamente variável**: picos previsíveis em horários comerciais (7h–10h e 13h–16h) e picos imprevisíveis em datas sazonais (Black Friday, Natal). O sistema legado N-Tier da Fase 1 foi identificado como incapaz de lidar com essa variabilidade sem superprovisionamento constante de infraestrutura.

As forças em conflito são:

- **Custo vs. Disponibilidade:** manter servidores sempre provisionados para o pico desperdiça recurso 70% do tempo; escalar sob demanda exige automação e abstrações de nuvem.
- **Latência vs. Resiliência:** o Serviço de Rastreio precisa processar posições GPS com latência < 200ms, mas também precisa sobreviver a falhas de zona de disponibilidade.
- **Complexidade operacional vs. Controle:** soluções IaaS (VMs puras) dão controle total mas exigem equipe de infra dedicada; PaaS reduz esse custo a preço de menor customização.
- **Stateless vs. Stateful:** a maioria dos microsserviços é stateless (pode escalar horizontalmente sem coordenação), mas o Serviço de Rastreio mantém conexões WebSocket abertas — é stateful e requer tratamento diferenciado.

---

## 2. Decisão

Adotar **Google Cloud Platform (GCP)** como provedor de nuvem, com a seguinte distribuição de modelos de serviço:

### 2.1 Cloud Run (PaaS/Serverless) — Serviços Stateless

Os microsserviços **Rotas, Frota, Entregas e Notificações** serão deployados no **Cloud Run**, que provê:

- Escalonamento automático de 0 a N instâncias baseado em requisições (scale-to-zero).
- Cobrança por 100ms de CPU/memória consumidos — custo zero em ociosidade.
- Deploy via imagem Docker sem gerenciamento de nós ou clusters.
- Integração nativa com Cloud IAM, Secret Manager e Cloud SQL.

**Justificativa teórica:** Segundo o Google Cloud Architecture Framework (2024), Cloud Run é a escolha recomendada para workloads HTTP stateless com padrão de tráfego variável, pois elimina o custo de gerenciamento de infraestrutura enquanto mantém portabilidade (padrão OCI/Docker). Newman (2021, cap. 8) reforça que microsserviços stateless são os candidatos naturais a deployments serverless por sua ausência de estado compartilhado entre instâncias.

### 2.2 GKE — Google Kubernetes Engine (CaaS) — Serviço Stateful

O **Serviço de Rastreio** (Go/WebSocket, alta frequência de escrita GPS) será deployado no **GKE** com:

- Pods gerenciados por Deployment com HPA (Horizontal Pod Autoscaler) baseado em métricas customizadas (conexões WebSocket ativas).
- Node pools dedicados com máquinas otimizadas para rede (n2-standard-4).
- Uso de Firestore como banco de dados NoSQL para suportar a alta taxa de escrita de coordenadas GPS.

**Justificativa teórica:** Burns et al. (2016) em *Borg, Omega, and Kubernetes* argumentam que workloads stateful com conexões de longa duração requerem orquestração com consciência de estado, algo que o modelo serverless puro não oferece sem workarounds significativos. O GKE permite controle granular de afinidade de pods e gerenciamento de sessões persistentes.

### 2.3 Estratégia de Escalabilidade

| Tipo | Mecanismo | Onde aplicado |
|------|-----------|---------------|
| **Horizontal** | Cloud Run auto-scale / GKE HPA | Todos os microsserviços |
| **Vertical** | Ajuste de CPU/memória por container | GKE node pools |
| **Geográfica** | Cloud Run multi-região (futuro) | Serviço de Rotas |

A preferência é por **escalabilidade horizontal**, alinhada ao princípio do fator XII (*twelve-factor app*) que recomenda processos stateless e replicáveis (Wiggins, 2012).

---

## 3. Alternativas Consideradas e Rejeitadas

### 3.1 IaaS puro (Compute Engine / VMs)

**Rejeitado porque:** exige gerenciamento manual de SO, patches de segurança, load balancers e grupos de instâncias. Para uma equipe pequena, o custo operacional supera o benefício de controle total. Pressman (2011) alerta que complexidade operacional desnecessária é fonte direta de dívida técnica.

### 3.2 AWS (ECS/Fargate + EKS)

**Rejeitado porque:** o LogiTrack depende fortemente da **Google Maps Platform** para otimização de rotas. Hospedar na GCP elimina latência de rede entre o Serviço de Rotas e a API do Google Maps (tráfego interno à infraestrutura Google), além de simplificar billing e IAM.

### 3.3 Serverless puro (Cloud Functions)

**Rejeitado porque:** Cloud Functions tem cold start de 200–800ms (Google Cloud, 2024), incompatível com o requisito de resposta < 200ms do Serviço de Rastreio. Além disso, não suporta WebSockets nativamente.

---

## 4. Trade-offs da Decisão

| Dimensão | Ganho | Custo |
|---|---|---|
| **Custo financeiro** | Scale-to-zero no Cloud Run elimina custo em ócio | GKE tem custo fixo de control plane (~$72/mês) mesmo sem carga |
| **Operacional** | Cloud Run elimina gerenciamento de nós | GKE exige conhecimento de Kubernetes para o time |
| **Latência** | Tráfego GCP→Google Maps é interno (< 5ms) | Cold start do Cloud Run pode adicionar 50–200ms na primeira requisição após ociosidade |
| **Portabilidade** | Containers Docker são portáveis para qualquer provedor | Uso de Pub/Sub e Firestore cria lock-in parcial com GCP |
| **Escalabilidade** | HPA no GKE responde a métricas customizadas | Configuração de HPA com métricas WebSocket requer instrumentação extra |

---

## 5. Consequências

- O `docker-compose.yml` do ambiente local emula Cloud Run com containers stateless e GKE com volumes nomeados para o serviço de rastreio.
- O CI/CD pipeline (GitHub Actions) faz deploy automático no Cloud Run via `gcloud run deploy` e no GKE via `kubectl apply`.
- A política de versionamento de imagem Docker segue SemVer: `logitrack/svc-rotas:1.2.3`.

---

## 6. Referências

- NEWMAN, Sam. *Building Microservices*. 2ª ed. O'Reilly, 2021. Cap. 8.
- PRESSMAN, Roger S. *Engenharia de Software*. 7ª ed. McGraw-Hill, 2011.
- WIGGINS, Adam. *The Twelve-Factor App*. 2012. Disponível em: https://12factor.net
- BURNS, Brendan et al. *Borg, Omega, and Kubernetes*. ACM Queue, 2016.
- Google Cloud Architecture Framework. Disponível em: https://cloud.google.com/architecture/framework
