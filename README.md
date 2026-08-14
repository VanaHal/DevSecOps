
# Secure Event Ticketing Platform



DevSecOps projekt: mikroservisna aplikacija za prodaju ulaznica, s kompletnim ciklusom od lokalnog razvoja do produkcijskog OpenShift deploymenta, uključujući sigurnosno skeniranje, CI/CD pipeline i runbook za incidente.



## Arhitektura



```

                    ┌─────────────┐

   korisnik ──────► │  frontend   │  (Node.js, port 3000)

                    └──────┬──────┘

                           │ HTTP

                    ┌──────▼──────┐

                    │     api     │  (Node.js/Express, port 8080)

                    └──┬───────┬──┘

                       │       │

                ┌──────▼─┐   ┌─▼──────┐

                │postgres│   │ redis  │

                │ :5432  │   │ :6379  │

                └────────┘   └───┬────┘

                                  │ ticket_orders queue

                            ┌─────▼─────┐

                            │  worker   │  (Node.js, preuzima poruke)

                            └───────────┘

```



- **frontend** — poslužuje statičku stranicu, šalje zahtjeve api-ju

- **api** — prima narudžbe, upisuje ih u Redis red (`ticket_orders`), odmah vraća odgovor korisniku

- **worker** — preuzima poruke iz Redis reda, upisuje potvrđene narudžbe u PostgreSQL

- **postgres** — perzistentna pohrana narudžbi

- **redis** — red poruka između api i worker servisa



## Preduvjeti



- Podman + podman-compose (ili Docker + docker-compose)

- Pristup OpenShift/Kubernetes klasteru (`oc` CLI) za produkcijski dio

- Node.js 20+ (samo ako se razvija izvan kontejnera)



---



# 1. dio — Lokalni razvoj (Compose)



## Pokretanje cijelog stacka



```bash

cp .env.example .env

# uredi .env po potrebi (lozinke, portovi)


podman-compose up -d

```



Ova naredba pokreće sve servise (postgres, redis, api, worker, frontend) u dev modu, s hot-reload-om preko nodemon i mount-anim src folderima — izmjene u kodu odmah se reflektiraju bez ponovnog builda slike.



## Provjera funkcionalnosti



```bash

curl http://localhost:8080/healthz

```



Očekivani odgovor: `{"status":"ready"}`



Pristupiti `http://localhost:3000` u pregledniku, napraviti testnu kupnju, i provjeriti worker log:



```bash

podman-compose logs worker --tail=20

```



Trebalo bi se vidjeti zapis `Order processed`.



## Zaustavljanje



```bash

podman-compose down

```



Podaci u PostgreSQL-u ostaju sačuvani između pokretanja zahvaljujući named volumeu (`pgdata`) — za potpuno brisanje podataka:



```bash

podman-compose down -v

```



## Struktura Containerfile-ova



Svaki servis (`api/`, `worker/`, `frontend/`) ima multi-stage `Containerfile`:



- **`prod-deps`** — instalira samo produkcijske ovisnosti (`npm ci --omit=dev`)

- **`dev`** — dodaje dev ovisnosti i `nodemon` za hot-reload, koristi se u `compose.yaml`

- **`production`** — minimalna runtime slika, non-root korisnik (UID 1001), `npm`/`npx`/`corepack` uklonjeni, `apk upgrade` za najnovije sigurnosne zakrpe — koristi se isključivo za K8s/OpenShift deploy



---



# 2. dio — Produkcijski deployment (OpenShift/Kubernetes)



## Preduvjeti



```bash

export KUBECONFIG=~/.auth/ocp4-kubeconfig   # prilagodi putanju

oc login <cluster-url>

oc new-project devops-ticketing   # ili oc project devops-ticketing ako već postoji

```



> Napomena: `KUBECONFIG` varijablu potrebno trajno dodati u `~/.bashrc` kako ju ne bi bilo potrebno ručno postavljati u svakoj novoj terminal sesiji:

> ```bash

> echo 'export KUBECONFIG=~/.auth/ocp4-kubeconfig' >> ~/.bashrc

> ```



## Deploy svih servisa



```bash

oc apply -f k8s/postgres.yaml

oc apply -f k8s/redis.yaml

oc apply -f k8s/api.yaml

oc apply -f k8s/worker.yaml

oc apply -f k8s/frontend.yaml

oc apply -f k8s/networkpolicy.yaml

```



## Prva inicijalizacija baze (samo pri prvom postavljanju, prazan PVC)



