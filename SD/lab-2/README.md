[Uploading README.md…]()

# Laboratorul 2: Cuantifică citirile Dashboard-ului

Radu · 13 octombrie 2026

Dashboard-ul trebuie să suporte o țintă de **103 / 1,030 / 10,296 RPS** la deschiderea pieței pentru 300 / 3,000 / 30,000 de Utilizatori concurenți; citirile Stock price produc aproximativ 76% din traficul stabil.

## Ipoteze generale

Punctul de pornire este analiza din Laboratorul 1: Utilizatorul autentificat este actorul uman direct, Market Data Provider este sistemul extern, iar prima versiune oferă Overview, Filter, Search, Stock detail, price history și un Watchlist privat.

- Fiecare acțiune enumerată a Utilizatorului creează exact o cerere către Dashboard.
- O cerere către Dashboard **nu** creează automat o cerere către furnizor; traficul către furnizor nu este estimat aici.
- Procentele sunt ponderi medii ale Utilizatorilor concurenți în intervalul ales; toate fluxurile pot avea loc simultan.
- Reîncercările, activitatea din fundal și octeții răspunsurilor sunt excluși.
- RPS = cereri pe secundă. Rezultatele intermediare nu sunt rotunjite; se rotunjește în sus numai ținta finală de capacitate.
- Stocarea folosește octeți zecimali (1 MB = 10⁶ B, 1 GB = 10⁹ B), cu echivalentul binar (MiB, GiB) acolo unde ajută.
- Fusul orar al pieței este ET (America/New_York), deoarece domeniul ales în secțiunea 4 este piața din SUA.

## 1. Cerințe de calitate

Fiecare cerință are forma `măsură + țintă + condiție de operare`. Un rezultat indisponibil este numărat separat de un rezultat cu date corecte: un „Indisponibil” rapid poate respecta latența, dar consumă bugetul de disponibilitate.

### 1.1 Latența citirilor

**Începutul măsurării:** browserul Utilizatorului trimite cererea (latență observată de Client, măsurată prin monitorizare în browser).

**Sfârșitul măsurării:** rezultatul corect este afișat și utilizabil, inclusiv ora furnizorului sau eticheta de întârziere pentru prețuri.

**Condiția de operare:** în orele de tranzacționare, la cel mult ținta RPS de la deschiderea pieței pentru nivelul ales (103 / 1,030 / 10,296 RPS, secțiunea 3), măsurat zilnic pe fiecare tip de citire.

| Citire | p50 | p95 | Limită (timeout) | Motiv |
| --- | --- | --- | --- | --- |
| Stock price | ≤ 200 ms | ≤ 500 ms | 2 s | Clientul cere ca prețul să „pară imediat”; este citit o dată pe secundă |
| Overview | ≤ 500 ms | ≤ 2 s | 2 s | Pagina principală, reîmprospătată la 30 s |
| Filter | ≤ 500 ms | ≤ 2 s | 2 s | Interacțiune repetată (3 pe minut) |
| Watchlist | ≤ 500 ms | ≤ 2 s | 2 s | Lista personală, reîmprospătată la 1 min |
| Search | ≤ 500 ms | ≤ 2 s | 2 s | Utilizatorul așteaptă activ rezultatul |
| History | ≤ 1 s | ≤ 2 s | 2 s | Mai multe date (grafic), cerere rară |

Cerința principală, scrisă complet:

> În orele de tranzacționare, la cel mult ținta RPS de deschidere a pieței, măsurată de la trimiterea cererii de către browser până când prețul corect și ora furnizorului sunt afișate, latența citirii Stock price trebuie să fie p50 ≤ 200 ms și p95 ≤ 500 ms.

„Majoritatea citirilor” din cerința clientului este interpretată ca **p95**: 95 din 100 de citiri corecte se termină în limita aleasă.

**Ce se întâmplă la 2 secunde:**

1. Browserul oprește așteptarea și afișează explicit „Indisponibil — reîncearcă”. O valoare afișată anterior poate rămâne numai cu ora furnizorului și cu eticheta de vechime, niciodată ca preț curent.
2. Serverul anulează lucrul pentru acea cerere, ca să nu consume capacitate pentru un răspuns pe care nimeni nu îl mai așteaptă.
3. Încercarea este numărată ca citire peste 2 s în distribuția de latență și ca încercare inacceptabilă pentru disponibilitate și debit.

