## Fase 3 — Monitoring con Prometheus + Grafana

Concetto chiave prima di iniziare: in Kubernetes il monitoring non funziona come Zabbix dove configuri ogni host manualmente. Prometheus **scopre i target da solo** interrogando le API di k8s — si chiama **service discovery**. Grafana legge da Prometheus e mostra le dashboard.

Il tuo background con Zabbix ti aiuta molto qui — i concetti sono gli stessi, cambia solo il modo in cui i dati vengono raccolti.

---

### Step 1 — Installa Helm

Helm è il "package manager" di Kubernetes. Invece di applicare 20 YAML separati, installi tutto con un comando.

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verifica:

```bash
helm version
```

---

### Step 2 — Installa lo stack completo

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo update
```

Crea un namespace dedicato:

```bash
kubectl create namespace monitoring
```

Installa tutto lo stack:

```bash
helm install monitoring \
  prometheus-community/kube-prometheus-stack \
  -n monitoring
```

Questo installa in un colpo solo:
- **Prometheus** — raccoglie le metriche
- **Grafana** — dashboard
- **AlertManager** — gestisce gli alert
- **Node Exporter** — metriche CPU/RAM/disco del nodo
- **kube-state-metrics** — metriche degli oggetti Kubernetes (pod, deployment, ecc.)

Aspetta che siano tutti pronti:

```bash
kubectl get pods -n monitoring -w
```

Quando tutti sono `Running`, `Ctrl+C`.

---

### Step 3 — Accedi a Grafana

Port-forward sulla tua macchina:

```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80 --address 0.0.0.0
```

Apri il browser su:
```
http://10.5.0.83:3000
```

Credenziali default:
- **Username:** `admin`
- **Password:** `prom-operator`

---

### Step 4 — Esplora le dashboard preinstallate

Grafana viene con decine di dashboard già pronte. Vai su **Dashboards → Browse** e apri queste tre:

**"Kubernetes / Compute Resources / Cluster"** — uso CPU e RAM di tutto il cluster in tempo reale.

**"Kubernetes / Compute Resources / Namespace (Pods)"** — seleziona il namespace `lab` e vedi i tuoi pod nginx quanto consumano.

**"Node Exporter / Nodes"** — metriche del nodo fisico: CPU, RAM, disco, rete. Identico a quello che vedi in Zabbix, ma qui i dati vengono raccolti automaticamente.

Prenditi 5 minuti a esplorare — capire cosa mostrano queste dashboard è già una competenza spendibile al colloquio.

---

### Step 5 — Crea un alert reale

Questo è il pezzo più importante. Al colloquio ti chiederanno: *"Come ti accorgi se qualcosa va male dopo un deploy?"*

Creiamo un alert che scatta se un pod nel namespace `lab` è in crash da più di 1 minuto.

```bash
nano ~/lab-k8s/alertrule.yaml
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: lab-alerts
  namespace: lab
  labels:
    release: monitoring
spec:
  groups:
  - name: lab.rules
    rules:
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total{namespace="lab"}[5m]) * 60 > 0
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} sta crashando"
        description: "Il pod {{ $labels.pod }} nel namespace lab si è riavviato di recente"
```

```bash
kubectl apply -f alertrule.yaml
git add alertrule.yaml
git commit -m "add crash alert rule"
git push
```

Nota: lo hai pushato su Git — ArgoCD lo applicherà automaticamente. Stai già lavorando in GitOps.

---

### Step 6 — Triggera l'alert di proposito

Creiamo un pod che crasha subito per vedere l'alert in azione:

```bash
nano ~/lab-k8s/crashpod.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crasher
  namespace: lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crasher
  template:
    metadata:
      labels:
        app: crasher
    spec:
      containers:
      - name: crasher
        image: busybox
        command: ["sh", "-c", "exit 1"]
```

```bash
kubectl apply -f crashpod.yaml
```

Guarda cosa succede:

```bash
kubectl get pods -n lab -w
```

Vedrai il pod passare in `CrashLoopBackOff` — k8s continua a riavviarlo perché il container esce con errore. Questo è uno dei problemi più comuni in produzione.

Dopo 1-2 minuti vai su Grafana → **Alerting → Alert rules** — dovresti vedere l'alert `PodCrashLooping` passare in stato `Firing`.

Quando hai finito di osservare, pulisci:

```bash
kubectl delete -f crashpod.yaml
```

---

### Step 7 — Una query PromQL da sapere a memoria

Prometheus usa un linguaggio chiamato **PromQL**. Non devi saperlo scrivere fluentemente, ma devi capire le query base.

Vai su Grafana → **Explore**, seleziona Prometheus come datasource e prova questa:

```promql
rate(container_cpu_usage_seconds_total{namespace="lab"}[5m])
```

Questa query mostra il consumo CPU dei container nel namespace `lab` negli ultimi 5 minuti. Al colloquio puoi dire: *"Con Prometheus posso interrogare metriche storiche e in tempo reale — ho usato questa query per diagnosticare un spike di CPU dopo un deploy."*

---

### Verifica finale Fase 3

```bash
kubectl get pods -n monitoring
kubectl get prometheusrule -n lab
```

Dovresti avere tutti i pod monitoring in `Running` e la tua alert rule presente.
