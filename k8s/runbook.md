# Operativni runbook — Secure Event Ticketing Platform

Ovaj dokument sadrži upute za rješavanje incidenata i rutinske operacije na produkcijskom
Kubernetes klasteru. Namijenjen je dežurnom inženjeru (on-call) i članovima tima koji
održavaju servis.

> Sve naredbe pretpostavljaju da je `kubectl` konfiguriran i spojen na ciljani klaster.
> Sve resurse aplikacije nalaze se u `namespace`-u `ticketing`.

---

## Brza referenca

```bash
# Postavi namespace kao default da skratiš naredbe
NS=ticketing
kubectl config set-context --current --namespace=$NS

# Pregled svih podova
kubectl get pods -n $NS

# Posljednji događaji u namespaceu (sortirano)
kubectl get events -n $NS --sort-by='.lastTimestamp'

# Logovi cijelog deploymenta
kubectl logs -n $NS deploy/<ime-deploymenta>

# Detaljan opis problematičnog poda
kubectl describe pod -n $NS -l app=<ime-aplikacije>

# Provjera ulaza i servisa
kubectl get ingress,svc -n $NS
```

### Imena deploymenta i labela

| Servis     | Deployment | Labela `app=` | Service port |
|------------|------------|---------------|--------------|
| API        | `api`      | `api`         | 8080         |
| Worker     | `worker`   | `worker`      | —            |
| Frontend   | `frontend` | `frontend`    | 3000         |
| PostgreSQL | `postgres` | `postgres`    | 5432         |
| Redis      | `redis`    | `redis`       | 6379         |

---

## Scenarij 1 — Baza podataka u CrashLoopBackOff statusu

**Simptomi**

- Pristup `/tickets/orders` vraća HTTP 503
- API readiness sonda javlja `not-ready`
- `postgres` pod nije u stanju `Running` (vidljivo kroz `kubectl get pods`)

**Dijagnostika**

```bash
kubectl describe pod -n ticketing -l app=postgres
kubectl logs -n ticketing -l app=postgres --previous
```

**Mogući uzroci i rješenja**

| Uzrok | Rješenje |
|-------|----------|
| Pogrešna lozinka u `Secret`-u | `kubectl edit secret ticketing-secret -n ticketing` → ažuriraj `POSTGRES_PASSWORD` (base64), zatim `kubectl rollout restart deploy/postgres -n ticketing` |
| PVC izgubljen / storage class nedostupan | `kubectl describe pvc postgres-data -n ticketing`, ponovo kreiraj PVC i obnovi iz backup-a |
| `OOMKilled` | Povećaj `resources.limits.memory` u `k8s/postgres.yaml`, zatim `kubectl apply -f k8s/postgres.yaml` |
| Greška u inicijalizacijskom SQL skripti pri prvom bootu | Pregledaj logove za SQL greške, ispravi `postgres-initdb` ConfigMap, obriši pod kako bi se init ponovo pokrenuo |
| Pod ne može mountati volume (PV pending) | Provjeri ima li klaster aktivnu `StorageClass`: `kubectl get storageclass`. Na k3s je `local-path`, na minikube `standard` |

**Provjera oporavka**

```bash
kubectl rollout status deploy/postgres -n ticketing
kubectl exec -n ticketing deploy/postgres -- pg_isready -U ticketing_user -d ticketing
```

---

## Scenarij 2 — Loš image tag u deploymentu

**Simptomi**

- API ili frontend podovi u stanju `ImagePullBackOff` ili `ErrImagePull`
- Rollout zapinje na novoj reviziji

**Dijagnostika**

```bash
kubectl describe pod -n ticketing -l app=api
# u Events sekciji traži poruku "Failed to pull image"
```

**Brzo rješenje — rollback na prethodnu reviziju**