### 1.2 Disponibilitate bazată pe timpul de funcționare

**Când este Dashboard-ul utilizabil.** Un minut este utilizabil dacă cel puțin 99% dintre încercările valide de citire (toate cele șase citiri) din acel minut returnează un rezultat acceptabil în cel mult 2 s. Rezultat acceptabil = date corecte; pentru Stock price, în orele de tranzacționare, prețul trebuie să fie în limita de vechime de la 1.3. În minutele fără trafic real se folosește o sondă sintetică la 10 s (Overview, un Stock price de referință, Watchlist-ul unui Utilizator de test).

**Intervalul orelor de tranzacționare:** zilele de tranzacționare NYSE/Nasdaq, **09:15–16:15 ET** (16:15–23:15 ora Moldovei, vară). Sesiunea regulată este 09:30–16:00 ET; adăugăm 15 minute înainte (Utilizatorii se pregătesc pentru deschidere) și 15 minute după (cu întârzierea de ≈15 min a furnizorului, ultimele prețuri ale sesiunii sosesc până la ≈16:15). **Restul zilei** = tot timpul rămas, inclusiv weekendurile și sărbătorile bursiere.

**Țintele:**

- Ore de tranzacționare: **99.9%** (trei nouari).
- Restul zilei: **99%** (doi nouari).

**De ce ținte diferite.** În orele de tranzacționare prețurile se schimbă, apare vârful de la deschidere, iar valoarea pentru Utilizator este cea mai mare. În afara lor, prețul afișat este prețul de închidere, care nu se schimbă, traficul este mai mic, iar acest interval poate găzdui mentenanța planificată. Nu alegem 99.99%: Dashboard-ul afișează date întârziate cu ≈15 min și nu execută tranzacții, deci câteva minute de indisponibilitate nu justifică costul unui buget de 10 ori mai mic (argument ROI din Lecția 3).

**Bugetul de indisponibilitate pe 30 de zile** (măsurat separat pentru fiecare interval; presupunem 21 de zile de tranzacționare în 30 de zile calendaristice):

| Interval | Timp măsurat în 30 zile | Țintă | Calcul | Buget de indisponibilitate |
| --- | --- | --- | --- | --- |
| Ore de tranzacționare | 21 zile × 7 h = 147 h = 8,820 min | 99.9% | 8,820 min × 0.001 | **8.82 min = 8 min 49 s** |
| Restul zilei | 720 h − 147 h = 573 h = 34,380 min | 99% | 34,380 min × 0.01 | **343.8 min = 5 h 43 min 48 s** |

Cu doar 20 de zile de tranzacționare (o sărbătoare în interval), bugetul pentru orele de tranzacționare devine 140 h × 60 × 0.001 = 8.4 min. Procentul nu spune dacă bugetul este consumat de o singură întrerupere sau de mai multe.

### 1.3 Consistență

**Stock price — actualitate (vechime limitată).** Vechimea = momentul răspunsului − ora furnizorului pentru valoarea respectivă (ora la care furnizorul declară valabil prețul, nu ora ultimei tranzacții).

> În orele de tranzacționare, o citire Stock price returnează ultimul preț acceptat de la furnizor, cu ora furnizorului și eticheta „Întârziat ≈15 min”, numai dacă vechimea este **≤ 20 min**. Dacă nu există un preț acceptat sau vechimea depășește 20 min, citirea returnează **Indisponibil**: nu afișează zero și nu prezintă o valoare veche drept curentă.

- 20 min = întârzierea așteptată de 15 min + 5 min toleranță pentru sincronizare și variația furnizorului.
- În afara orelor de tranzacționare se afișează prețul oficial de închidere, etichetat „Închidere, `<dată>`”, valabil până la 20 min după deschiderea sesiunii următoare.
- Citire monotonă: pentru același Stock, un Utilizator nu vede niciodată o oră a furnizorului mai veche decât una pe care a văzut-o deja.
- Aceeași regulă se aplică prețurilor din Overview, Filter și Watchlist.

**Watchlist — citirea propriilor scrieri.**

