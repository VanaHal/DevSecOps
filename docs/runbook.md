
# Runbook za incidente — DevSecOps Ticketing Platform



Ovaj dokument opisuje tri stvarno simulirana i testirana scenarija na OpenShift klasteru (namespace `devops-ticketing`).



---



## Scenarij 1 — Pad baze podataka



### Simptom

Kupnja karte i dalje "prolazi" na frontendu (`Order queued` + orderId) jer api samo gura narudžbu u Redis red i odmah vraća odgovor, ne čekajući bazu — problem je skriven od korisnika.



### Dijagnoza

```bash

oc scale deployment/postgres --replicas=0

oc logs deployment/worker --tail=20

```

Worker log:

```

Worker loop error: connect ECONNREFUSED 172.30.22.232:5432

```

Api log ostaje potpuno tih — api nije svjestan problema jer njegov posao završava čim gurne poruku u red.



### Popravak

```bash

oc scale deployment/postgres --replicas=1

oc rollout restart deployment/worker

```

Worker se sam ne oporavlja nakon gubitka konekcije (nema retry logiku), potreban je ručni restart.



### Validacija

```bash

oc get pods -l app=postgres

oc logs deployment/worker --tail=20

```



### Otkriveni arhitekturni nedostatak

Worker koristi blokirajuću "pop" operaciju (`BRPOP`) koja trajno uklanja poruku iz Redis reda čim je pročita, prije uspješne obrade. Ako obrada padne (baza nedostupna), poruka je trajno izgubljena — nema "at-least-once" garancije. Potvrđeno upitom na `ticket_orders`: narudžba naručena za vrijeme pada baze nikad nije zapisana u bazu, unatoč potvrdi "Order queued" korisniku.



**Preporuka:** Redis Streams s consumer grupama i `XACK`, ili "peek-then-remove" pristup gdje se poruka briše iz reda tek nakon potvrđenog upisa u bazu.



---



## Scenarij 2 — Loš image tag (rolling update/rollback)



### Simptom

Nakon `oc set image` s nepostojećim tagom, novi pod ne postaje spreman.



### Dijagnoza

```bash

oc rollout status deployment/api

oc get pods -l app=api

```

Izlaz pokazuje dva poda: stari (`1/1 Running`) i novi (`0/1 ImagePullBackOff`). Rollout status javlja "1 old replicas are pending termination" — Kubernetes ne gasi stari pod dok novi ne prođe readiness proveru. Aplikacija ostaje potpuno dostupna korisnicima tijekom cijelog incidenta.



### Popravak

```bash

oc rollout undo deployment/api

oc rollout status deployment/api

```



### Validacija

```bash

oc get pods -l app=api

oc get deployment api -o jsonpath='{.spec.template.spec.containers[0].image}'

```

Potvrđen povratak na ispravnu sliku, jedan zdrav pod, uspješna kupnja karte na frontendu.



---



## Scenarij 3 — Neispravan secret (kriva lozinka baze)



### Simptom

Nakon izmjene `POSTGRES_PASSWORD` u Secretu i restarta api/worker podova, autentikacija na bazu počinje padati.



### Dijagnoza

```bash

oc logs deployment/worker --tail=20

```

```

Worker fatal error: error: password authentication failed for user "ticketing_user"

```



### Popravak

```bash

oc set data secret/postgres-secret --from-literal=POSTGRES_PASSWORD=<ispravna_lozinka>

oc rollout restart deployment/api deployment/worker

```



### Validacija

Uspješna kupnja karte + `Order processed` u worker logu.



### Ključna lekcija

Promjena Kubernetes Secreta ne primorava Postgres da automatski promijeni internu lozinku — to ovisi o tome je li Postgres pod restartan nakon izmjene. Za pouzdanu rotaciju lozinke potreban je ili eksplicitan `ALTER USER ... PASSWORD` unutar baze, ili potvrđen restart Postgres Deploymenta zajedno s aplikacijskim servisima.



---



## Opća napomena o okruženju



Kroz sve scenarije korištena je varijabla `KUBECONFIG`, koja se gubi u svakoj novoj terminal sesiji ako nije trajno postavljena:

```bash

echo 'export KUBECONFIG=~/.auth/ocp4-kubeconfig' >> ~/.bashrc

```

