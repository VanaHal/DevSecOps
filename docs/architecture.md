
# Arhitektura i usporedba pristupa (kontejneri vs. virtualne mašine)



## Arhitekturni dijagram



```

                    ┌─────────────┐

   korisnik ──────► │  frontend   │  (Node.js, port 3000)

                    └──────┬──────┘

                           │ HTTP (REST)

                    ┌──────▼──────┐

                    │     api     │  (Node.js/Express, port 8080)

                    └──┬───────┬──┘

                       │       │

                ┌──────▼─┐   ┌─▼──────┐

                │postgres│   │ redis  │

                │ :5432  │   │ :6379  │

                └────────┘   └───┬────┘

                                  │ Redis red: ticket_orders

                            ┌─────▼─────┐

                            │  worker   │  (Node.js, konzumira red)

                            └───────────┘

```



**Tok podataka pri kupnji karte:**

1. Korisnik šalje zahtjev preko frontenda → api

2. Api upisuje narudžbu u Redis red (`ticket_orders`) i **odmah** vraća odgovor (`"Order queued"`, `orderId`) — korisnik ne čeka obradu

3. Worker asinkrono konzumira red, upisuje potvrđenu narudžbu u PostgreSQL

4. Ova arhitektura razdvaja **sinkroni** korisnički odgovor od **asinkrone** obrade, čime api ostaje brz i responzivan čak i pod opterećenjem ili privremenim problemima s bazom



## Zašto kontejneri, a ne virtualne mašine — obrazloženje za ovu aplikaciju



Odluka o kontejnerizaciji nije donesena generički ("kontejneri su uvijek bolji"), nego na temelju konkretnih karakteristika ove aplikacije:



### 1. Servisi su stateless, stanje je izdvojeno



`frontend`, `api` i `worker` ne drže nikakvo trajno stanje unutar sebe — svo stanje (narudžbe, red zadataka) živi u `postgres` i `redis`. To je upravo scenarij za koji su kontejneri idealni: proces se može ugasiti, zamijeniti, ili skalirati bez brige o gubitku podataka, jer podaci nisu ni bili unutra. Kod VM pristupa, ova prednost postoji i tamo, ali s puno većim overheadom po instanci.



### 2. Brzina pokretanja i gustoća



Naši kontejneri (Node.js Alpine bazirane slike) pokreću se u **sekundama** i teže su **~140 MB**. Ekvivalentna VM (čak i minimalna, npr. Alpine Linux VM s istim runtime-om) zahtijeva zaseban kernel, boot proces od desetaka sekundi do minuta, i gigabajte diskovnog prostora po instanci. Za pet servisa koje pokrećemo (`frontend`, `api`, `worker`, `postgres`, `redis`), razlika u gustoći (koliko instanci stane na isti hardver) i brzini pokretanja izravno utječe na brzinu rolling update/rollback operacija koje demonstriramo (vidi `docs/delivery-metrics.md`).



### 3. Konzistentnost okruženja kroz cijeli ciklus



Isti Containerfile `production` stage koristi se lokalno (test build), u CI/CD pipelineu (build+scan), i na OpenShift klasteru (deploy). Kod VM pristupa, ekvivalentna konzistentnost zahtijeva alate poput Packer/golden image pipelinea koji su znatno teži za održavati na razini pojedinačnog razvojnog projekta ovog opsega.



### 4. Orkestracija i samo-oporavak



Kubernetes/OpenShift readiness i liveness probe (implementirane za sve naše servise, uključujući `exec` probu za worker) omogućuju automatsko prepoznavanje i zamjenu nezdravih instanci. Naš rollback demo (`docs/runbook.md`, Scenarij 1) pokazuje da se ovo postiže s jednom naredbom (`oc rollout undo`) — ekvivalentan proces na VM infrastrukturi (bez orkestratora) zahtijevao bi ručnu intervenciju ili zaseban, teži alat (npr. Ansible playbook, load balancer reconfiguration).



### Gdje bi VM pristup i dalje imao smisla (poštena usporedba)



Kontejneri nisu univerzalno rješenje. Za `postgres`, na primjer, u velikim produkcijskim okruženjima često se koristi **managed database servis** (RDS, Cloud SQL) ili dedicirana VM s finijom kontrolom nad diskovnim I/O i memorijom, umjesto kontejnera — jer baza ima drugačije zahtjeve za perzistenciju i performanse od stateless aplikacijskih servisa. U našem projektu smo svjesno zadržale Postgres kao kontejner (uz PVC za perzistenciju) zbog opsega vježbe i jednostavnosti demonstracije cijelog stacka unutar jednog klastera — u stvarnoj produkcijskoj arhitekturi većeg opsega, razdvajanje "managed state" od kontejneriziranih stateless servisa bi bilo standardna praksa.



## Sažetak usporedbe



| Kriterij | Kontejneri (naš pristup) | Virtualne mašine |

|---|---|---|

| Vrijeme pokretanja | Sekunde | Desetci sekundi – minute |

| Veličina po instanci | ~140 MB (production slike) | Gigabajti |

| Izolacija | Proces/namespace razina (dijeli kernel hosta) | Puna, zaseban kernel |

| Gustoća (instanci po hardveru) | Visoka | Niska |

| Orkestracija/samo-oporavak | Nativno (K8s probe, rollout) | Zahtijeva dodatne alate |

| Konzistentnost dev→produkcija | Ista slika, svi stageovi | Zahtijeva golden image pipeline |

| Prikladno za naš scenarij | ✅ (stateless servisi, brz razvoj, orkestrirana isporuka) | Djelomično (baza u većem opsegu) |

