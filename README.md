# Laboratorio Kubernetes K3s - PostgreSQL + API1/API2 + NGINX


# Prerequisiti - Installazione K3s su Alpine Linux

Questa esercitazione presuppone che gli studenti dispongano già della VM Alpine Linux utilizzata negli esercizi precedenti.

L'obiettivo è installare K3s e utilizzare Kubernetes per eseguire l'applicazione PostgreSQL + API1 + API2 + NGINX.

---

# 1. Aggiornamento del sistema

Aggiornare il sistema operativo Alpine:

```bash
apk update
apk upgrade
```

---

# 2. Installazione prerequisiti

Installare i pacchetti necessari:

```bash
apk add bash curl iptables
```

---

# 3. Installazione K3s

Installare K3s utilizzando lo script ufficiale:

```bash
curl -sfL https://get.k3s.io | sh -
```

---

# 4. Verifica installazione

Verificare che il cluster Kubernetes sia attivo:

```bash
kubectl get nodes
```

Output atteso:

```text
NAME            STATUS   ROLES           AGE   VERSION
alpine-slim02   Ready    control-plane   ...   ...
```

Il nodo deve risultare nello stato:

```text
Ready
```

---


Questa directory contiene i manifest Kubernetes dell'esercizio derivato dal `docker-compose.yml` originale.

## Architettura

```text
Browser / curl
    ↓
NodePort nginx
    ↓
Service nginx
    ↓
Pod nginx
    ↓
Service api1 / Service api2
    ↓
Pod api1 / Pod api2
    ↓
Service postgres
    ↓
Pod postgres
    ↓
PVC postgres-pvc
```

## Immagini richieste nello store K3s/containerd

Verificare con:

```bash
k3s ctr -n k8s.io images ls | grep -E "postgres|api|nginx"
```

Devono comparire:

```text
docker.io/library/postgres:17
docker.io/library/postgres-api1:latest
docker.io/library/postgres-api2:latest
docker.io/library/nginx:latest
```

Se non compaiono, importarle da Docker:

```bash
docker save postgres:17 -o postgres17.tar
k3s ctr -n k8s.io images import postgres17.tar

docker save postgres-api1:latest -o postgres-api1.tar
k3s ctr -n k8s.io images import postgres-api1.tar

docker save postgres-api2:latest -o postgres-api2.tar
k3s ctr -n k8s.io images import postgres-api2.tar

docker save nginx:latest -o nginx.tar
k3s ctr -n k8s.io images import nginx.tar
```

## Applicazione manifest

Applicare in ordine:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f postgres/
kubectl apply -f api1/
kubectl apply -f api2/
kubectl apply -f nginx/
```

Oppure tutto insieme, dopo aver creato il namespace:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f postgres/ -f api1/ -f api2/ -f nginx/
```

## Verifica

```bash
kubectl get all -n lab
kubectl get pvc -n lab
kubectl get svc -n lab
kubectl get pods -n lab -o wide
```

## Test finale

Individuare il NodePort di nginx:

```bash
kubectl get svc -n lab nginx
```

Esempio:

```text
nginx   NodePort   10.43.x.x   80:31234/TCP
```

Test dalla VM:

```bash
curl http://localhost:31234
curl http://localhost:31234/add/mario
curl http://localhost:31234/lista
```

Da browser esterno:

```text
http://IP_VM:31234
http://IP_VM:31234/add/mario
http://IP_VM:31234/lista

```
# Self-healing Kubernetes

Uno dei vantaggi principali di Kubernetes è il self-healing.

Se un Pod viene eliminato o va in errore, Kubernetes lo ricrea automaticamente per mantenere lo stato desiderato definito nel Deployment.

## Test pratico

Aprire una finestra di monitoraggio:

```bash
kubectl get pods -n lab -w
```

Aprire una seconda console ed eliminare un Pod:

```bash
kubectl delete pod -n lab NOME_POD
```

Esempio:

```bash
kubectl delete pod -n lab nginx-6d8b9d55cb-lfzd6
```

Oppure:

```bash
kubectl delete pod -n lab api1-669848bb9c-j2t6n
```

Nella finestra di monitoraggio si vedrà il Pod passare in stato:

```text
Terminating
```

e subito dopo Kubernetes creerà automaticamente un nuovo Pod.

Output tipico:

```text
nginx-6d8b9d55cb-lfzd6   Terminating

nginx-7c9ff6854c-fz7ml  Pending

nginx-7c9ff6854c-fz7ml  Running
```

Il nuovo Pod avrà un nome diverso ma svolgerà la stessa funzione del precedente.

## Perché succede?

Il Deployment contiene:

```yaml
replicas: 1
```

Kubernetes controlla continuamente che esista una replica attiva.

Se il Pod viene eliminato:

```text
Pod eliminato
        ↓
Replica attive = 0
        ↓
Stato desiderato = 1
        ↓
Kubernetes crea un nuovo Pod
```

Questo meccanismo prende il nome di:

```text
Self-healing
```

ed è una delle principali differenze tra Kubernetes e Docker eseguito manualmente.

## Diagnostica

```bash
kubectl logs -n lab deployment/postgres
kubectl logs -n lab deployment/api1
kubectl logs -n lab deployment/api2
kubectl logs -n lab deployment/nginx
```

```bash
kubectl describe pod -n lab NOME_POD
kubectl describe svc -n lab nginx
kubectl get pods -n lab --show-labels
```

## Pulizia laboratorio

```bash
kubectl delete namespace lab
```

Attenzione: elimina anche il PVC e quindi i dati PostgreSQL del laboratorio.
