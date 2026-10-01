# FitAcademy: rezumatul complet al proiectului (până la 1 octombrie 2026)

Documentul explică ce s-a făcut, în ce ordine și de ce. Pentru fiecare decizie arată ce alte opțiuni existau și ce se mai poate îmbunătăți. După ce îl citești ar trebui să poți explica proiectul la un interviu fără alt material.

> **Stadiul pe scurt:** planificarea este gata. Avem viziunea, cercetarea, cele 53 de cerințe și roadmap-ul în 7 faze. **Încă nu există cod.** Discuția pentru Faza 1 a început și este pe pauză: am stabilit înregistrarea și consimțământul, dar nu și sesiunile și parolele.

---

## Cuprins

1. [Pe scurt](#1-pe-scurt)
2. [Cum am lucrat: metoda GSD](#2-cum-am-lucrat-metoda-gsd)
3. [Cronologia: ce s-a făcut, pas cu pas](#3-cronologia-ce-s-a-făcut-pas-cu-pas)
4. [Deciziile de produs](#4-deciziile-de-produs)
5. [Deciziile tehnice](#5-deciziile-tehnice)
6. [Setările de lucru (config.json)](#6-setările-de-lucru-configjson)
7. [Cerințele v1 (53)](#7-cerințele-v1-53)
8. [Roadmap-ul: 7 faze](#8-roadmap-ul-7-faze)
9. [Faza 1: ce am discutat până acum](#9-faza-1-ce-am-discutat-până-acum)
10. [Riscuri, întrebări deschise, ce se poate îmbunătăți](#10-riscuri-întrebări-deschise-ce-se-poate-îmbunătăți)
11. [Structura fișierelor](#11-structura-fișierelor)
12. [Cum îl pui pe GitHub ca să arate bine la interviu](#12-cum-îl-pui-pe-github-ca-să-arate-bine-la-interviu)
13. [Glosar](#13-glosar)

---

## 1. Pe scurt

**FitAcademy** este o aplicație mobilă de wellness personal. Utilizatorul își notează ce mănâncă și ce antrenamente face. Un **coach** îi spune apoi, cu explicații, ce să schimbe, în loc să-i arate doar cifre.

- **Valoarea centrală:** loghezi rapid, primești sfaturi clare și explicabile.
- **Pentru cine:** întâi este un proiect de portofoliu. Apoi îl folosești tu zilnic, împreună cu prieteni. Mai târziu poate fi lansat public.
- **v1 conține doar „bucla de bază”:** cont, profil și ținte, jurnal de mese, scanare de coduri de bare, antrenamente, coach, un admin minimal.
- **Socialul, gamificarea, Q&A cu antrenori, portofelul de abonamente și recompensele** rămân pentru după v1.
- **Tehnologii:** backend în Python (FastAPI) cu PostgreSQL. Aplicația mobilă este în React Native + Expo (TypeScript). Adminul web este în Next.js. Serverul stă pe un VPS în UE.
- **v1 e gata când:** backendul rulează în cloud, iar tu ai logat mese și antrenamente zilnic, cel puțin 2 săptămâni.

---

## 2. Cum am lucrat: metoda GSD

Am folosit **GSD** („Get Shit Done”), un set de comenzi pentru Claude Code. GSD transformă o idee într-un plan executabil, în etape fixe:

```
/gsd-new-project      →  întrebări → cercetare → cerințe → roadmap        ✅ GATA
/gsd-discuss-phase 1  →  decizii de implementare pentru Faza 1            ⏸ PE PAUZĂ
/gsd-plan-phase 1     →  plan detaliat pe task-uri (PLAN.md)              ○ urmează
/gsd-execute-phase 1  →  scrierea codului, cu commit-uri atomice          ○
/gsd-verify-work 1    →  verificarea că faza chiar face ce a promis      ○
```

**De ce GSD și nu „scriem direct cod”?**
- **Pentru interviu:** se vede procesul. Viziunea, cercetarea, cerințele cu ID-uri, roadmap-ul și deciziile sunt documentate și urmăribile.
- **Pentru calitate:** fiecare fază are criterii de succes verificabile. Fiecare cerință e mapată la exact o fază, deci nu se pierde nimic.
- **Pentru context:** totul stă în `.planning/`. O sesiune nouă de lucru, cu un alt agent sau cu tine peste 3 luni, pornește de la aceleași decizii.

**Alternative:** să începem direct cu codul, ceea ce e mai rapid pe termen scurt, dar duce la decizii pierdute și refaceri. Sau un document de design scris de mână, fără structura de faze și urmărire a cerințelor.

**Ce se poate îmbunătăți:** GSD are un cost de „ceremonie”, adică mulți pași și multe documente. La un proiect solo, fazele foarte mici pot fi rulate cu `/gsd-quick` sau `/gsd-fast` ca să nu pierzi timp.

---

## 3. Cronologia: ce s-a făcut, pas cu pas

Fiecare pas important are un commit separat în git. Istoricul arată astfel:

| # | Commit | Ce conține |
|---|--------|------------|
| 1 | `ff20583 docs: initialize project` | `PROJECT.md`, viziunea sintetizată din Google Doc-ul tău plus răspunsurile tale |
| 2 | `98be092 chore: add project config` | `config.json`, cu preferințele de lucru |
| 3 | `729346a chore: add MCP config and gitignore` | `.mcp.json` + `.gitignore` (uneltele GSD au fost scoase ulterior din istoric, vezi secțiunea 12) |
| 4 | `47020e4 docs: complete project research` | cele 4 rapoarte de cercetare |
| 5 | `85d0af5 docs: add research summary` | `SUMMARY.md`, sinteza cercetării |
| 6 | `c7aacfb docs: define v1 requirements` | `REQUIREMENTS.md`, cu 53 de cerințe |
| 7 | `f7ee384 docs: create roadmap (7 phases)` | `ROADMAP.md`, `STATE.md`, `.claude/CLAUDE.md` |

### Pasul 1: inițializare
- Folderul `/Users/macbookpro/Claude` nu era un repo git, așa că am rulat `git init`.
- Nu exista cod. Proiectul e deci **greenfield** (pornit de la zero), fără cartografierea unui cod existent.

### Pasul 2: întrebările (deep questioning)
Mi-ai dat Google Doc-ul „FitAcademy Structure / Health Companion”. L-am citit integral și ți-am pus doar întrebările la care documentul nu răspundea:

| Întrebare | Răspunsul tău |
|-----------|---------------|
| Ce intră în v1? | Doar bucla de bază |
| Pentru cine este? | Portofoliu, apoi tu și prietenii, apoi lansare publică (toate trei, în ordine) |
| Framework mobil? | „Ceva care să conțină și Python, dar aștept recomandări” |
| Nume? | FitAcademy |
| Cum arată un insight bun? | Card zilnic + review săptămânal |
| LLM în v1? | Nu. Întâi reguli + șabloane de text |
| Admin web în v1? | Minimal |
| Când e v1 „gata”? | Backend deployat + îl folosești zilnic 2+ săptămâni |
| Limba aplicației? | Engleză + română de la început |
| Sursa datelor despre alimente? | Să decidă cercetarea |
| Echipă și termen? | Solo, fără deadline |

### Pasul 3: `PROJECT.md`
Am sintetizat totul într-un document viu. Conține ce e proiectul, valoarea centrală, cerințele active, ce e în afara scopului (cu motive), contextul, constrângerile și tabelul de decizii cheie.

### Pasul 4: setările de lucru
Vezi [secțiunea 6](#6-setările-de-lucru-configjson).

### Pasul 5: cercetarea
Am pornit **4 agenți de cercetare în paralel**, fiecare pe o dimensiune:

| Agent | Fișier | Despre ce |
|-------|--------|-----------|
| Stack | `research/STACK.md` | biblioteci și versiuni, **alegerea framework-ului mobil**, **sursa datelor despre alimente**, hosting |
| Features | `research/FEATURES.md` | ce au concurenții (MyFitnessPal, Cronometer, MacroFactor, Hevy, Strong, Fitbod), ce e obligatoriu, ce diferențiază, ce NU trebuie construit |
| Architecture | `research/ARCHITECTURE.md` | granițele modulelor, evenimente, modelarea datelor, coach, programarea joburilor, i18n, versionarea API |
| Pitfalls | `research/PITFALLS.md` | 30 de greșeli tipice în domeniu și cum le evităm |

Apoi un al 5-lea agent a făcut sinteza (`SUMMARY.md`).

> ⚠ **Incident:** agentul de sinteză (modelul „haiku”, cel mai ieftin) a returnat textul în loc să scrie fișierul. Era o problemă cunoscută, așa că am salvat eu fișierul. Am corectat și o eroare din el: scria că alternativa e „Flet”, iar cercetarea spunea **Flutter**.

### Pasul 6: cerințele
Ți-am prezentat funcționalitățile pe categorii, inclusiv cele propuse de cercetare (marcate cu ★). Tu ai ales ce intră în v1. A rezultat `REQUIREMENTS.md` cu **53 de cerințe v1**, plus o listă v2 și o listă „în afara scopului”.

### Pasul 7: structura și roadmap-ul
Ai ales **Vertical MVP**. Un agent „roadmapper” (modelul „opus”, cel mai puternic) a creat **7 faze**. Fiecare cerință e mapată la exact o fază (53 din 53). Ai aprobat roadmap-ul.

### Pasul 8: commit la cererea ta
Când ai cerut „commit the working tree changes”, am inspectat fișierele necomise ca să nu existe secrete. Am exclus prin `.gitignore` setările locale ale mașinii (`settings.local.json`, care conține calea ta spre Node) și am făcut commit-ul 3.

### Pasul 9: discuția pentru Faza 1 (pe pauză)
Vezi [secțiunea 9](#9-faza-1-ce-am-discutat-până-acum).

---

## 4. Deciziile de produs

Fiecare decizie are același format: ce am ales, de ce, ce alternative existau și ce se poate îmbunătăți.

### 4.1 Numele: FitAcademy
- **De ce:** l-ai ales tu. Documentul avea două nume: „FitAcademy” în titlu și „Health Companion” în text.
- **Alternative:** Health Companion, sau un nume nou.
- **De îmbunătățit:** înainte de o lansare publică, verifică disponibilitatea numelui în App Store și Google Play, a domeniului și a mărcii. „FitAcademy” e un nume destul de generic.

### 4.2 v1 = doar bucla de bază
- **Ce intră:** cont, profil și ținte, jurnal de mese, cod de bare, antrenamente, coach, admin minimal.
- **De ce:** valoarea centrală este „loghezi și primești sfaturi”. Socialul și gamificarea au sens doar dacă oamenii loghează deja constant. Cercetarea confirmă că **retenția** (dacă oamenii continuă să folosească aplicația) e riscul real în acest domeniu.
- **Alternative:** bucla de bază + gamificare și feed social; sau toate cele 7 zone din document, fiecare la nivel de bază (multă lățime, puțină profunzime).
- **De îmbunătățit:** după 2 săptămâni de folosire reală, decide pe baza datelor ce urmează. Cercetarea propune ca criteriu ca durata mediană de logare a unei mese să fie sub 30 de secunde.

### 4.3 Coach: card zilnic + review săptămânal
- **De ce:** cardul zilnic te împinge la acțiune („ieri ai avut 30 g proteine prea puțin; mâine adaugă X”). Review-ul săptămânal arată tendințe, care sunt mai stabile decât o zi izolată.
- **Alternative:** doar zilnic, sau doar săptămânal.
- **De îmbunătățit:** ora livrării cardului zilnic. Acum e „dimineața, despre ziua de ieri”. Ai putea vrea și un card „seara, pentru azi”.

### 4.4 Coach fără LLM în v1 (reguli + șabloane de text)
- **De ce:** documentul tău cere ca un LLM (model de limbaj, de tip ChatGPT sau Claude) să NU ia decizii de sănătate. Regulile deterministe sunt testabile, explicabile și reproductibile: aceleași date dau același sfat. Un LLM adaugă costuri, latență și riscul de „halucinații”.
- **Alternative:** LLM-ul Claude să reformuleze textul încă din v1; sau un alt furnizor de LLM.
- **De îmbunătățit:** arhitectura e pregătită pentru un LLM mai târziu. Insight-urile se salvează ca date structurate (regulă + parametri), iar LLM-ul ar putea doar să le reformuleze.

### 4.5 Admin web minimal (alimente + exerciții)
- **De ce:** fără admin, corectezi datele direct în baza de date, ceea ce e riscant. Gestionarea utilizatorilor nu e necesară cât timp ești doar tu cu câțiva prieteni.
- **Alternative:** fără admin (doar scripturi); sau admin complet (moderare, analytics).
- **De îmbunătățit:** jurnalul de audit al acțiunilor de admin e amânat pentru v2. Devine important când mai multe persoane au rol de admin.

### 4.6 Engleză + română de la început (i18n)
- **De ce:** adăugarea traducerilor mai târziu atinge fiecare ecran. Româna are 3 forme de plural („1 zi”, „2 zile”, „20 de zile”), deci sistemul trebuie gândit corect de la început.
- **Alternative:** doar engleză (mai simplu); doar română.
- **De îmbunătățit:** CI (verificarea automată la fiecare commit) va pica dacă lipsește o cheie de traducere. Astfel nicio limbă nu rămâne în urmă.

### 4.7 Cont: ce ai ales în v1
- **În v1:** resetare parolă, consimțământ pentru date de sănătate.
- **Amânat:** ștergerea contului din aplicație, exportul datelor.
- **De ce:** pentru „tu + prieteni” nu sunt strict necesare.
- ⚠ **De îmbunătățit:** **ștergerea contului din aplicație este obligatorie** în App Store și Google Play. E marcată în v2 ca **blocant de lansare**. Cercetarea spune că e ieftin de făcut devreme. Merită reconsiderat dacă vrei să ajungi repede în magazine.

### 4.8 Profil și ținte
- **În v1:** jurnal de greutate cu linie de tendință, ecranul „cum am calculat”, posibilitatea de a-ți seta țintele manual.
- **Calculul:** formula Mifflin-St Jeor dă metabolismul bazal. Se înmulțește cu factorul de activitate, apoi se ajustează după obiectiv (deficit, menținere, surplus). Proteinele se stabilesc primele, apoi grăsimile, iar carbohidrații completează restul.
- **Limite de siguranță în cod:** un minim de calorii, o limită a ritmului săptămânal de slăbire, și blocarea obiectivului de slăbit pentru minori sau pentru persoanele subponderale.
- **De ce:** ecranul „cum am calculat” este prima dovadă concretă de explicabilitate, adică diferențiatorul aplicației. Limitele protejează împotriva tulburărilor de alimentație.
- **De îmbunătățit:** valorile exacte (de exemplu minimum ~1200 kcal pentru femei și ~1500 pentru bărbați) sunt convenții. Trebuie alese cu surse citate, într-un ADR (vezi glosarul) în Faza 2.

### 4.9 Logare rapidă
- **În v1:** alimente recente și frecvente, quick-add de calorii, copierea unei mese sau a zilei de ieri.
- **De ce:** cercetarea arată că **viteza de logare decide dacă oamenii continuă** să folosească aplicația. Acestea sunt funcționalități de bază, nu finisaje.

### 4.10 Antrenamente
- **În v1:** performanța anterioară afișată în timp ce loghezi, seturi de încălzire și de lucru, cronometru de pauză, repetarea ultimului antrenament, grafice de progres (1RM estimat, recorduri, volum).
- **Adăugat de mine:** **WORK-07**, adică antrenamentul în curs supraviețuiește închiderii aplicației și semnalului slab. Nu te-am întrebat explicit, dar cercetarea îl consideră obligatoriu. În sală semnalul e adesea slab. Ți l-am arătat în lista finală și ai aprobat.

### 4.11 Coach: extrase în v1
- **În v1:**
  - „Poarta de completitudine”: nu dăm sfaturi pe baza unei zile logate pe jumătate. O zi în care ai logat doar o cafea ar arăta ca un deficit uriaș.
  - Mesaj de siguranță la aport foarte scăzut, în loc de laude pentru deficit.
- **Amânat:** istoricul insight-urilor, evaluarea cardurilor (util / nu e util).
- **De îmbunătățit:** evaluarea cardurilor ar fi o măsură directă pentru criteriul „cardurile sunt utile”. Ar putea fi mutată în v1 dacă vrei date concrete după perioada de folosire.

### 4.12 Ce e explicit în afara scopului (și de ce)
| Exclus | Motiv |
|--------|-------|
| Decizii de sănătate luate de un LLM | Siguranță: deciziile sunt doar reguli deterministe |
| „Mănânci înapoi” caloriile arse la sală | Estimările de calorii arse sunt foarte imprecise și strică explicabilitatea |
| Recunoaștere AI din poza mesei | Complexitate și precizie slabă |
| Planuri de masă automate | Non-obiectiv până la validare |
| Integrare cu brățări / ceasuri | Non-obiectiv în prima etapă |
| Micronutrienți | Datele crowdsourced sunt prea slabe |
| Diagnostic medical | Răspundere legală; ține aplicația în afara reglementării dispozitivelor medicale din UE |
| Login cu Google / Apple | Dacă oferi Google, Apple te obligă să oferi și „Sign in with Apple” |
| UI mobil în Python | Vezi 5.1 |

---

## 5. Deciziile tehnice

### 5.1 Aplicația mobilă: React Native + Expo (TypeScript)

**Ai cerut „ceva cu Python”.** Cercetarea a comparat:

| Opțiune | Avantaje | Dezavantaje |
|---------|----------|-------------|
| **React Native + Expo** ✅ | Scanare de coduri de bare inclusă (`expo-camera`); matur pentru App Store și Play; **aceeași limbă (TypeScript) ca adminul Next.js**, deci se pot partaja tipuri, validări și traduceri; build-uri în cloud și actualizări fără reinstalare (OTA) | Nu e Python |
| Flutter (locul 2) | UI excelent, i18n foarte bun, performanță | Limbaj nou (Dart), fără cod partajat cu adminul |
| Flet (Python) | E Python | Nu are decodare de coduri de bare inclusă; doar ~100 de pachete Python merg pe telefon |
| Kivy (Python) | E Python | Nicio versiune nouă din decembrie 2024 |
| BeeWare / Toga (Python) | E Python | Încă sub versiunea 1.0, imatur |

- **Concluzie:** Python rămâne acolo unde contează. Tot backendul (API, joburi în fundal, **motorul coach-ului**, importul de date despre alimente) e în Python. Telefonul e doar interfața.
- **De îmbunătățit:** Faza 1 începe cu un **test de o zi pe telefonul tău real**: scanezi un cod de bare cu Expo. Dacă nu merge bine, trecem pe Flutter înainte să scriem mult cod.

### 5.2 Datele despre alimente: un hibrid, cu sursa fiecărui rând

| Sursă | Pentru ce | Licență |
|-------|-----------|---------|
| **USDA FoodData Central** (Foundation + SR Legacy) | ~300–600 de alimente generice curate (piept de pui, orez…) | CC0, adică liberă total |
| **Open Food Facts (OFF)** | Coduri de bare: import în masă al produselor din România (~31.900, cifră neverificată), plus căutare live pentru restul | ODbL (vezi mai jos) |
| Alimentele tale / ale utilizatorilor | Ce lipsește | — |

- **De ce:** OFF are cea mai bună acoperire pentru coduri de bare europene și e gratuit. USDA are date generice de calitate. Nu există o bază deschisă de compoziție a alimentelor românești.
- **Alternative:**
  - Doar OFF: generic slab, iar datele crowdsourced au erori.
  - USDA Branded: are produse americane, inutile în România.
  - API-uri comerciale (FatSecret, Nutritionix, Edamam): costă și nu permit o bază de date locală.
  - CIQUAL (Franța): opțional.
- **Capcane:**
  - OFF permite doar **15 cereri pe minut** de pe același IP. Serverul face căutările, cu cache și limitare, iar telefonul nu apelează niciodată OFF direct.
  - Datele OFF au erori tipice: kJ în loc de kcal, valori pe 100 g amestecate cu valori pe porție.
  - Licența **ODbL** cere atribuire. Dacă publici baza derivată, poate cere s-o partajezi sub aceeași licență. De aceea fiecare rând are coloane `source`, `license` și `attribution`.
- **De îmbunătățit:** în Faza 4, scanează ~30 de produse într-un supermarket din România ca să măsori acoperirea reală. Înainte de lansarea publică, cere o verificare juridică a ODbL.

### 5.3 Backend: FastAPI ca „monolit modular”
- **Ce înseamnă:** o singură aplicație deployată, împărțită intern în module separate: auth, users, nutrition, barcode, workouts, coach.
- **Reguli de structură:**
  - Fiecare modul expune doar un fișier public (`public.py`).
  - Un instrument (**import-linter**) blochează în CI orice import „pe ușa din spate” între module.
  - Fiecare modul are propria schemă în PostgreSQL.
- **De ce:** era deja decis în documentul tău. Deploy-ul e simplu și granițele sunt clare. Coach-ul poate fi extras mai târziu ca serviciu separat.
- **Alternative:** microservicii (prea complex pentru o singură persoană); un monolit simplu, fără granițe (se degradează rapid).
- **De îmbunătățit:** riscul de supra-inginerie. Regula este să creezi un modul doar când îi construiești prima funcționalitate, nu schelete goale.

### 5.4 Evenimente între module: „outbox-lite”
- **Cum funcționează:** când loghezi o masă, evenimentul `MealLogged` se salvează **în aceeași tranzacție** cu masa, apoi e procesat imediat. Un worker reîncearcă ce a eșuat.
- **De ce:** nu se pierd evenimente, nici dacă serverul cade, și nu e nevoie de un broker separat (Kafka, RabbitMQ).
- **Alternative:** evenimente doar în memorie (se pot pierde); un broker de mesaje (prea mult pentru v1).

### 5.5 Joburi în fundal: Procrastinate (pe PostgreSQL) în loc de Redis
- **De ce:** joburile se pun în coadă în aceeași tranzacție cu datele. Are programare tip cron inclusă și istoric interogabil, și nu se pierd dacă Redis repornește.
- **Abatere de la document:** documentul tău spunea „Redis job queues”. Asta trebuie justificat într-un ADR în Faza 1.
- **Alternative:** Celery (prea greu), arq (doar întreținut, fără dezvoltare nouă), Dramatiq (fără cron), Taskiq + Redis (alternativa de rezervă).
- **De îmbunătățit:** combinația cu SQLAlchemy asincron nu e 100% verificată. E programat un spike (experiment scurt) de o zi în Faza 1.

### 5.6 Autentificare
- **Cum:** făcută de mână cu PyJWT + pwdlib (Argon2). Token de acces scurt (10–15 min) plus refresh token rotit, salvat ca hash, cu detectarea reutilizării.
- **De ce:** biblioteca `fastapi-users` e doar întreținută din octombrie 2025. Pentru portofoliu, o implementare proprie bine testată arată competență. Tutorialul oficial FastAPI folosește aceleași biblioteci.
- **Capcană cunoscută:** pe mobil, mai multe cereri pot expira simultan și declanșa „reutilizare de token”, ceea ce te deloghează aleatoriu. Soluția: un singur refresh odată, pe client, și o fereastră scurtă de grație pe server.
- **Alternative:** Auth0, Clerk sau Supabase Auth (rapide, dar dependență externă și mai puțin de arătat la interviu).

### 5.7 Modelarea datelor: decizii greu de schimbat ulterior
| Decizie | De ce | Faza |
|---------|-------|------|
| Fiecare masă logată păstrează o **copie a valorilor nutritive** | Dacă editezi alimentul, istoricul nu se rescrie | 3 |
| Fiecare intrare are `log_date` fixat la scriere, în fusul orar al utilizatorului | O masă la 23:30 nu „sare” în ziua următoare. Testăm cu 25 octombrie 2026, o zi de 25 de ore în România | 3 |
| Țintele se păstrează ca **istoric** (`effective_from`) | Zilele trecute se judecă după ținta valabilă atunci | 2 |
| Insight-urile se salvează **structurat** (regulă, versiune, parametri, dovezi), iar textul se generează la afișare | Se pot traduce, audita și reformula ulterior de un LLM | 6 |
| Valori numerice exacte (NUMERIC), nu float | Fără erori de rotunjire la sume | 3 |
| Repository-urile cer obligatoriu `user_id` | E structural imposibil să citești datele altcuiva | 1 |

### 5.8 Programarea coach-ului
- **Cum:** un job rulează la fiecare 15 minute și verifică pentru fiecare utilizator dacă e „dimineață la el”. Dacă da, generează cardul pentru ziua precedentă.
- **De ce:** funcționează corect pentru utilizatori din orice fus orar.

### 5.9 Hosting: Hetzner (VPS în UE) + Docker Compose + Caddy
- **Pe scurt:** un singur server, la ~5,5 €/lună. Caddy oferă HTTPS automat. Fișierele merg în Cloudflare R2, iar backup-urile se fac noaptea, pe alt server.
- **De ce:** datele de sănătate sunt protejate special de GDPR (articolul 9), așa că sunt găzduite în UE. E ieftin și mereu pornit.
- **Alternative:** Render, Railway sau Fly (managed, de 2–5 ori mai scump, dar fără administrare de server).
- **De îmbunătățit:** dacă administrarea serverului îți fură timp, mută-te pe o platformă managed.

### 5.10 Alte unelte
- **Python și backend:** Python 3.14, Pydantic 2.13, SQLAlchemy 2.1, Alembic (migrații), PostgreSQL 18.
- **Teste:** pytest, cu testcontainers, adică teste pe un PostgreSQL real, nu simulat. Plus hypothesis, care generează automat mii de cazuri pentru regulile coach-ului.
- **Calitatea codului:** Ruff, mypy strict.
- **Mobil:** TanStack Query, `expo-sqlite` (coadă locală când nu ai semnal), i18next.
- **Admin:** Next.js 16, Tailwind 4, shadcn.
- **NU folosim MinIO** (versiunea comunitară a fost arhivată), python-jose, passlib sau fastapi-users.
- **Notă:** versiunile au fost verificate pe 1 octombrie 2026 direct din registrele PyPI și npm.

---

## 6. Setările de lucru (`config.json`)

| Setare | Ales | De ce | Alternativă |
|--------|------|-------|-------------|
| Mod | **Interactive** | Confirmi la fiecare pas, potrivit când vrei să înțelegi tot | YOLO, adică aprobare automată |
| Granularitate | **Standard** (5–8 faze) | Echilibru între ansamblu și detaliu | Coarse (3–5) / Fine (8–12) |
| Execuție | **Paralelă** | Planurile independente rulează simultan | Secvențială |
| Git tracking | **Da** | Documentele de planificare apar în istoric, bine pentru interviu | Doar local |
| Research per fază | **Da** | Fiecare fază e cercetată înainte de planificare | Nu |
| Plan check | **Da** | Un agent verifică dacă planul chiar atinge scopul fazei | Nu |
| Verifier | **Da** | După execuție, se verifică că s-a livrat ce s-a promis | Nu |
| Compact content | **Nu** | Instrucțiuni complete, mai sigur | Da (mai puțin context) |
| Modele AI | **Adaptive** | Rolurile grele folosesc opus, cercetarea sonnet, sinteza haiku | Quality / Balanced / Budget / Inherit |
| Secțiuni PR | **User Stories, Risks** | Descrierile PR-urilor arată profesionist pentru portofoliu | Success Metrics, Stakeholder Review |

**De îmbunătățit:**
- „Adaptive” a ales **haiku** pentru sinteză, iar haiku a eșuat la scrierea fișierului. Pentru sinteze importante, profilul **Balanced** ar fi mai sigur.
- Răspunzi în română, dar întrebările au fost în engleză. Se poate seta limba răspunsurilor la română.

---

## 7. Cerințele v1 (53)

Lista completă e în `.planning/REQUIREMENTS.md`. Pe categorii:

| Categorie | ID-uri | Ce acoperă |
|-----------|--------|------------|
| Cont și autentificare | AUTH-01…06 | înregistrare, login persistent, logout, resetare parolă, consimțământ, izolarea datelor între utilizatori |
| Profil și ținte | PROF-01…08 | profil, obiectiv, ținte calculate, limite de siguranță, „cum am calculat”, ținte manuale, greutate cu tendință, istoricul țintelor |
| Nutriție | NUTR-01…12 | căutare EN/RO cu sau fără diacritice, logare, editare, sumar zilnic, istoric, alimente proprii, favorite, recente, quick-add, copiere, valori păstrate, ziua corectă |
| Cod de bare | BARC-01…05 | scanare, produse românești preîncărcate, căutare live, formular când produsul nu e găsit, atribuirea sursei |
| Antrenamente | WORK-01…10 | bibliotecă, exerciții proprii, seturi, încălzire și lucru, cronometru, performanța anterioară, rezistență la offline, istoric, progres, repetare |
| Coach | COACH-01…07 | card zilnic, review săptămânal, „de ce”, reguli versionate, completitudine, siguranță, text EN/RO neutru |
| Admin | ADMN-01…02 | alimente și exerciții, cu traduceri |
| Platformă | PLAT-01…03 | EN/RO, deploy în UE, backup-uri testate |

**v2 (amânat):** ștergerea contului (⚠ blocant pentru magazine), export de date, gestionarea utilizatorilor în admin și jurnal de audit, istoric și evaluare a insight-urilor, LLM, ajustarea automată a țintelor, rutine, rețete, notificări, social, gamificare, antrenori, portofel, recompense.

---

## 8. Roadmap-ul: 7 faze

**Ce înseamnă „Vertical MVP”:** fiecare fază livrează ceva **folosibil cap-coadă**: backend, aplicație mobilă și deploy. Alternativa, „straturi orizontale” (întâi toată baza de date, apoi tot API-ul, apoi tot UI-ul), îți dă ceva de folosit abia la final. Cu Vertical MVP începi să folosești aplicația devreme.

| # | Fază | Ce poți face la final | Cerințe |
|---|------|-----------------------|---------|
| 1 | Walking Skeleton & Secure Accounts | Instalezi aplicația pe telefon, îți faci cont pe serverul din UE și rămâi logat, în EN sau RO | AUTH-01…06, PLAT-01…03 |
| 2 | Profile, Goals & Safe Targets | Primești ținte sigure și explicate și îți urmărești greutatea | PROF-01…08 |
| 3 | Food Search & Daily Meal Log | Loghezi tot ce mănânci și vezi ziua față de ținte (**de aici începi să folosești aplicația zilnic**) | NUTR-01…05, 11, 12 |
| 4 | Fast Logging & Barcode Scanning | Loghezi o masă în câteva secunde | NUTR-06…10, BARC-01…05 |
| 5 | Workout Logging & Progress | Loghezi un antrenament complet, chiar cu semnal slab | WORK-01…10 |
| 6 | Explainable AI Coach | Primești card zilnic și review săptămânal, cu explicații | COACH-01…07 |
| 7 | Web Admin Curation | Gestionezi alimentele și exercițiile din browser | ADMN-01…02 |

**Dependențe:**
```
1 → 2 → 3 → 4
      ↘
        5  (poate merge în paralel cu 3–4)
3 + 5 → 6
4 + 5 → 7  (poate merge în paralel cu 6)
```

**Ce a schimbat roadmapper-ul față de cercetare (care propunea 9 faze):**
- Fundația și autentificarea au devenit o singură fază, ca să ai aplicația pe telefon încă din Faza 1.
- Logarea meselor a fost împărțită în „de bază” (Faza 3) și „rapidă + cod de bare” (Faza 4).
- Faza „pregătire pentru magazine” a dispărut. Cerința de 2 săptămâni de folosire devine o verificare la auditul final.

**De îmbunătățit:**
- Faza 7 are doar 2 cerințe. Se poate comasa: adminul de alimente în Faza 4, cel de exerciții în Faza 5, deci 6 faze în loc de 7.
- Faza 1 e cea mai încărcată: setup, deploy, autentificare, i18n, backup-uri și două spike-uri. Poate fi nevoie s-o împărțim (de exemplu 1 și 1.1).

---

## 9. Faza 1: ce am discutat până acum

Discuția are două zone. Prima e gata, a doua e pe pauză.

**Înregistrare și consimțământ** ✅
| Întrebare | Decizie | De ce | Alternative |
|-----------|---------|-------|-------------|
| Ce câmpuri are formularul de înregistrare? | Doar email + parolă | Fricțiune minimă; restul datelor vin în onboarding (Faza 2) | + nume; + data nașterii |
| Verificarea emailului? | Se cere, dar nu blochează aplicația; resetarea parolei merge doar la adrese verificate | Intri imediat, dar o greșeală de tastare la email e prinsă | Blocare până la verificare; fără verificare |
| Cum arată consimțământul? | Ecran separat după înregistrare, cu explicație simplă, link spre politica de confidențialitate și o bifă nebifată | GDPR cere consimțământ separat pentru datele de sănătate, nu „la pachet” cu termenii | Bifă pe formular |
| Dacă refuzi sau retragi consimțământul? | Contul rămâne, dar logarea și coach-ul sunt blocate, cu explicație și un buton „dau consimțământul”; datele existente se păstrează | Clar și reversibil | Blocarea totală a aplicației; ștergerea datelor de sănătate |

**Sesiuni și parole** ⏸ (nediscutat)
- Rămân de stabilit: cât timp rămâi logat, dacă poți fi logat pe mai multe dispozitive, regulile pentru parolă și dacă link-ul de resetare se deschide în aplicație sau într-o pagină web.

**Neselectate pentru discuție:** telefonul (iOS sau Android) și domeniul / furnizorul de email. ⚠ **Platforma telefonului trebuie totuși aflată** înainte de Faza 1, pentru testul de scanare și pentru taxa Apple de dezvoltator.

Deciziile sunt salvate în `.planning/phases/01-walking-skeleton-secure-accounts/01-DISCUSS-CHECKPOINT.json`. Când rulezi `/gsd-discuss-phase 1`, ți se oferă să continui de unde ai rămas.

---

## 10. Riscuri, întrebări deschise, ce se poate îmbunătăți

### Riscuri de produs
1. **Retenția.** Dacă logarea durează mult, nimeni nu continuă, nici tu. Măsoară de la început durata de logare a unei mese.
2. **Siguranța sfaturilor.** Ținte prea mici pot încuraja tulburări de alimentație. Limitele stau în cod, iar formulările trebuie revizuite.
3. **Calitatea datelor despre alimente.** OFF e crowdsourced, de unde nevoia de validator, ierarhie de încredere și set curat de alimente românești.

### Riscuri tehnice
4. **Testul de scanare pe telefon** poate schimba alegerea framework-ului mobil.
5. **Procrastinate + SQLAlchemy asincron** încă nu e verificat (spike în Faza 1).
6. **Supra-ingineria.** E mult „design de producție” pentru o singură persoană. Antidotul: module create doar când e nevoie de ele, cu granițe impuse automat.

### Riscuri legale (de verificat înainte de lansarea publică, nu acum)
7. **Licența ODbL** pentru datele OFF.
8. **GDPR:** consimțământ, date de sănătate, găzduire în UE.
9. **Magazinele de aplicații:** ștergerea contului, declarația Google Play pentru aplicații de sănătate, taxele.
10. Vârsta minimă de consimțământ digital în România (16 ani, de verificat).

### Întrebări deschise
- Telefonul tău e **iOS sau Android**?
- Ce **seturi de date deschise de exerciții** folosim și cu ce licență (free-exercise-db, wger)?
- Ce **valori exacte** au limitele de calorii și din ce surse?
- Publicăm sau nu subsetul derivat din OFF la lansarea publică?

### Ce aș face diferit sau ce se poate îmbunătăți în procesul de până acum
| Problemă | Detalii | Propunere |
|----------|---------|-----------|
| `.claude/` era în git | 795 de fișiere și ~15 MB de unelte GSD | ✅ Rezolvat: scos din istoric, ignorat |
| `.planning/research/.cache/` era în git | 13 fișiere de cache | ✅ Rezolvat: scos din istoric, ignorat |
| Sinteza a eșuat cu haiku | Am reparat manual | Profilul Balanced pentru sinteze importante |
| Limba | Întrebările au fost în engleză | Setăm limba răspunsurilor la română |
| WORK-07 adăugat fără întrebare directă | L-am semnalat, iar tu ai aprobat lista | — |
| Încrederea în cercetare | Multe informații vin din căutări web, iar unele sunt marcate „VERIFY-LATER” | Se reverifică în cercetarea fiecărei faze |

---

## 11. Structura fișierelor

```
/Users/macbookpro/Claude/
├── .planning/                         ← „creierul” proiectului (documente GSD)
│   ├── PROJECT.md                     ← viziune, valoare, scop, constrângeri, decizii cheie
│   ├── REQUIREMENTS.md                ← 53 de cerințe v1 + v2 + în afara scopului + urmărire
│   ├── ROADMAP.md                     ← 7 faze cu obiective, criterii de succes, note
│   ├── STATE.md                       ← unde am rămas (memoria între sesiuni)
│   ├── config.json                    ← setările de lucru
│   ├── research/
│   │   ├── SUMMARY.md                 ← START AICI: sinteza cercetării
│   │   ├── STACK.md                   ← tehnologii + decizia mobil + decizia alimente
│   │   ├── FEATURES.md                ← concurență, funcționalități
│   │   ├── ARCHITECTURE.md            ← cum se leagă componentele
│   │   └── PITFALLS.md                ← 30 de capcane
│   └── phases/01-walking-skeleton-secure-accounts/
│       └── 01-DISCUSS-CHECKPOINT.json ← deciziile Fazei 1 de până acum
├── docs/
│   └── REZUMAT-PROIECT.md             ← acest document
├── README.md                          ← prezentarea pentru GitHub (în engleză)
├── .claude/                           ← uneltele GSD + CLAUDE.md (instrucțiuni pentru Claude)
├── .mcp.json                          ← configurare MCP (claude-eyes, pentru testare în browser)
└── .gitignore
```

**Ordinea recomandată de citire:** acest document → `README.md` → `.planning/PROJECT.md` → `.planning/ROADMAP.md` → `.planning/research/SUMMARY.md`.

Codul va apărea, conform documentului tău, în: `apps/api` (FastAPI), `apps/mobile` (Expo), `apps/web-admin` (Next.js), `workers/`, `infra/` și `docs/adr/`.

---

## 12. Cum îl pui pe GitHub ca să arate bine la interviu

### Unde e proiectul acum
Pe Mac-ul tău, în `/Users/macbookpro/Claude`. E deja un repo git cu 10 commit-uri și nu trebuie „descărcat”. Dacă vrei o arhivă, poți crea un zip doar cu fișierele urmărite de git:
```bash
git -C /Users/macbookpro/Claude archive --format=zip -o ~/Desktop/fitacademy.zip HEAD
```

### Ce face un repo să arate bine la interviu
1. **Un README clar:** ce e proiectul, arhitectura, deciziile, cum îl rulezi. ✅ L-am creat.
2. **Istoric curat, pe etape,** cu mesaje convenționale (`docs:`, `feat:`, `fix:`, `chore:`). ✅ Deja așa, cu excepția uneltelor GSD.
3. **Fără zgomot:** fără unelte, cache-uri sau fișiere generate. ⚠ De curățat (vezi mai jos).
4. **Etichete (tags) pe etape,** de exemplu `v0.1-planning`, `phase-1`, `phase-2`. Pe GitHub poți arăta „uite cum arăta proiectul după fiecare fază”.
5. **Câte un Pull Request per fază,** cu descriere (GSD le generează cu `/gsd-ship`) și CI verde. Arată profesionist.
6. **ADR-uri în `docs/adr/`:** fiecare decizie importantă, cu motivul ei. Intervievatorii apreciază mult asta.

### Planul propus
1. ✅ **Curățare făcută:** istoricul nu mai conține `.claude/` și `.cache/`. Uneltele rămân pe disc, ignorate de git. Repo-ul are acum doar 14 fișiere urmărite, toate relevante.
2. ✅ **Tag `v0.1-planning`** pus pe starea curentă: „planificare completă”.
3. **Creare repo GitHub și push.** Trebuie să te autentifici tu (`gh auth login`), apoi:
   ```bash
   gh repo create fitacademy --private --source=/Users/macbookpro/Claude --push
   ```
   Recomand **privat** până ai cod funcțional, apoi îl faci public.
4. **Pentru fiecare fază:** un branch `phase-N-...`, commit-uri atomice (GSD le face automat), un PR spre `main`, merge, apoi tag-ul `phase-N`.

---

## 13. Glosar

| Termen | Explicație |
|--------|------------|
| **GSD** | Setul de comenzi care ghidează proiectul: întrebări, cercetare, cerințe, roadmap, discuție, plan, execuție, verificare |
| **Greenfield** | Proiect pornit de la zero, fără cod existent |
| **MVP** | Minimum Viable Product: cea mai mică versiune care livrează valoare reală |
| **Vertical slice** | O funcționalitate construită complet, de la baza de date până la ecran |
| **Walking skeleton** | Cea mai mică versiune cap-coadă care rulează în producție, pe care se construiește restul |
| **Monolit modular** | O singură aplicație, împărțită intern în module cu granițe stricte |
| **Outbox** | Evenimentele se salvează în aceeași tranzacție cu datele, ca să nu se piardă |
| **ADR** | Architecture Decision Record: un document scurt cu „ce am decis, de ce, ce alternative existau” |
| **Spike** | Un experiment scurt (de exemplu o zi) care răspunde la o întrebare tehnică înainte de a construi |
| **CI** | Continuous Integration: teste și verificări rulate automat la fiecare commit sau PR |
| **JWT / refresh token** | Token scurt pentru acces, plus token lung pentru reînnoire, rotit la fiecare folosire |
| **RBAC** | Role-Based Access Control: drepturi pe roluri (user, trainer, admin) |
| **i18n** | Internaționalizare: suport pentru mai multe limbi |
| **IANA timezone** | Numele standard al unui fus orar, de exemplu `Europe/Bucharest` |
| **DST** | Ora de vară. Schimbarea creează zile de 23 sau 25 de ore |
| **BMR / TDEE** | Metabolismul bazal / cheltuiala zilnică totală de energie |
| **Mifflin-St Jeor** | Formula standard pentru BMR |
| **1RM** | One-Rep Max: greutatea maximă pe care o poți ridica o singură dată |
| **OFF** | Open Food Facts: bază de date deschisă de produse alimentare |
| **USDA FDC** | Baza de date nutrițională a Departamentului Agriculturii din SUA |
| **ODbL** | Licența bazei OFF: cere atribuire și, la publicarea unei baze derivate, partajarea ei sub aceeași licență |
| **GDPR art. 9** | Datele de sănătate sunt „categorie specială”: cer consimțământ explicit |
| **Dogfooding** | Îți folosești propriul produs zilnic ca să-l validezi |
| **Expo / EAS** | Platforma care simplifică dezvoltarea React Native și build-urile pentru magazine |
| **LLM** | Large Language Model, de exemplu Claude sau GPT |

---

*Generat: 1 octombrie 2026. Starea: planificare completă, Faza 1 în discuție (pe pauză).*
