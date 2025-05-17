# Telegram ReBot Helm Chart

Helm chart per il deploy di **telegram-rebot**, un bot intermedio basato su [Telethon](https://github.com/LonamiWebs/Telethon) che interagisce con un bot pubblico Telegram simulando un utente reale. Questo chart gestisce l'intero ciclo di deploy, inclusi login via OTP, sessioni persistenti e comunicazione tramite webhook.

## Caratteristiche

- Init container per il login Telegram con codice OTP
- Container principale per inoltro messaggi
- Volumi persistenti per sessione e chiavi
- Supporto per ingress e TLS (opzionale)
- Service dedicato per il login interattivo
- CronJob opzionale per logrotate

---

## Installazione

### 1. Aggiungi il tuo repository (opzionale)
```bash
helm repo add telegram-rebot https://tuo-repo.github.io/helm-charts
helm repo update
```

### 2. Installa il chart
```bash
helm install telegram-rebot ./telegram-rebot \
  --set api_id=<ID> \
  --set api_hash=<HASH> \
  --set phone_number="<+39...>" \
  --set bot_username=<bot_username> \
  --set master_api_key=<api_key>
```

> ⚠️ Dopo l'installazione, accedi a `/init` del `telegram-login-service` per completare il login con codice OTP. Il container si chiuderà automaticamente dopo un login riuscito.

---

## Valori Configurabili

Ecco un estratto dei valori principali presenti in `values.yaml`:

```yaml
api_id: 123456
api_hash: your_api_hash
bot_username: vcnngr_bot
master_api_key: admin-super-key

image:
  repository: vcnngr/telegram-rebot
  tag: latest
  pullPolicy: IfNotPresent

persistence:
  keys:
    enabled: true
    size: 1Gi
    accessModes:
      - ReadWriteOnce
  logs:
    enabled: true
    size: 1Gi
  session:
    enabled: true
    size: 1Gi

ingress:
  enabled: true
  host: bot.tuodominio.com
  tls: true
  tlsSecretName: tls-secret
```

Per una lista completa, consulta `values.yaml`.

---

## Ingress
Se abilitato, il servizio espone un endpoint HTTPS:

```yaml
ingress:
  enabled: true
  host: bot.tuodominio.com
  tls: true
  tlsSecretName: tls-secret
```

---

## Accesso al servizio di login
Per completare la procedura di login Telegram:

1. Installa il chart
2. Effettua port-forward:
   ```bash
   kubectl port-forward svc/telegram-rebot-login 8080:8080
   ```
3. Visita [http://localhost:8080/init](http://localhost:8080/init) e inserisci il codice OTP

---

## Componenti inclusi

- `deployment.yaml` – deploy di init e main container con volumi condivisi
- `telegram-login-service.yaml` – service per esporre l'interfaccia login
- `keys-pvc.yaml`, `log-pvc.yaml`, `pvc.yaml` – volumi persistenti
- `secret.yaml` – gestione credenziali Telegram
- `cronjob-logrotate.yaml` – rotazione log (opzionale)

---

## Sviluppo e Debug

- **Sessione non necessaria?** Il container di login termina automaticamente se trova una sessione valida
- **CrashLoopBackOff del login?** Assicurati che il codice OTP sia valido
- **Vuoi solo rigenerare la sessione?** Disinstalla e reinstalla solo il login con:

```bash
helm upgrade telegram-rebot ./telegram-rebot --reuse-values --set force_login=true
```

---

## License
MIT
