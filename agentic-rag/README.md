# Agentic RAG Helm Chart - Guida Installazione

## Panoramica

Questo chart Helm distribuisce l'applicazione Agentic RAG Knowledge Graph su Kubernetes con:
- ✅ **PostgreSQL** con estensione pgvector
- ✅ **Neo4j** per il knowledge graph  
- ✅ **Applicazione principale** senza nginx
- ✅ **ConfigMap** per variabili ENV non sensibili
- ✅ **Secret** per API keys e credenziali
- ✅ **Persistent Volumes** per i dati
- ✅ **Compatibilità Traefik** (nessun ingress automatico)

## Struttura del Chart

```
agentic-rag/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml          # Applicazione principale
│   ├── service.yaml             # Service per l'app
│   ├── configmap.yaml           # Variabili ENV non sensibili
│   ├── secret.yaml              # API keys e credenziali
│   ├── pvc.yaml                 # Persistent Volume Claims per l'app
│   ├── postgresql-deployment.yaml
│   ├── postgresql-service.yaml
│   ├── postgresql-pvc.yaml
│   ├── neo4j-deployment.yaml
│   ├── neo4j-service.yaml
│   ├── neo4j-pvc.yaml
│   ├── serviceaccount.yaml
│   └── _helpers.tpl
└── README.md
```

## Prerequisiti

1. **Kubernetes cluster** attivo
2. **Helm 3.x** installato
3. **Traefik** configurato nel cluster (per l'ingress che gestirai tu)
4. **StorageClass** per i persistent volumes

## Installazione

### 1. Preparazione dei valori

Crea un file `my-values.yaml`:

```yaml
# my-values.yaml

# Configurazione dell'immagine
image:
  repository: your-registry/agentic-rag
  tag: "latest"
  pullPolicy: IfNotPresent

# Configurazione LLM (IMPORTANTE: inserire le API keys reali)
llm:
  provider: openai
  baseUrl: https://api.openai.com/v1
  apiKey: "sk-proj-your-actual-api-key-here"  # ⚠️ SOSTITUIRE
  choice: gpt-4o-mini
  
  embedding:
    provider: openai
    baseUrl: https://api.openai.com/v1
    apiKey: "sk-proj-your-actual-api-key-here"  # ⚠️ SOSTITUIRE
    model: text-embedding-3-small
  
  ingestion:
    choice: gpt-4o-mini

# Configurazione database
postgresql:
  enabled: true
  auth:
    database: agentic_rag
    username: postgres
    password: your-secure-password  # ⚠️ CAMBIARE
  primary:
    persistence:
      enabled: true
      size: 20Gi

# Configurazione Neo4j
neo4j:
  enabled: true
  auth:
    username: neo4j
    password: your-secure-neo4j-password  # ⚠️ CAMBIARE
  persistence:
    data:
      enabled: true
      size: 20Gi
    logs:
      enabled: true
      size: 5Gi

# Configurazione applicazione
app:
  env: production
  logLevel: info
  # Tuning delle performance
  chunkSize: 800
  chunkOverlap: 150
  maxChunkSize: 1500
  vectorDimension: 1536
  maxSearchResults: 10

# Persistent storage per l'applicazione
persistence:
  enabled: true
  data:
    size: 10Gi
  logs:
    size: 5Gi
  documents:
    size: 20Gi

# Risorse
resources:
  limits:
    cpu: 2000m
    memory: 4Gi
  requests:
    cpu: 1000m
    memory: 2Gi

# Auto-scaling (opzionale)
autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 3
  targetCPUUtilizationPercentage: 70
```

### 2. Installazione del Chart

```bash
# Aggiungi il repository (se necessario)
# helm repo add your-repo https://your-helm-repo.com

# Installa il chart
helm install agentic-rag ./agentic-rag -f my-values.yaml

# Oppure da un repository
# helm install agentic-rag your-repo/agentic-rag -f my-values.yaml
```

### 3. Verifica dell'installazione

```bash
# Controlla lo stato dei pod
kubectl get pods -l app.kubernetes.io/name=agentic-rag

# Controlla i servizi
kubectl get services -l app.kubernetes.io/name=agentic-rag

# Controlla i persistent volumes
kubectl get pvc -l app.kubernetes.io/name=agentic-rag

# Verifica i logs dell'applicazione
kubectl logs -l app.kubernetes.io/name=agentic-rag -c agentic-rag
```

### 4. Configurazione Traefik Ingress

Dato che gestisci tu l'ingress, ecco un esempio per Traefik:

```yaml
# traefik-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: agentic-rag-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  tls:
    - hosts:
        - agentic-rag.yourdomain.com
      secretName: agentic-rag-tls
  rules:
    - host: agentic-rag.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: agentic-rag  # Nome del service generato da Helm
                port:
                  number: 8058
```

Applica l'ingress:
```bash
kubectl apply -f traefik-ingress.yaml
```

## Configurazioni Avanzate

### Gestione Secrets Esterni

Se usi un gestore di secrets esterno (come External Secrets Operator):

```yaml
# In my-values.yaml
externalSecrets:
  enabled: true
  secretStore:
    name: vault-secret-store
    kind: SecretStore

# Disabilita la creazione di secrets interni
llm:
  apiKey: ""  # Sarà caricato dal secret esterno
```

### Backup e Ripristino

```bash
# Backup PostgreSQL
kubectl exec -it deployment/agentic-rag-postgresql -- pg_dump -U postgres agentic_rag > backup.sql

# Backup Neo4j
kubectl exec -it deployment/agentic-rag-neo4j -- neo4j-admin dump --database=neo4j --to=/data/backup.dump
```

### Monitoraggio

```yaml
# In my-values.yaml per abilitare metriche
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8058"
  prometheus.io/path: "/metrics"
```

## Troubleshooting

### Pod non si avvia

```bash
# Controlla gli eventi
kubectl describe pod <pod-name>

# Controlla i logs
kubectl logs <pod-name> -c <container-name>

# Controlla le configurazioni
kubectl get configmap agentic-rag-config -o yaml
kubectl get secret agentic-rag-secrets -o yaml
```

### Problemi di connessione database

```bash
# Test connessione PostgreSQL
kubectl exec -it deployment/agentic-rag-postgresql -- psql -U postgres -d agentic_rag -c "SELECT 1;"

# Test connessione Neo4j
kubectl exec -it deployment/agentic-rag-neo4j -- cypher-shell -u neo4j -p password "RETURN 1;"
```

### Aggiornamento del Chart

```bash
# Aggiorna con nuovi valori
helm upgrade agentic-rag ./agentic-rag -f my-values.yaml

# Rollback se necessario
helm rollback agentic-rag 1
```

## Disinstallazione

```bash
# Rimuovi l'installazione Helm
helm uninstall agentic-rag

# Rimuovi i PVC se necessario (⚠️ Perderai i dati)
kubectl delete pvc -l app.kubernetes.io/name=agentic-rag
```

## Note di Sicurezza

1. **Cambia sempre le password di default** in `my-values.yaml`
2. **Usa secret management tools** per le API keys in produzione
3. **Configura network policies** se necessario
4. **Monitora i logs** per attività sospette
5. **Backup regolari** dei database

## Supporto

Per problemi con il chart, controlla:
1. I logs dei pod con `kubectl logs`
2. Gli eventi con `kubectl get events`
3. La documentazione del progetto originale