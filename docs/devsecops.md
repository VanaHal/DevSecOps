
# DevSecOps metodologija — alati, prakse i obrazloženje odabira



## Shift-left princip u ovom projektu



"Shift-left" znači pomicanje sigurnosnih provjera što ranije u razvojni ciklus, umjesto da se sigurnost provjerava tek prije produkcijskog deploya. U ovom projektu to je implementirano kroz:



1. **Sken slike prije bilo kakvog deploya** — Trivy sken pokreće se lokalno (Faza 7, ručno) prije prvog K8s deploya, i automatski u CI/CD prije svakog pusha slike

2. **Quality gate koji stvarno blokira** — ne samo izvještava o ranjivostima, nego zaustavlja pipeline (`exit-code: 1`) ako su pronađene CRITICAL/HIGH ranjivosti s dostupnim popravkom

3. **Sigurnost ugrađena u sam Containerfile** — non-root korisnik, uklonjeni nepotrebni alati, `apk upgrade` — nije naknadna zakrpa, nego dio definicije slike od početka



## Odabrani alati i zašto



| Alat | Svrha | Zašto ovaj, a ne alternativa |

|---|---|---|

| **Trivy** | Skeniranje ranjivosti kontejnerskih slika (OS paketi + npm ovisnosti) | Open-source, brz, pokriva i OS-level i jezično-specifične (node-pkg) ranjivosti u jednom alatu, laka GitHub Actions integracija |

| **GitHub Actions** | CI/CD orkestracija | Nativna integracija s GitHub repozitorijem, besplatan za privatne repozitorije u razumnim granicama, dobra podrška za Docker/Buildx |

| **GHCR (GitHub Container Registry)** | Registry za produkcijske slike | Besplatan za privatne repozitorije uz GitHub Actions token (`GITHUB_TOKEN`), nema potrebe za dodatnim vanjskim servisom/kredencijalima |

| **OpenShift SCC (Security Context Constraints)** | Nametanje non-root/arbitrary-UID izvršavanja na razini klastera | Nadograđuje standardni Kubernetes `securityContext` — čak i ako bi Containerfile/manifest dopustio root, klaster to odbija |

| **Kubernetes NetworkPolicy** | Segmentacija mrežnog prometa (zero-trust) | Nativan Kubernetes resurs, ne zahtijeva dodatni CNI plugin na OpenShiftu (za razliku od nekih k3s/Flannel postavki gdje NetworkPolicy zahtijeva zamjenu CNI-ja za Calico/Cilium) |



## Sigurnosne kontrole implementirane kroz pipeline (mapa na I4 kriterije)



1. **Sigurnosne provjere u CI/CD toku** — Trivy sken (informativni + quality gate) na svaku sliku, svaki push

2. **Quality gate prije objave** — push na GHCR uvjetovan uspješnim prolaskom skena

3. **Tajne i konfiguracija bez hardcodinga** — sve lozinke idu kroz Kubernetes `Secret`, sve ne-osjetljive vrijednosti (portovi, imena hostova) kroz `ConfigMap`; niti jedna lozinka nije zapisana u kodu ili Containerfile-u

4. **Least-privilege pristup** — zaseban `ServiceAccount` po servisu, `automountServiceAccountToken: false` (servisi ne pozivaju Kubernetes API), `securityContext` s `allowPrivilegeEscalation: false` i `capabilities: drop: ["ALL"]` na svim workload-ovima



## Dosljednost nalaza i korektivnih mjera — primjer iz projekta



Tijekom implementacije CI/CD pipelinea otkriven je i ispravljen konkretan lanac problema, dokumentiran ovdje radi transparentnosti procesa:



1. **Nalaz:** Trivy sken otkrio `CVE-2026-45447` (HIGH) na `libssl3`/`libcrypto3` u baznoj `node:20-alpine` slici

2. **Korekcija:** dodan `apk update && apk upgrade --no-cache` u `production` stage svih Containerfile-ova

3. **Validacija:** ponovni sken potvrdio nestanak oba HIGH nalaza (verificirano usporedbom SARIF izvještaja prije/poslije)

4. **Sekundarni nalaz:** quality gate je nastavio padati unatoč čistom rezultatu — istraženo i utvrđeno da `trivy-action` u `sarif` formatu interno zaobilazi `severity` filter za potrebe potpunosti GitHub Security tab izvještaja

5. **Korekcija arhitekture pipelinea:** odvojeno generiranje SARIF izvještaja (uvijek "meko", `exit-code: 0`) od stvarne quality gate provjere (`format: json`, gdje `severity` filter ispravno funkcionira)

6. **Validacija:** pipeline uspješno prolazi za sva tri servisa, samo preostali MEDIUM nalaz (`uuid` paket) ne blokira gate, u skladu s namjerom (gate cilja CRITICAL/HIGH)



Ovaj primjer pokazuje sustavan pristup: nalaz → korekcija → validacija → dodatni nalaz → korekcija arhitekture → konačna validacija, umjesto zaobilaženja provjere (npr. brisanjem gate-a) kad je prvi pokušaj popravka otkrio dodatnu složenost.