> După ce Dashboard-ul confirmă Utilizatorului adăugarea sau eliminarea unui Stock, următoarea citire a Watchlist-ului făcută de același Utilizator (din orice sesiune sau dispozitiv) include modificarea, iar citirile ulterioare nu mai arată lista de dinainte. Dacă această garanție nu poate fi respectată, citirea returnează **Watchlist indisponibil**, nu o listă veche.

- O modificare neconfirmată (de ex. timeout) este afișată „Neconfirmată”, nu ca salvată; următoarea citire arată starea reală.
- Un Utilizator nu poate citi sau modifica Watchlist-ul altui Utilizator; o asemenea cerere este refuzată, nu returnează date.

### 1.4 Debit

> În fereastra de deschidere a pieței și în minutele următoare, Dashboard-ul finalizează cel puțin **103 / 1,030 / 10,296 citiri acceptabile pe secundă** (nivelul ales), în timp ce țintele de latență, disponibilitate și consistență continuă să fie respectate. În regim stabil, ținta este 80 / 792 / 7,920 citiri acceptabile pe secundă.

O citire care depășește 2 s, returnează Indisponibil sau arată un preț peste limita de vechime nu este numărată ca citire acceptabilă finalizată.

## 2. Estimări RPS în regim stabil

Traficul stabil este **79.2 / 792 / 7,920 RPS**, adică 0.264 RPS pentru fiecare Utilizator concurent; Stock price produce singur 75.8% din el.

Formula: `RPS = Utilizatori concurenți × pondere participantă × acțiuni per Utilizator / secunde`. Unitatea: Utilizatori × (acțiuni / Utilizator) / s = cereri / s.

Rata per Utilizator concurent (factorul care se înmulțește cu 300, 3,000 sau 30,000):

| Citire | Comportament | Calcul pe Utilizator | RPS / Utilizator |
| --- | --- | --- | --- |
| Overview | 70%, 1 reîmprospătare / 30 s | 0.70 × 1 / 30 s | 0.023333… |
| Filter | 50%, 3 modificări / 60 s | 0.50 × 3 / 60 s | 0.025 |
| Stock price | 20%, 1 cerere / 1 s | 0.20 × 1 / 1 s | 0.2 |
| History | 20%, 1 cerere / 300 s | 0.20 × 1 / 300 s | 0.000666… |
| Watchlist | 60%, 1 reîmprospătare / 60 s | 0.60 × 1 / 60 s | 0.01 |
| Search | 10%, 3 căutări / 60 s | 0.10 × 3 / 60 s | 0.005 |
| **Total** |  |  | **0.264** |

Tabelul cerut (rezultate nerotunjite, în RPS):

| Citire | 300 Utilizatori | 3,000 Utilizatori | 30,000 Utilizatori |
| --- | --- | --- | --- |
| Overview | 300 × 0.70 × 1 / 30 = **7** | 3,000 × 0.70 × 1 / 30 = **70** | 30,000 × 0.70 × 1 / 30 = **700** |
| Filter | 300 × 0.50 × 3 / 60 = **7.5** | 3,000 × 0.50 × 3 / 60 = **75** | 30,000 × 0.50 × 3 / 60 = **750** |
| Stock price | 300 × 0.20 × 1 / 1 = **60** | 3,000 × 0.20 × 1 / 1 = **600** | 30,000 × 0.20 × 1 / 1 = **6,000** |
| History | 300 × 0.20 × 1 / 300 = **0.2** | 3,000 × 0.20 × 1 / 300 = **2** | 30,000 × 0.20 × 1 / 300 = **20** |
| Watchlist | 300 × 0.60 × 1 / 60 = **3** | 3,000 × 0.60 × 1 / 60 = **30** | 30,000 × 0.60 × 1 / 60 = **300** |
| Search | 300 × 0.10 × 3 / 60 = **1.5** | 3,000 × 0.10 × 3 / 60 = **15** | 30,000 × 0.10 × 3 / 60 = **150** |
| **Total în regim stabil** | 7 + 7.5 + 60 + 0.2 + 3 + 1.5 = **79.2** | 70 + 75 + 600 + 2 + 30 + 15 = **792** | 700 + 750 + 6,000 + 20 + 300 + 150 = **7,920** |

