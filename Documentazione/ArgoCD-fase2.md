## Fase 2 — ArgoCD (GitOps)

Prima di mettere le mani sul terminale, capisci **perché esiste ArgoCD**.

Senza GitOps il workflow è così:
```
Modifichi un YAML → kubectl apply a mano → speriamo bene
```

Con ArgoCD:
```
Modifichi un YAML su Git → ArgoCD se ne accorge → applica da solo → se qualcosa diverge, te lo dice
```

Il cluster diventa uno **specchio del tuo repo Git**. Questo è esattamente quello che intendono le aziende con "Infrastructure as Code" e "GitOps".

---

### Step 1 — Installa ArgoCD

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Aspetta che tutti i pod siano pronti — ci vuole circa 2 minuti:

```bash
kubectl get pods -n argocd -w
```

Quando tutti sono `Running`, premi `Ctrl+C`.

---

### Step 2 — Accedi alla UI

ArgoCD ha una UI web. Esponiamola in locale con port-forward:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address 0.0.0.0
```

Il flag `--address 0.0.0.0` serve perché sei su una VM — così la porta è raggiungibile anche dalla tua macchina host.

Apri il browser su:
```
https://10.5.0.83:8080
```

Ignora l'avviso certificato e vai avanti.

Recupera la password di admin:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Entra con:
- **Username:** `admin`
- **Password:** quella che hai appena ottenuto

---

### Step 3 — Prepara il repo Git

ArgoCD ha bisogno di un repo Git da monitorare. Usiamo **GitHub** — se non hai un account crealo, è gratis.

Crea un nuovo repo pubblico chiamato `k8s-lab` e poi fai questo sulla VM:

```bash
cd ~/lab-k8s

# Inizializza git
git init
git add .
git commit -m "first commit"

# Collega al tuo repo (sostituisci con il tuo username)
git remote add origin https://github.com/TUO-USERNAME/k8s-lab.git
git push -u origin main
```

Ora i tuoi YAML della Fase 1 sono su GitHub.

---

### Step 4 — Crea la prima Application in ArgoCD

Dalla UI di ArgoCD:

1. Clicca **"+ New App"**
2. Compila così:
   - **Application Name:** `lab-web`
   - **Project:** `default`
   - **Sync Policy:** `Automatic` ← questo è il cuore del GitOps
   - **Repository URL:** `https://github.com/TUO-USERNAME/k8s-lab`
   - **Path:** `.` (tutti i file nella root)
   - **Cluster URL:** `https://kubernetes.default.svc`
   - **Namespace:** `lab`
3. Clicca **Create**

ArgoCD legge il repo, trova i tuoi YAML e mostra lo stato del cluster.

---

### Step 5 — Il momento della verità

Ora fai una modifica al repo e guarda cosa succede.

Sulla VM modifica il deployment per portare le repliche da 2 a 3:

```bash
nano ~/lab-k8s/deployment.yaml
```

Cambia:
```yaml
replicas: 2
```
in:
```yaml
replicas: 3
```

Poi pusha:

```bash
git add deployment.yaml
git commit -m "scale web to 3 replicas"
git push
```

Torna nella UI di ArgoCD e guarda — entro pochi secondi ArgoCD rileva la modifica e crea il terzo pod automaticamente. Verifica anche da terminale:

```bash
kubectl get pods -n lab
```

Dovresti vedere 3 pod in `Running`.

**Questo è GitOps.** Git è la fonte di verità. Nessuno fa `kubectl apply` a mano in produzione — tutto passa dal repo.

---

### Step 6 — Prova il rollback

Torna a 2 repliche:

```bash
# Nella VM
sed -i 's/replicas: 3/replicas: 2/' deployment.yaml
git add deployment.yaml
git commit -m "scale back to 2 replicas"
git push
```

ArgoCD torna a 2 pod. Il log di Git è la storia completa di ogni cambiamento — sai chi ha fatto cosa e quando, e puoi tornare indietro con un `git revert`.
