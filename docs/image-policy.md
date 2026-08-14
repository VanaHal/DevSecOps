
# Politika upravljanja kontejnerskim slikama



## Strategija build-a: multi-stage



Svaki servis (`api`, `worker`, `frontend`) ima `Containerfile` s tri stagea:



| Stage | Svrha | Koristi se u |

|---|---|---|

| `prod-deps` | Instalira samo produkcijske npm ovisnosti (`npm ci --omit=dev`) | Osnova za `production` stage |

| `dev` | Dodaje dev ovisnosti + `nodemon` za hot-reload | Lokalni razvoj (`compose.yaml`) |

| `production` | Minimalna runtime slika, non-root, bez build alata | CI/CD i K8s/OpenShift deploy |



Razdvajanje `dev` i `production` stageova nije kozmetičko — otkriveno je tijekom projekta da je greškom pushana `dev` slika (s `nodemon`-om) na OpenShift, što je uzrokovalo da worker proces prestane raditi nakon greške bez automatskog restarta, dok je Kubernetes i dalje prijavljivao pod kao "zdrav" (vidi `docs/runbook.md`, Scenarij 5 u proširenoj verziji). Ovo je konkretan, stvaran primjer zašto strogo razdvajanje ovih stageova ima sigurnosnu i operativnu težinu, ne samo teoretsku.



## Sadržaj production slike (hardening)



- Bazna slika: `node:20-alpine` (minimalna Alpine varijanta)

- `RUN apk update && apk upgrade --no-cache` — povlači najnovije sigurnosne zakrpe za OS pakete pri svakom buildu (rješava CVE u `openssl`-derivatima `libssl3`/`libcrypto3` otkrivene Trivy skenom)

- Non-root korisnik: `addgroup -S appgroup && adduser -S -u 1001 -G appgroup appuser`, `USER 1001`

- Uklonjeni build/paket alati koji nisu potrebni u runtimeu: `npm`, `npx`, `corepack` (`rm -rf /usr/local/lib/node_modules/npm /usr/local/bin/npm /usr/local/bin/npx /usr/local/bin/corepack`) — smanjuje napadnu površinu jer aplikacija u produkciji nikad ne treba pokretati `npm install` niti proizvoljne pakete preko `npx`

- Rezultat: ~140 MB po slici (izmjereno lokalno), bez nepotrebnih razvojnih alata



## Tagging politika



Svaka slika se u CI/CD pipelineu tagira **dvostruko**:



```yaml

tags: |

  ghcr.io/<owner>/devops-project-app-<servis>:latest

  ghcr.io/<owner>/devops-project-app-<servis>:<github.sha>

```



- **`:latest`** — uvijek pokazuje na najnoviju sliku koja je prošla quality gate na `main` grani; koristi se za standardni deploy

- **`:<commit-sha>`** — nepromjenjiv, jedinstven tag vezan uz točan commit; omogućuje precizan rollback na **točno poznatu** verziju koda (`oc rollout undo` vraća na prethodnu reviziju Deploymenta, ali SHA tag omogućuje i ručni, eksplicitni odabir bilo koje prošle verzije ako je potrebno)



**Zašto ne samo `:latest`:** oslanjanje isključivo na `:latest` otežava audit trag (koja točno verzija koda je bila u produkciji u danom trenutku) i onemogućuje precizan rollback na specifičnu, ne nužno "prethodnu", verziju. Kombinacija oba taga daje i jednostavnost (`:latest` za svakodnevni deploy) i preciznost (`:sha` za audit i ciljani rollback).



**Prostor za poboljšanje (svjesno ostavljeno izvan opsega ovog projekta):** semantičko verzioniranje (`vX.Y.Z`) kroz git tagove i `docker/metadata-action` bilo bi sljedeći korak za produkcijski zreliji sustav, gdje bi verzije bile čitljive ljudima (ne samo SHA), uz jasnu razliku između major/minor/patch izmjena.



## Politika objave (kad se slika smije pushati)



Push u registry (GHCR) izvršava se **isključivo** ako:

1. Radi se o `push` eventu na `main` granu (ne na svaki PR/feature branch)

2. Slika je uspješno izgrađena (`target: production`)

3. Trivy quality gate je prošao (nema CRITICAL/HIGH ranjivosti s dostupnim popravkom)



```yaml

- name: Push image (only if scan passed and on main)

  if: github.event_name == 'push' && github.ref == 'refs/heads/main'

```



Ovo implementira "secure-by-default" princip — nesigurna slika nikad ne stigne u registry, čak ni privremeno.



## Skeniranje i evidencija ranjivosti



Svaka slika prolazi Trivy sken u dva koraka pipelinea:

1. **Informativni** (`format: table`, `exit-code: 0`) — čitljiv izvještaj u CI logu, ne blokira

2. **Quality gate** (`format: json`, `severity: CRITICAL,HIGH`, `exit-code: 1`) — blokira push ako postoji CRITICAL/HIGH nalaz s dostupnim fixom



Detaljno sigurnosno izvješće s popisom svih pronađenih i otklonjenih ranjivosti nalazi se u `docs/security/image-scan-report.md`. Trenutni preostali nalaz nakon svih popravaka: `CVE-2026-41907` (paket `uuid`, MEDIUM, ima dostupan fix na `uuid@11+`) — ne blokira gate jer gate cilja samo CRITICAL/HIGH, dokumentirano kao poznat, prihvaćen rizik niskog prioriteta za ovaj opseg projekta.