Distribuția pe citiri este aceeași la toate nivelurile: Stock price 75.8% (0.2 / 0.264), Filter 9.5%, Overview 8.8%, Watchlist 3.8%, Search 1.9%, History 0.25%. Modelul este liniar: fiecare nivel este exact de 10 ori precedentul, deci trecerea de la 3,000 la 30,000 este chiar testul de sensibilitate la 10x. Acest rezultat nu dovedește că comportamentul Utilizatorilor, datele sau capacitatea unei instanțe rămân aceleași la 10x.

Dacă este nevoie de o țintă de capacitate în regim stabil, rotunjirea în sus dă 80 / 792 / 7,920 RPS.

## 3. Estimări RPS la deschiderea pieței

Țintele de capacitate la deschiderea pieței sunt **103 / 1,030 / 10,296 RPS**, adică subtotalul este cu 18% peste traficul stabil, iar apoi se adaugă marja de 10%.

Comportamentul suplimentar se adaugă peste traficul stabil, care continuă. Traficul stabil pentru Overview și Watchlist rămâne o singură dată în rândul „Trafic stabil de citire”; rândurile următoare conțin numai cererile suplimentare.

- Flux suplimentar Overview: `N × 0.30 × 1 / 10 s` = 0.03 RPS per Utilizator.
- Flux suplimentar Watchlist: `N × 0.30 × 0.60 × 1 / 10 s` = 0.018 RPS per Utilizator (60% din grupul de 30%, adică 18% din toți Utilizatorii).
- Subtotal: 0.264 + 0.03 + 0.018 = 0.312 RPS per Utilizator; marja de 10% se aplică după subtotal.

| Calcul la deschiderea pieței | 300 Utilizatori | 3,000 Utilizatori | 30,000 Utilizatori |
| --- | --- | --- | --- |
| Trafic stabil de citire | 79.2 (secțiunea 2) | 792 (secțiunea 2) | 7,920 (secțiunea 2) |
| Flux suplimentar de reîmprospătare Overview | 300 × 0.30 / 10 = **9** | 3,000 × 0.30 / 10 = **90** | 30,000 × 0.30 / 10 = **900** |
| Flux suplimentar de reîmprospătare Watchlist | 300 × 0.30 × 0.60 / 10 = **5.4** | 3,000 × 0.30 × 0.60 / 10 = **54** | 30,000 × 0.30 × 0.60 / 10 = **540** |
| **Subtotal la deschiderea pieței** | 79.2 + 9 + 5.4 = **93.6** | 792 + 90 + 54 = **936** | 7,920 + 900 + 540 = **9,360** |
| Marjă de capacitate de 10% | 93.6 × 0.10 = **9.36** | 936 × 0.10 = **93.6** | 9,360 × 0.10 = **936** |
| **Țîntă rotunjită în sus la deschiderea pieței** | 93.6 + 9.36 = 102.96 → **103 RPS** | 936 + 93.6 = 1,029.6 → **1,030 RPS** | 9,360 + 936 = 10,296 → **10,296 RPS** (deja întreg) |

Numai ținta finală este rotunjită. La 30,000 de Utilizatori, cele două fluxuri suplimentare adaugă 1,440 RPS (+18% față de traficul stabil), dar Stock price rămâne cel mai mare rând (6,000 din 9,360 RPS, 64%).

Modelul presupune că reîmprospătările suplimentare sunt distribuite uniform în cele 10 secunde. Dacă mulți Utilizatori reîmprospătează exact la 09:30:00, vârful pe o secundă poate fi mai mare decât această medie; acest lucru este analizat la blocajul de debit (secțiunea 5).

## 4. Estimări de stocare

Pentru ≈5,600 de Stocks din SUA, stocarea brută a datelor de piață este **≈2.94 GB inițial** și **≈3.01 GB după un an**; aproape tot volumul vine din barele de 1 minut păstrate 30 de zile.

### 4.1 Ce este un Stock

