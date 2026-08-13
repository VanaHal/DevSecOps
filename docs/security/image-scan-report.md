# Sigurnosno izvjesce - Trivy sken kontejnerskih slika



**Datum skeniranja:** 13.08.2026

**Alat:** Trivy v0.73.0

**Skenirane slike:** devops-project-app_api, devops-project-app_worker, devops-project-app_frontend

**Baza slika:** node:20-alpine (Alpine 3.23.4)



## Sazetak rezultata



| Slika | OS sloj (Alpine) | Node.js paketi (node-pkg) | Ukupno HIGH/CRITICAL |

|---|---|---|---|

| api | HIGH: 2, CRITICAL: 0 | HIGH: 17, CRITICAL: 1 | 19 HIGH, 1 CRITICAL |

| worker | HIGH: 2, CRITICAL: 0 | HIGH: 17, CRITICAL: 1 | 19 HIGH, 1 CRITICAL |

| frontend | HIGH: 2, CRITICAL: 0 | HIGH: 17, CRITICAL: 1 | 19 HIGH, 1 CRITICAL |



Rezultati su gotovo identicni kroz sve tri slike jer sve tri dijele istu baznu sliku

(node:20-alpine) i istu verziju npm-a s istim tranzitivnim ovisnostima.



## Kljucni nalazi



### 1. CRITICAL - tar (node-pkg), verzija 6.2.1



- CVE-2026-59873 - Denial of Service preko zlonamjerne "gzip bomb" arhive

- Paket je tranzitivna ovisnost npm-a (node_modules/npm/node_modules/tar), ne

  izravna ovisnost aplikacije

- Fix dostupan: 7.5.19

- Prisutan u sve tri slike (api, worker, frontend)



### 2. HIGH - openssl (OS sloj: libcrypto3 / libssl3), verzija 3.5.6-r0



- CVE-2026-45447 - Heap Use-After-Free u PKCS7_verify()

- Dio Alpine base image (node:20-alpine)

- Fix dostupan: 3.5.7-r0 (rebuild s azuriranom bazom rjesava problem)



### 3. HIGH - vise paketa vezanih uz regex/DoS ranjivosti



Sljedeci tranzitivni npm paketi (dio npm-ove interne strukture, ne direktne

ovisnosti aplikacije) imaju vise HIGH ranjivosti tipa "Denial of Service" preko

regex backtrackinga ili exponential-time kompleksnosti:



- brace-expansion (2.0.1) - 3 CVE-a (DoS preko exponential-time complexity)

- minimatch (vise verzija) - 2 CVE-a (DoS preko catastrophic backtracking)

- cross-spawn (7.0.3) - CVE-2024-21538 (regex DoS)

- glob (10.4.2) - CVE-2025-64756 (command injection preko malicioznih filenamea)

- ip-address (9.0.5) - CVE-2026-69192 (SSRF preko inconsistent IP parsinga),

  CVE-2026-42338 (XSS preko HTML escapinga)

- sigstore / @sigstore/core - CVE-2026-48815, CVE-2026-48758 (signature bypass)



Sve navedene ranjivosti imaju dostupne fix verzije.



## Analiza uzroka (root cause)



Sve pronadene CRITICAL i vecina HIGH ranjivosti ne dolaze iz koda aplikacije

(api/src, worker/src, frontend/src) niti iz direktnih dependencies u

package.json, nego iz:



1. Bazne slike (node:20-alpine) - zastarjela verzija openssl unutar Alpine sloja

2. npm-ovih internih tranzitivnih paketa (node_modules/npm/node_modules/...) koji

   se instaliraju automatski uz sam npm alat tijekom npm ci, a ne uz aplikacijske

   ovisnosti



Buduci da se identican skup ranjivosti ponavlja kroz sve tri slike, zakljucak je da je

jedan zajednicki root cause (zastarjela node:20-alpine bazna slika) uzrok gotovo

svih nalaza, umjesto tri odvojena, nepovezana problema.



## Preporuke za remediaciju



1. Azurirati baznu sliku - koristiti najnoviju dostupnu node:20-alpine verziju

   (ili prijeci na node:22-alpine), koja ukljucuje noviji openssl paket bez

   CVE-2026-45447

2. Rebuild bez cachea nakon azuriranja baze:

   podman-compose build --no-cache

3. Pokrenuti npm audit fix unutar svakog servisa kako bi se, gdje je moguce,

   azurirale tranzitivne ovisnosti (tar, minimatch, brace-expansion, itd.) na

   verzije koje sadrze fix

4. Integrirati Trivy u CI/CD pipeline (Faza 8) kao quality gate koji zaustavlja

   build pri pronalasku CRITICAL ranjivosti bez dostupnog patcha

5. Ponoviti sken nakon primjene fixeva i azurirati ovaj izvjestaj s novim rezultatima



## Naredbe koristene za skeniranje



trivy image --severity CRITICAL,HIGH localhost/devops-project-app_api:latest

trivy image --severity CRITICAL,HIGH localhost/devops-project-app_worker:latest

trivy image --severity CRITICAL,HIGH localhost/devops-project-app_frontend:latest



## Napomena o okruzenju



Prije prvog uspjesnog skena bilo je potrebno pokrenuti Podman rootless socket servis,

jer Trivy komunicira s Podmanom preko API socketa:



systemctl --user start podman.socket



Bez ovog koraka Trivy vraca gresku:

unable to find the specified image ... podman error: unable to initialize Podman client: no podman socket found