```bash
# Pregled povijesti rollouta
kubectl rollout history deploy/api -n ticketing

# Rollback za jednu reviziju unatrag
kubectl rollout undo deploy/api -n ticketing

# Rollback na specifičnu reviziju
kubectl rollout undo deploy/api -n ticketing --to-revision=2

# Provjera da je rollback uspio
kubectl rollout status deploy/api -n ticketing
```

**Trajno rješenje**

1. Identificiraj koji se tag pokušava povući: `kubectl get deploy/api -n ticketing -o jsonpath='{.spec.template.spec.containers[0].image}'`
2. Provjeri postoji li tag na GHCR-u (`ghcr.io/krebor/devops-project-app-*`)
3. Ako tag ne postoji, ispravi `image:` polje u manifestu i ponovo primijeni

---

## Scenarij 3 — Pogrešan ili nedostajući Secret

**Simptomi**

- API podovi se podignu pa odmah pucaju
- U logovima poruka `password authentication failed for user "ticketing_user"`

**Dijagnostika**

```bash
kubectl logs -n ticketing -l app=api --tail=50
kubectl logs -n ticketing -l app=worker --tail=50
```

**Rješenje**

```bash
# 1. Generiraj base64 vrijednost za točnu lozinku
echo -n "ispravna-lozinka" | base64

# 2. Ažuriraj Secret (zamijeni <BASE64> rezultatom iz prvog koraka)
kubectl patch secret ticketing-secret -n ticketing \
  --type='json' \
  -p='[{"op":"replace","path":"/data/POSTGRES_PASSWORD","value":"<BASE64>"}]'

# 3. Restartaj deploymente koji koriste taj secret
kubectl rollout restart deploy/api deploy/worker -n ticketing

# 4. Provjeri da je readiness sonda zelena
kubectl rollout status deploy/api -n ticketing
```

**Važno:** PostgreSQL pod inicijalno postavlja lozinku samo kod prvog boota PVC-a.
Ako mijenjaš lozinku nakon prvog deploya, moraš se ili spojiti u bazu i ručno
promijeniti lozinku (`ALTER USER`), ili obrisati PVC i pustiti da se baza inicijalizira
ispočetka — što briše sve podatke!

---

## Scenarij 4 — Worker je prestao obrađivati narudžbe

**Simptomi**

- Narudžbe ostaju u Redis listi i ne pojavljuju se u tablici `ticket_orders`
- `GET /tickets/orders` ne vraća nove narudžbe nakon `POST /tickets/purchase`

**Dijagnostika**

```bash
# Logovi workera
kubectl logs -n ticketing -l app=worker --tail=100

# Duljina Redis queue-a (broj neobrađenih narudžbi)
kubectl exec -n ticketing deploy/redis -- redis-cli llen ticket_orders

# Status workera
kubectl get pods -n ticketing -l app=worker
```

**Rješenja**

```bash
# 1. Restart workera (najčešće rješava tranzijentne probleme)
kubectl rollout restart deploy/worker -n ticketing

# 2. Ako queue ima previše stavki, scaleupdaj broj workera
kubectl scale deploy/worker -n ticketing --replicas=3

# 3. Vraćanje na 1 replicu kad se queue isprazni
kubectl scale deploy/worker -n ticketing --replicas=1
```

Ako worker stalno puca, provjeri:
- Je li Postgres dostupan iz workera? `kubectl exec -n ticketing deploy/worker -- nc -zv postgres 5432`
- Je li Redis dostupan? `kubectl exec -n ticketing deploy/worker -- nc -zv redis 6379`
- Postoje li NetworkPolicy pravila koja blokiraju komunikaciju? (`kubectl get networkpolicy -n ticketing`)

---

## Scenarij 5 — Frontend dobiva 502/503 kroz Ingress

**Simptomi**

- Otvaranjem `http://ticketing.local/` u pregledniku javlja se Bad Gateway ili Service Unavailable

**Dijagnostika**

