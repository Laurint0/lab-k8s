## Fase 1 — Step 1: verifica che tutto sia pulito

Prima di installare qualsiasi cosa, capiamo cosa c'è già nel cluster.

```bash
kubectl get nodes
```

Dovresti vedere qualcosa tipo:
```
NAME     STATUS   ROLES                  AGE   VERSION
debian   Ready    control-plane,master   1h    v1.28.x
```

Poi:

```bash
kubectl get pods -A
```

Questo mostra **tutti i pod in tutti i namespace**. k3s installa già alcune cose di default — vedrai pod in `kube-system`. È normale. L'importante è che siano tutti `Running`.

**Perché lo facciamo:** in produzione, prima di toccare un cluster si verifica sempre lo stato. È un'abitudine che vale oro.

---

## Fase 1 — Step 2: capire i namespace

```bash
kubectl get namespaces
```

Vedrai qualcosa tipo:
```
NAME              STATUS
default           Active
kube-system       Active
kube-public       Active
```

**Cos'è un namespace:** è un isolamento logico. Immagina cartelle dentro il cluster. `kube-system` è roba interna di Kubernetes, non la tocchi mai. `default` è dove finisce tutto quello che crei senza specificare altro.

Noi creeremo un namespace dedicato per i nostri esperimenti:

```bash
kubectl create namespace lab
```

Verifica:
```bash
kubectl get namespaces
```

Ora `lab` è nella lista. Da qui in avanti, ogni cosa che facciamo va dentro `lab` — se rompiamo qualcosa, cancelliamo il namespace e ripartiamo da zero in 10 secondi.

---

## Fase 1 — Step 3: il tuo primo Deployment

Anziché copiare comandi, creiamo un file YAML. Questo è il modo in cui si lavora in produzione — i file vengono versionati su Git, non si digita tutto a mano.

```bash
mkdir ~/lab-k8s && cd ~/lab-k8s
```

Crea il file:

```bash
nano deployment.yaml
```

Incolla questo:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:alpine
        ports:
        - containerPort: 80
```

Salva (`Ctrl+X`, `Y`, `Enter`) e applicalo:

```bash
kubectl apply -f deployment.yaml
```

Guarda cosa succede in tempo reale:

```bash
kubectl get pods -n lab -w
```

Il flag `-w` è **watch** — il terminale si aggiorna da solo. Vedrai i pod passare da `Pending` → `ContainerCreating` → `Running`. Quando sono entrambi `Running`, premi `Ctrl+C`.

**Cosa hai appena fatto:** hai detto a Kubernetes "voglio sempre 2 copie di nginx in esecuzione". Se uno crasha, k8s lo riavvia da solo. Provalo:

```bash
# Prendi il nome di un pod
kubectl get pods -n lab

# Cancellalo (simula un crash)
kubectl delete pod <nome-del-pod> -n lab

# Guarda subito
kubectl get pods -n lab
```

Vedrai che k8s crea immediatamente un pod sostitutivo per tornare a 2 repliche. **Questo è il self-healing** di Kubernetes — uno dei motivi per cui le aziende lo usano.

---

## Fase 1 — Step 4: il Service

Il Deployment fa girare i pod, ma come ci si connette? I pod hanno IP temporanei che cambiano ogni volta che vengono ricreati. Il **Service** risolve questo problema — è un IP stabile che bilancia il traffico tra tutti i pod.

```bash
nano service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: lab
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f service.yaml
kubectl get svc -n lab
```

Vedrai qualcosa tipo:
```
NAME   TYPE        CLUSTER-IP      PORT(S)
web    ClusterIP   10.43.112.45    80/TCP
```

Il `CLUSTER-IP` è raggiungibile solo dall'interno del cluster. Verifica:

```bash
# Apri una shell temporanea dentro il cluster
kubectl run test --image=curlimages/curl -it --rm --restart=Never -- curl http://web.lab.svc.cluster.local
```

Questo avvia un pod temporaneo con curl, fa una richiesta al tuo Service usando il **DNS interno di Kubernetes** (`web.lab.svc.cluster.local`), e poi si cancella da solo. Dovresti vedere l'HTML di nginx.

**Il pattern `nome-service.namespace.svc.cluster.local` è come i microservizi si parlano tra loro** dentro un cluster. Tienilo a mente.

---

## Fase 1 — Step 5: l'Ingress

Adesso vogliamo raggiungere nginx dal browser della tua macchina host. Qui entra l'**Ingress** — un reverse proxy gestito da Kubernetes.

k3s include già **Traefik** come Ingress controller. Verificalo:

```bash
kubectl get pods -n kube-system | grep traefik
```

Ottimo. Ora trova l'IP della tua VM:

```bash
hostname -I
# oppure
ip a | grep inet
```

Supponiamo che il tuo IP sia `192.168.1.100`. Useremo `web.192.168.1.100.nip.io` come dominio — nip.io risolve automaticamente qualsiasi `*.IP.nip.io` all'IP corrispondente, senza configurare niente.

```bash
nano ingress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: lab
spec:
  rules:
  - host: web.192.168.1.100.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

**Sostituisci `192.168.1.100` con il tuo IP reale.**

```bash
kubectl apply -f ingress.yaml
kubectl get ingress -n lab
```

Apri il browser sulla tua macchina host e vai su `http://web.192.168.1.100.nip.io`. Dovresti vedere la pagina di nginx.

---

## Fase 1 — Step 6: TLS con cert-manager

Questo è il pezzo più vicino alla produzione. Installiamo cert-manager e aggiungiamo HTTPS.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

Aspetta che i pod siano pronti (ci vuole un minuto):

```bash
kubectl get pods -n cert-manager -w
```

Quando tutti e tre sono `Running`, premi `Ctrl+C`.

Ora crea un **ClusterIssuer** — è l'entità che firma i certificati. In locale usiamo `selfsigned`:

```bash
nano issuer.yaml
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
```

```bash
kubectl apply -f issuer.yaml
```

Aggiorna l'Ingress per abilitare TLS:

```bash
nano ingress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: lab
  annotations:
    cert-manager.io/cluster-issuer: selfsigned
spec:
  tls:
  - hosts:
    - web.192.168.1.100.nip.io
    secretName: web-tls
  rules:
  - host: web.192.168.1.100.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

```bash
kubectl apply -f ingress.yaml
```

Guarda cert-manager creare il certificato in automatico:

```bash
kubectl get certificate -n lab -w
```

Quando vedi `READY = True`, vai su `https://web.192.168.1.100.nip.io`. Il browser dirà "certificato non sicuro" perché è self-signed — accettalo. Sei su HTTPS.

**In produzione con un dominio reale**, cambi solo il ClusterIssuer da `selfsigned` a Let's Encrypt — tutto il resto del workflow è identico.

---

## Verifica finale della Fase 1

Esegui questi comandi e mandami l'output se qualcosa non torna:

```bash
kubectl get all -n lab
kubectl get ingress -n lab
kubectl get certificate -n lab
```

Dovresti avere:
- 2 pod `Running`
- 1 service `web`
- 1 ingress con host configurato
- 1 certificate `READY = True`
