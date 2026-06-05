# Gold Plating — Runbook Operacional LogiTrack (Fase 3)

> Artefato extra para avaliação de bônus. Documenta procedimentos operacionais de produção para o ambiente GCP do LogiTrack.

---

## 1. Resposta a Incidentes

### 1.1 Circuit Breaker ABERTO — Serviço de Rotas

**Sintoma:** Alertas de `circuit_breaker_state=OPEN` no Cloud Monitoring. App mobile recebendo rotas em cache (mensagem "Rota estimada — dados podem estar desatualizados").

**Procedimento:**
```bash
# 1. Verificar status da Google Maps API
curl https://maps.googleapis.com/maps/api/directions/json?key=$MAPS_KEY&origin=-15.78&destination=-15.79

# 2. Verificar logs do Svc. Rotas
gcloud logs read "resource.type=cloud_run_revision AND resource.labels.service_name=svc-rotas" \
  --limit=50 --format=json | jq '.[] | .jsonPayload.message'

# 3. Se a API Maps estiver operacional, forçar reset do Circuit Breaker
curl -X POST https://api.logitrack.io/v1/admin/circuit-breaker/reset \
  -H "Authorization: Bearer $ADMIN_TOKEN"

# 4. Monitorar transição SEMI-ABERTO → FECHADO (aguardar ~30s)
```

---

### 1.2 Invalidação de Token Comprometido (JTI Blocklist)

**Sintoma:** Suspeita de token JWT vazado ou phishing confirmado.

**Procedimento:**
```bash
# 1. Extrair o JTI do token suspeito
echo $SUSPICIOUS_TOKEN | cut -d. -f2 | base64 -d | jq .jti

# 2. Adicionar à blocklist Redis no Memorystore
redis-cli -h $REDIS_HOST SET "blocklist:$JTI" "revoked" EX 86400

# 3. Verificar que o token está bloqueado
redis-cli -h $REDIS_HOST GET "blocklist:$JTI"
# Esperado: "revoked"

# 4. Registrar incidente no log de auditoria
gcloud logging write logitrack-security-audit \
  '{"severity":"WARNING","message":"JTI revogado","jti":"'$JTI'","reason":"phishing_suspeito"}'
```

---

### 1.3 Dead Letter Queue — Eventos não Processados

**Sintoma:** Alertas de mensagens acumuladas no tópico `logitrack-events-dlq`.

**Procedimento:**
```bash
# 1. Inspecionar mensagens na DLQ
gcloud pubsub subscriptions pull logitrack-dlq-sub --limit=5 --format=json

# 2. Identificar tipo de evento e consumidor com falha
# 3. Corrigir o consumidor e fazer redeploy
gcloud run deploy svc-notificacoes --image gcr.io/$PROJECT/svc-notificacoes:$VERSION

# 4. Replay das mensagens da DLQ após correção
# (processar manualmente ou via script de replay)
```

---

## 2. Diagrama de Fluxo — Confirmação de Entrega (Saga)

```
Motorista (App)
    │
    ▼
[POST /v1/entregas/{id}/confirmar]
    │
    ▼
Svc. Entregas ──────────────────────────────────────────────────────┐
    │  1. Atualiza status no DB Entregas (ENTREGUE)                  │
    │  2. Publica evento EntregaConfirmada no Pub/Sub                │
    └──────────────────────────────────────────────────────────────► Pub/Sub
                                                                      │
                              ┌───────────────────────────────────────┤
                              │                                       │
                              ▼                                       ▼
                    Svc. Notificações                      ERP Adapter
                    3. Envia SMS/push                  4. Sincroniza pedido
                       ao cliente                          como entregue
                              │
                              ▼
                    (idempotente: verifica
                     message_id já processado)
```

**Compensação em caso de falha:**
- Se Svc. Notificações falha 5x → mensagem vai para DLQ → alerta operacional → intervenção manual ou reprocessamento
- O status da entrega já foi gravado com sucesso — a falha é apenas na notificação, não no estado core

---

## 3. Política de Escalabilidade — Configuração GCP

### Cloud Run (serviços stateless)
```yaml
# Configuração recomendada para Svc. Rotas e Svc. Entregas
min-instances: 1        # Evita cold start em horário comercial
max-instances: 50       # Limite de custo
concurrency: 80         # Requisições simultâneas por instância
cpu: 2                  # vCPUs
memory: 512Mi
```

### GKE HPA (Svc. Rastreio)
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: svc-rastreio-hpa
spec:
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: websocket_connections_active
      target:
        type: AverageValue
        averageValue: 2500   # 2.500 conexões por pod
```

---

## 4. Checklist de Deploy em Produção

- [ ] Contract tests passando no CI (`npm run test:contract`)
- [ ] Imagem Docker tagueada com SemVer (`v1.2.3`)
- [ ] Variáveis de ambiente atualizadas no Secret Manager
- [ ] Migration de banco aplicada (se houver) com rollback testado
- [ ] Smoke test no ambiente de homologação
- [ ] Health check respondendo `200 OK` em `/health/ready`
- [ ] Alertas do Cloud Monitoring configurados para a nova versão
- [ ] Rollback documentado: `gcloud run deploy --image gcr.io/$PROJECT/svc-X:$VERSAO_ANTERIOR`

---

*Runbook mantido pela equipe de arquitetura LogiTrack. Última revisão: 2026-06-05.*