Kubernetes Deployment nema automatski `docker-entrypoint-initdb.d` mehanizam kao lokalni Compose, pa je shemu potrebno ručno učitati:



```bash

POSTGRES_POD=$(oc get pods -l app=postgres -o jsonpath='{.items[0].metadata.name}')

oc cp infra/postgres/init.sql $POSTGRES_POD:/tmp/init.sql

oc exec $POSTGRES_POD -- psql -U ticketing_user -d ticketing -f /tmp/init.sql

```



## Provjera statusa



```bash

oc get pods

oc get route frontend

```



Pristupiti URL-u iz `oc get route frontend` u pregledniku i testirati kupnju karte.



## Sigurnosni minimum implementiran u produkciji



- **ConfigMap + Secret** — nema hardkodiranih lozinki, sve osjetljive vrijednosti idu kroz `postgres-secret`

- **Liveness/readiness probe** — za sve servise (api/frontend: HTTP; postgres/redis: TCP; worker: `exec` proba jer nema HTTP port)

- **Resource requests/limits** — definirani za sve Deploymente

- **RBAC** — zaseban `ServiceAccount` po servisu, `automountServiceAccountToken: false` (servisi ne pozivaju Kubernetes API pa im token nije potreban)

- **securityContext** — `runAsNonRoot: true`, `allowPrivilegeEscalation: false`, `capabilities: drop: ["ALL"]`, `seccompProfile: RuntimeDefault` na svim workload-ovima (bez forsiranja fiksnog `runAsUser` — OpenShift dodjeljuje nasumični UID unutar dozvoljenog raspona)

- **NetworkPolicy** — `default-deny-all` baseline, uz eksplicitna pravila: postgres/redis primaju promet samo od api/worker; api prima promet od frontenda i OpenShift routera; frontend prima promet samo od routera

- **Rolling update / rollback** — demonstrirano i dokumentirano (vidi `docs/runbook.md`, Scenarij 2)



## Rolling update i rollback



```bash

oc set image deployment/api api=<novi-image-tag>

oc rollout status deployment/api


# u slučaju problema:

oc rollout undo deployment/api

```



---



# CI/CD Pipeline



GitHub Actions workflow (`.github/workflows/ci.yml`) se pokreće na svaki push/PR prema `main`:



1. **Build** — produkcijska (`target: production`) slika za svaki servis (`api`, `worker`, `frontend`)

2. **Trivy sken (informativni)** — čitljiva tablica u logu, ne blokira pipeline

3. **Trivy sken (quality gate)** — ako postoji CRITICAL/HIGH ranjivost s dostupnim fixom, pipeline staje i push se ne izvršava

4. **SARIF izvještaj** — uploadan kao artifact (30 dana čuvanja), pokušava se poslati u GitHub Security tab (zahtijeva GitHub Advanced Security — nedostupno za privatne repozitorije bez plaćene licence)

5. **Push na GHCR** — samo ako je gate prošao i radi se o push-u na `main`



CVE iznimke (ako ih ima) dokumentirane su u `.trivyignore`.



---



# Dokumentacija



| Dokument | Sadržaj |

|---|---|

| `docs/architecture.md` | Arhitekturni dijagram i kritička usporedba kontejnera i virtualnih mašina za ovu aplikaciju |

| `docs/image-policy.md` | Politika upravljanja slikama — tagging strategija, sadržaj production slike, uvjeti objave |

| `docs/devsecops.md` | DevSecOps metodologija — obrazloženje odabranih alata i sigurnosnih kontrola u pipelineu |

| `docs/delivery-metrics.md` | Mjerenje brzine isporuke — ručni proces vs. automatizirani CI/CD |

| `docs/security/image-scan-report.md` | Detaljan Trivy sken svih slika (prije prve K8s implementacije) |

| `docs/runbook.md` | Tri testirana scenarija (pad baze, loš image tag, neispravan secret), format Simptom → Dijagnoza → Popravak → Validacija |



---



# Struktura repozitorija



```

.

├── api/                    # API servis (Node.js/Express)

├── worker/                 # Worker servis (preuzima poruke iz Redis reda)

├── frontend/               # Frontend servis

├── infra/postgres/init.sql # Shema baze

├── k8s/                    # Kubernetes/OpenShift manifesti

├── docs/

│   ├── architecture.md
│   ├── image-policy.md
│   ├── devsecops.md
│   ├── delivery-metrics.md
│   ├── security/image-scan-report.md
│   └── runbook.md

├── .github/workflows/ci.yml

├── .trivyignore

├── compose.yaml

├── .env.example

└── README.md

```