Un „Stock” poate însemna lucruri diferite într-un produs de date de piață. SEC definește acțiunile ca titluri care dau deținătorului o parte din proprietatea unei companii și distinge două tipuri principale: common stock (cu drept de vot) și preferred stock (de obicei fără vot, dar cu prioritate la dividende) ([Investor.gov — Stocks FAQs](https://www.investor.gov/introduction-investing/investing-basics/investment-products/stocks)). Acțiunile majorității companiilor străine tranzacționate în SUA sunt ADR-uri, emise de bănci depozitare americane, al căror preț urmărește acțiunea din piața de origine ([Investor.gov — ADRs](https://www.investor.gov/introduction-investing/investing-basics/glossary/american-depositary-receipts-adrs)). Furnizorii mai pun sub același simbol și ETF-uri, fonduri, warrants, units sau rights.

**Definiția pentru acest Dashboard:** un Stock este o clasă de acțiuni comune (common stock) sau un ADR listat pe o bursă americană acceptată, identificat unic prin simbol + bursă (și ISIN). Fiecare clasă este un Stock separat (de ex. GOOGL și GOOG sunt două Stocks).

### 4.2 Decizii despre produs

| Decizie | Alegere | Justificare |
| --- | --- | --- |
| Piață / burse | SUA: Nasdaq, NYSE, NYSE American | O singură țară și un singur fus orar (ET) dă o singură fereastră de tranzacționare pentru cerința de disponibilitate; OTC este exclus (multe titluri ilichide, date de calitate inegală) |
| Instrumente incluse | Common stock (toate clasele), ADR-uri | Sunt acțiunile pe care Utilizatorii le caută și le urmăresc; ADR-urile acoperă companii mari străine (TSM, ASML, SAP) |
| Instrumente excluse | Preferred stock, ETF-uri, fonduri, warrants, rights, units | Au alt comportament de preț și alte filtre; pot fi adăugate într-o versiune ulterioară |
| Instrumente inactive / delistate | Rămân în date ca **inactive**: nu apar în Search și Filter, dar rămân vizibile în Watchlist-urile care le conțin („Delistat la `<dată>`”) și își păstrează istoricul zilnic | Un Utilizator nu trebuie să piardă în tăcere un element din Watchlist; fără preț curent, prețul este Indisponibil |
| Număr de Stocks | **5,602** (folosit ca ≈5,600) | Vezi mai jos |

**Numărul de Stocks**, observat pe 13 octombrie 2026 pe [StockAnalysis — lista burselor](https://stockanalysis.com/list/exchanges/):

| Bursă | Stocks listate | Sursă |
| --- | --- | --- |
| Nasdaq | 3,426 | [StockAnalysis — Nasdaq](https://stockanalysis.com/list/nasdaq-stocks/) |
| NYSE | 1,934 | [StockAnalysis — NYSE](https://stockanalysis.com/list/nyse-stocks/) |
| NYSE American | 242 | [StockAnalysis — lista burselor](https://stockanalysis.com/list/exchanges/) |
| **Total** | **5,602** |  |

Aceste liste exclud ETF-urile, dar includ câteva serii de preferred stock (de ex. GS.PRD) pe care Dashboard-ul le exclude. Folosim totalul de 5,602 ca limită superioară prudentă. Verificare încrucișată: [The Motley Fool](https://www.fool.com/investing/stock-market/companies/) (actualizat 15 septembrie 2026) dă ≈3,450 pe Nasdaq și ≈2,200 pe NYSE, inclusiv clasele multiple — același ordin de mărime.

### 4.3 Datele sincronizate

Fiecare set de date este legat de o nevoie din Laboratorul 1. Frecvențele de sincronizare sunt presupuneri; traficul către furnizor nu este estimat aici.

| Set de date | Nevoia produsului | Câmpuri stocate | Sincronizare (presupunere) |
| --- | --- | --- | --- |
| Date de referință Stock | Identitate în Stock detail, Search (simbol, nume), Filter (bursă, sector, industrie, capitalizare) | ID intern, ISIN, simbol, nume, nume normalizat pentru căutare, bursă (MIC), tip (common / ADR), monedă, țară, cod sector și industrie, număr de acțiuni în circulație, stare + date listare / delistare, ora actualizării | O dată pe zi, înainte de 09:15 ET |
| Cele mai recente prețuri | Stock price, Overview (creșteri / scăderi), Filter după preț și variație, Watchlist | Ultimul preț, închiderea anterioară (pentru variația %), deschidere / maxim / minim al zilei, volum, ora furnizorului, ora ultimei tranzacții, ora primirii la noi, indicatori (întârziat, suspendat) | La fiecare 15 s în orele de tranzacționare; o înregistrare per Stock, suprascrisă |
| Price history — intraday | Graficul „1 zi” și „5 zile” / „1 lună” din History | Bară de 1 minut OHLCV pentru sesiunea regulată (390 bare / zi) | La fiecare minut; **păstrare 30 de zile calendaristice** (≈21 zile de tranzacționare), apoi ștergere |
| Price history — zilnic | Graficele „1 an” și „5 ani” | Bară zilnică OHLCV | O dată pe zi după închidere; **5 ani încărcați inițial, păstrare nelimitată** |
| Alte date selectate | Corectarea istoricului după split-uri; calendarul pentru fereastra de tranzacționare și etichetele „Închis”; denumirile sectoarelor pentru Filter | Split-uri (Stock, dată, raport), calendar bursier (dată, ora deschiderii / închiderii, stare), dicționar sectoare / industrii | O dată pe zi |

Nu stocăm cotele bid / ask, tranzacțiile individuale, sesiunile pre-market / after-hours, știrile sau datele financiare: niciun flux din Laboratorul 1 nu are nevoie de ele.

### 4.4 Înregistrări reprezentative

Dimensiunile sunt pentru valori binare fixe (ID și prețuri ca întregi pe 8 octeți, de ex. preț × 10,000), fără overhead de rând, indexuri sau replici.

| Înregistrare | Exemplu | Câmpuri și octeți | Total |
| --- | --- | --- | --- |
| Referință Stock | AAPL, Apple Inc., XNAS, common, USD, US | ID 8 + ISIN 12 + simbol 6 (medie) + nume 33 (medie) + MIC 4 + tip 1 + monedă 3 + țară 2 + sector / industrie 4 + acțiuni în circulație 8 + stare și date 9 + nume normalizat 33 + actualizat la 8 | **131 B** |
| Cel mai recent preț | AAPL, 333.69, ora furnizorului 14:02:15 ET | ID 8 + ultimul preț 8 + închidere anterioară 8 + deschidere / maxim / minim 24 + volum 8 + ora furnizorului 8 + ora ultimei tranzacții 8 + ora primirii 8 + indicatori 1 | **81 B** |
| Bară de 1 minut | AAPL, 2026-10-13 10:31 ET | ID 8 + timestamp 8 + OHLC 4 × 8 = 32 + volum 8 | **56 B** |
| Bară zilnică | AAPL, 2026-10-12 | ID 8 + dată 4 + OHLC 32 + volum 8 | **52 B** |
| Split | AAPL, 2020-08-31, 4:1 | ID 8 + dată 4 + raport 8 + anunțat la 8 | 28 B |
| Zi din calendar | 2026-11-27, 09:30–13:00 ET, zi scurtă | dată 4 + deschidere 4 + închidere 4 + stare 1 | 13 B |

### 4.5 Calculul stocării (inițial)

`stocare brută = număr de înregistrări × octeți per înregistrare`; S = 5,602 Stocks; 252 zile de tranzacționare pe an.

| Set de date | Decizia despre produs și perioada de păstrare | Calculul numărului de înregistrări | Octeți per înregistrare | Stocare brută |
| --- | --- | --- | --- | --- |
| Date de referință Stock | Toate Stocks acceptate; delistatele rămân ca inactive | 5,602 | 131 B | 733,862 B ≈ **0.73 MB** |
| Cele mai recente prețuri | O înregistrare per Stock, suprascrisă la fiecare 15 s | 5,602 | 81 B | 453,762 B ≈ **0.45 MB** |
| Price history | Bare de 1 min păstrate 30 de zile + bare zilnice: 5 ani inițial, păstrare nelimitată | 1 min: 5,602 × 390 × 21 = 45,880,380; zilnic: 5,602 × 1 × 1,260 = 7,058,520 | 56 B / 52 B | 2,569,301,280 B + 367,043,040 B ≈ **2.94 GB** |
| Alte date selectate | Split-uri pe 5 ani (≈1,500, presupunere), calendar pe 2 ani (730 zile), dicționar de 161 sectoare / industrii | 1,500 + 730 + 161 | 28 B / 13 B / 40 B | 57,930 B ≈ **0.06 MB** |
| **Total** |  |  |  | **2,937,589,874 B ≈ 2.94 GB (2.74 GiB)** |

### 4.6 Zilnic și după un an

| Interval | Calcul | Rezultat |
| --- | --- | --- |
| Scris într-o zi de tranzacționare | bare 1 min: 5,602 × 390 × 56 B = 122,347,680 B; bare zilnice: 5,602 × 52 B = 291,304 B; Stocks noi și alte date: ≈200 B | **≈ 122.6 MB / zi** |
| Creștere netă zilnică | Barele de 1 minut mai vechi de 30 de zile sunt șterse, deci rămân doar barele zilnice + Stocks noi | **≈ 0.29 MB / zi** |
| Scris într-un an | 122.35 MB × 252 zile ≈ 30.8 GB bare de 1 minut (cele mai multe deja șterse) | **≈ 30.9 GB scriși / an** |
| Stocat după un an | referință (5,602 + 300 Stocks noi presupuse) × 131 B = 0.77 MB + prețuri 0.45 MB + bare 1 min 2.57 GB (constant) + bare zilnice 5,602 × 1,512 × 52 B = 0.44 GB + alte date 0.07 MB | **3,011,049,458 B ≈ 3.01 GB (2.80 GiB)** |

Tabelul cu cele mai recente prețuri nu crește, dar este suprascris de aproximativ 5,602 × (7 h × 3,600 s / 15 s) ≈ 9.4 milioane de ori pe zi. Estimarea exclude indexurile, replicile, jurnalele, copiile de siguranță și datele Utilizatorilor (conturi, Watchlist-uri).

## 5. Posibile blocaje

Cel mai probabil prim blocaj de latență și debit este calea de citire Stock price (6,000 din 7,920 RPS stabil la 30,000 Utilizatori), iar cel mai mare risc de disponibilitate și consistență este sincronizarea cu Market Data Provider, care are doar 5 minute de toleranță peste întârzierea de 15 minute. Toate rândurile sunt ipoteze de măsurat, nu cauze dovedite.

| Calitate | Posibil blocaj | Dovezi din acest laborator | Efect posibil | Ce trebuie măsurat în continuare |
| --- | --- | --- | --- | --- |
| Latență | Calea de citire Stock price | 75.8% din traficul stabil (60 / 600 / 6,000 RPS); cea mai strictă țintă (p95 ≤ 500 ms); 20% dintre Utilizatori o cer în fiecare secundă | p95 al Stock price depășește 500 ms sub încărcare, iar Utilizatorul nu mai percepe prețul ca „imediat” | p50 / p95 / p99 pentru Stock price într-un test cu rată fixă de sosire la 792 și 7,920 RPS, cu timpul descompus pe rețea, server și accesul la date |
| Consistență | Sincronizarea celor mai recente prețuri de la furnizor | Întârzierea furnizorului este ≈15 min, limita de vechime este 20 min, deci rămân doar 5 min de toleranță; prețurile sunt suprascrise ≈9.4 milioane de ori pe zi | O întârziere de sincronizare de peste 5 min face prețurile Indisponibile sau, dacă eticheta lipsește, arată un preț vechi drept curent | Distribuția vechimii (momentul răspunsului − ora furnizorului) pentru fiecare citire; întârzierea sincronizării per lot; numărul de citiri care încalcă monotonia sau regula de 20 min |
| Debit | Vârful de la deschiderea pieței (Overview + Watchlist) | +1,440 RPS la 30,000 Utilizatori (+18%); modelul presupune cereri uniforme în 10 s, dar Utilizatorii pot reîmprospăta exact la 09:30:00; Overview calculează clasamente pe ≈5,600 Stocks | Pe o singură secundă, rata reală poate fi mult peste media de 10,296 RPS; citirile acceptabile finalizate scad sub țintă, iar timeout-urile cresc | Histograma sosirilor pe secundă între 09:29 și 09:31 ET; citirile acceptabile finalizate pe secundă; costul unei citiri Overview față de o citire Stock price |
| Disponibilitate | Dependența externă Market Data Provider | Bugetul pentru orele de tranzacționare este doar 8 min 49 s în 30 de zile; Stock price, Overview, Filter și Watchlist depind toate de prețuri actuale; o întrerupere a furnizorului face prețurile Indisponibile după 5 min | O singură întrerupere de ≈14 min a furnizorului (5 min toleranță + 9 min) consumă tot bugetul lunar al orelor de tranzacționare | Istoricul de disponibilitate al furnizorului (durata și frecvența întreruperilor, SLA-ul lui); minutele utilizabile măsurate de sonda sintetică separat pentru cele două intervale |

**Explicații pe rânduri:**

1. **Latență — Stock price.** Presiunea vine din calea de citire cu cel mai mare volum și cea mai strictă țintă. Dacă fiecare cerere repetă aceeași muncă (căutare de date, serializare), timpul crește odată cu încărcarea și p95 depășește 500 ms. Dovada este ponderea de 75.8% și faptul că 6,000 de Utilizatori (la nivelul de 30,000) cer același tip de valoare în fiecare secundă, deși prețul se schimbă doar la 15 s. Un test de încărcare la 7,920 RPS stabil și 10,296 RPS la deschidere ar confirma sau respinge ipoteza.
2. **Consistență — sincronizarea prețurilor.** Presiunea vine din calea stării: furnizor → stocarea celor mai recente prețuri → citire. Ținta de 20 min lasă doar 5 min pentru orice întârziere a noastră; un lot de sincronizare blocat sau o eroare care ignoră ora furnizorului încalcă regula fără ca latența să arate vreo problemă. Ipoteza este respinsă dacă vechimea p99 măsurată rămâne mult sub 20 min, inclusiv la deschidere.
3. **Debit — deschiderea pieței.** Presiunea vine din două căi de citire care primesc trafic suplimentar concentrat. Modelul medie-pe-10-secunde poate subestima vârful pe o secundă. Măsurarea sosirilor pe secundă la deschidere arată dacă ținta de 10,296 RPS este suficientă sau dacă vârful real este mai mare.
4. **Disponibilitate — Market Data Provider.** Presiunea vine din dependența externă, pe care nu o controlăm. Un Dashboard care afișează corect „Indisponibil” respectă consistența, dar acele minute nu sunt utilizabile, deci consumă bugetul de 8 min 49 s. Comparația dintre istoricul de disponibilitate al furnizorului și bugetul nostru arată dacă ținta de 99.9% este realistă cu un singur furnizor.

Concluzie în forma din Lecția 3: *Stock price este un punct de presiune potențial pentru latență și debit, din cauza celor 75.8% din traficul stabil. Următoarea proiectare trebuie să servească această citire fără să repete toată munca pentru fiecare cerere. Mai avem nevoie de un test de încărcare și de distribuția vechimii prețurilor înainte să alegem un mecanism.*

## Listă de verificare

- [x] Am scris cerințe măsurabile pentru toate calitățile de bază (1.1–1.4).
- [x] Am ales și am justificat ținte ale timpului de funcționare pentru ambele părți ale zilei (99.9% și 99%, 1.2).
- [x] Am arătat calculele RPS în regim stabil și la deschiderea pieței pentru toate cele trei niveluri (2, 3).
- [x] Am cercetat și am definit domeniul de aplicare pentru Stocks acceptate (4.1, 4.2).
- [x] Am estimat stocarea brută inițială, zilnică și pentru un an a datelor despre piață (4.5, 4.6).
- [x] Am precizat presupunerile, unitățile, intervalele și rotunjirea finală.
- [x] Am analizat un posibil blocaj pentru fiecare calitate de bază (5).

## Surse

Numerele au fost observate pe 13 octombrie 2026.

- [Investor.gov — Stocks FAQs](https://www.investor.gov/introduction-investing/investing-basics/investment-products/stocks) — definiția stocks, common și preferred stock.
- [Investor.gov — American Depositary Receipts](https://www.investor.gov/introduction-investing/investing-basics/glossary/american-depositary-receipts-adrs) — definiția ADR.
- [StockAnalysis — List of Stock Exchanges](https://stockanalysis.com/list/exchanges/) — Nasdaq 3,426, NYSE 1,934, NYSE American 242.
- [StockAnalysis — All Stocks Listed on the NASDAQ](https://stockanalysis.com/list/nasdaq-stocks/) și [All Stocks Listed on the NYSE](https://stockanalysis.com/list/nyse-stocks/) — conținutul listelor (ADR-uri incluse, ETF-uri excluse).
- [The Motley Fool — The Largest Stock Market Companies in 2026](https://www.fool.com/investing/stock-market/companies/) — verificare încrucișată (aprox. 3,450 Nasdaq, aprox. 2,200 NYSE), actualizat 15 septembrie 2026.
- Lecția 3: Cerințe de calitate, volum de lucru și capacitate (materialul cursului).