```bash
# Da li Ingress controller uopće radi?
kubectl get pods -n ingress-nginx

# Da li Ingress objekt ima IP / address?
kubectl get ingress -n ticketing

# Logovi Ingress controllera (filtrirano za ticketing.local)
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=200 | grep ticketing.local

# Ima li frontend ready endpoint-a?
kubectl get endpoints -n ticketing frontend
```

**Najčešći uzroci**

| Uzrok | Rješenje |
|-------|----------|
| Frontend pod nije ready | Provjeri sondu: `kubectl describe pod -l app=frontend -n ticketing` |
| `endpoints` prazan | Selector `app=frontend` ne pogađa pod — provjeri labele |
| `ingressClassName` ne odgovara controlleru | Pregledaj naziv: `kubectl get ingressclass`, prilagodi `k8s/ingress.yaml` |
| `/etc/hosts` ne pokazuje na klaster | Dodaj `127.0.0.1 ticketing.local` (ili pravi IP klastera) |

---

## Standardne operacije

### Rolling update — deploy nove verzije slike

```bash
# Opcija A: ažuriranjem manifesta
# 1. promijeni image tag u k8s/api.yaml
# 2. primijeni promjenu
kubectl apply -f k8s/api.yaml
kubectl rollout status deploy/api -n ticketing

# Opcija B: imperativno (bez ažuriranja manifesta)
kubectl set image deploy/api api=ghcr.io/krebor/devops-project-app-api:v2 -n ticketing
kubectl rollout status deploy/api -n ticketing

# Ako rollout zapne, prekini ga
kubectl rollout pause deploy/api -n ticketing

# Ili rollback ako rezultat nije ok
kubectl rollout undo deploy/api -n ticketing
```

### Skaliranje aplikacije

```bash
# Ručno skaliranje broja replica
kubectl scale deploy/api -n ticketing --replicas=4
kubectl scale deploy/frontend -n ticketing --replicas=4

# Worker se može horizontalno skalirati ako je queue prevelik
kubectl scale deploy/worker -n ticketing --replicas=3
```

### Restart bez promjene manifesta

```bash
# Korisno za reload Secret/ConfigMap promjena u podu
kubectl rollout restart deploy/api -n ticketing
kubectl rollout restart deploy/worker -n ticketing
kubectl rollout restart deploy/frontend -n ticketing
```

### Pregled konfiguracije pri debug-u

```bash
# Trenutni ConfigMap
kubectl get configmap ticketing-config -n ticketing -o yaml

# Resource utilizacija (ako je metrics-server instaliran)
kubectl top pods -n ticketing
kubectl top nodes
```

---

## Health check — opći pregled stanja klastera

```bash
# Statusi svih podova
kubectl get pods -n ticketing -o wide

# Pregled svih ključnih objekata
kubectl get deploy,svc,ingress,pvc,configmap,secret -n ticketing

# Sigurno potroši test request kroz API pod (bypass Ingressa)
API_POD=$(kubectl get pod -n ticketing -l app=api -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n ticketing $API_POD -- wget -qO- http://localhost:8080/healthz
kubectl exec -n ticketing $API_POD -- wget -qO- http://localhost:8080/readyz

# Ručna provjera kroz Ingress (zamijeni hostname po potrebi)
curl -s http://ticketing.local/api/healthz
curl -s http://ticketing.local/api/readyz
```

---

## Eskalacija

Ako gornji koraci ne rješavaju problem unutar 15 minuta:

1. Snimi trenutno stanje za post-mortem:
   ```bash
   kubectl get all -n ticketing -o yaml > incident-snapshot.yaml
   kubectl describe pods -n ticketing > incident-pods-describe.txt
   kubectl logs -n ticketing --all-containers --prefix --tail=500 deploy/api > api-logs.txt
   ```
2. Otvori incident u project tracking sustavu i priloži snapshot
3. Kontaktiraj vlasnika servisa (vidi `CODEOWNERS` ili README)
