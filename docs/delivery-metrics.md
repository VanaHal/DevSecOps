
# Mjerenje brzine isporuke — ručni proces vs. automatizirani CI/CD



## Metodologija



Usporedba je napravljena na temelju stvarno izvedenih koraka tijekom razvoja ovog projekta — ne na sintetičkom benchmarku, nego na bilježenju vremena i broja ručnih koraka za istu vrstu zadatka ("izgraditi, provjeriti sigurnost, i objaviti novu verziju produkcijske slike u registry") prije i nakon uvođenja CI/CD pipelinea.



## Ručni proces (prije CI/CD, Faza 7 i rani dio Faze 9)



Koraci koje je developer morao ručno izvesti za svaku sliku:



1. `podman build --target production -t <naziv> ./<servis>` (~15-30s po slici)

2. Ručno pokretanje Trivy skena (`trivy image <naziv>`) i vizualni pregled tablice rezultata

3. Uspostava port-forwarda prema internom registryju (`oc port-forward ...`), uz ponavljanje ako je port već zauzet od prošle sesije

4. Generiranje i korištenje autentikacijskog tokena (`oc create token registry-pusher`), s poznatim rizikom greške pri ručnom kopiranju dugog tokena (stvarno se dogodilo tijekom projekta — token je oštećen pri copy-paste, zahtijevalo dodatnu iteraciju rješavanja kroz `--password-stdin` pristup)

5. `podman tag` i `podman push` za svaku sliku pojedinačno

6. `oc rollout restart` za svaki Deployment da povuče novu sliku

7. Ručna vizualna provjera (`oc get pods -w`) da su svi podovi ponovno `1/1 Running`



**Ukupno, za tri servisa, ovaj proces je u praksi trajao između 10 i 20 minuta** aktivnog rada, ovisno o tome je li nešto pošlo po zlu (npr. zauzet port, neispravan token) — što se, realno, dogodilo više puta tijekom projekta.



## Automatizirani proces (nakon CI/CD, Faza 8)



Jedina ručna radnja: `git push`.



Izmjereno na stvarnom CI/CD run-u (GitHub Actions, sva tri servisa paralelno kroz matrix strategiju):



| Servis | Trajanje |

|---|---|

| api | 2m 20s |

| worker | 2m 20s |

| frontend | 2m 14s |

| **Ukupno (paralelno)** | **2m 25s** |



Ovo uključuje: checkout koda, build produkcijske slike, dva Trivy skena (informativni + quality gate), upload SARIF artifacta, i push na GHCR — za sva tri servisa istovremeno, bez ijedne ručne intervencije.



## Usporedba



| Metrika | Ručni proces | CI/CD (automatizirano) | Poboljšanje |

|---|---|---|---|

| Vrijeme (tri servisa) | ~10-20 min | 2m 25s | **~80-88% brže** |

| Broj ručnih koraka | ~7 po servisu (21 ukupno) | 1 (`git push`) | **~95% manje ručnih koraka** |

| Rizik ljudske greške (npr. zaboravljen sken, oštećen token) | Prisutan, stvarno se dogodio tijekom projekta | Eliminiran (svaki korak deterministički skriptiran) | — |

| Konzistentnost (jednak proces svaki put) | Ovisi o disciplini developera | Zajamčena (isti workflow.yml za svaki push) | — |

| Sigurnosna provjera prije objave | Ručna, mogla se preskočiti pod pritiskom vremena | Obavezna, blokira push ako ne prođe | — |



## Zaključak



Najveći dobitak automatizacije nije samo brzina (iako je značajna, ~80%+ ušteda vremena), nego **eliminacija mogućnosti da se sigurnosni sken slučajno preskoči** — u ručnom procesu, sken je bio odvojen, opcionalan korak koji je ovisio o disciplini; u CI/CD procesu, sken je ugrađen kao **nepropusni gate** koji fizički onemogućuje objavu nesigurne slike, bez obzira na to je li developer "zaboravio" ili požurio.

