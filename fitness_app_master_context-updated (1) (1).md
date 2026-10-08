# Fitness aplikacija — master context za budući AI

## 1. Svrha ovog dokumenta

Ovaj dokument predstavlja glavni kontekst projekta i namenjen je budućem AI asistentu ili developeru koji treba da razume aplikaciju bez ponovnog objašnjavanja ideje od nule.

Aplikacija nije samo baza vežbi niti samo running tracker. Cilj je da bude sveobuhvatna fitness platforma sa više velikih modula:

1. interaktivna 3D anatomija i baza vežbi;
2. sistemi i konkretni planovi treninga;
3. istezanje i mobilnost;
4. fitness izazovi;
5. ozbiljan running modul koji funkcioniše skoro kao nezavisna aplikacija;
6. profil, statistika, napredak, društvene funkcije i premium sadržaj.

Aplikacija se razvija u Flutteru za Android i iOS. Backend i baza koriste Supabase, odnosno PostgreSQL sa PostGIS ekstenzijom za geoprostorne podatke.

## 1.1. Trenutni status implementacije — 4. avgust 2026.

### Završeno i provereno

- Supabase projekat `FitnessApp` je aktivan i predstavlja glavni backend.
- PostGIS i `pg_net` su uključeni.
- Napravljene su running, route, social, profile i notification tabele u `public` i neizloženoj `private` šemi.
- Dodati su strani ključevi, prostorni i relacijski indeksi, `CHECK` ograničenja, RLS politike i ograničene dozvole.
- Napravljene su kontrolisane RPC funkcije za zdravstveni profil, running sesije, prisustvo na mapi, radius pretragu, konekcije, hide/block logiku, FCM uređaje i notification preference.
- `finish_running_session()` atomski završava sesiju i kreira samo jedan notification event.
- Postavljen je Vault-zaštićen `pg_net` webhook i aktivna Supabase Edge Function `send-friend-run-notification`.
- Firebase projekat je napravljen, Firebase Cloud Messaging API V1 je uključen i Firebase service-account JSON je sačuvan kao Supabase Edge Function secret `FIREBASE_SERVICE_ACCOUNT_JSON`.
- Supabase → Edge Function test je uspešan (`200 OK`), a baza/RPC arhitektura je prošla transakcione testove bez ostavljanja test podataka.

### Još nije završeno

- Flutter aplikacija još nije povezana sa FCM klijentom (`firebase_core`, `firebase_messaging` i `google-services.json`).
- Flutter još ne preuzima, osvežava i šalje FCM token kroz `register_push_device()` RPC.
- Nije urađen end-to-end test stvarne push notifikacije na fizičkom Android uređaju.
- iOS APNs povezivanje, Apple push capabilities i test na iPhone uređaju ostaju za kasnije.
- Mobilni GPS tracking, Drift sinhronizacija, mape, privatni Realtime kanali i UI social/notifikacionih podešavanja i dalje se implementiraju u Flutter sloju.

Važno: završen je serverski temelj running/social/notification sistema, ali to ne znači da je ceo running modul u mobilnoj aplikaciji završen.

---

# 2. Glavna vizija proizvoda

Korisnik treba da može da koristi jednu aplikaciju za:

- učenje anatomije i pronalaženje vežbi za konkretan mišić;
- gledanje pravilnog izvođenja vežbe;
- izbor sistema treninga;
- praćenje konkretnog višenedeljnog plana;
- pravljenje ili izvođenje treninga sa običnim serijama, superserijama, giant setovima, ladder i AMRAP logikom;
- istezanje i mobilnost;
- završavanje izazova i praćenje ličnih rezultata;
- snimanje trčanja pomoću GPS-a;
- pregled trasa, tempa, kilometraže, uspona i potrošnje kalorija;
- pronalaženje popularnih running ruta;
- povezivanje sa drugim trkačima uz pažljivo rešenu privatnost;
- primanje opcionih obaveštenja kada konektovani prijatelj završi trčanje;
- kasnije povezivanje sa pametnim satovima.

Aplikacija treba da ima besplatni deo i premium sadržaj. Besplatni korisnik dobija osnovnu vrednost, dok premium može otključavati veći broj planova, tutorijal animacija, napredniju analitiku, dodatne izazove i druge napredne funkcije.

---

# 3. Glavni moduli aplikacije

## 3.1. Exercises & 3D Body

Ovo je centralni edukativni i anatomski deo aplikacije.

### Glavna ideja

Korisnik bira muško ili žensko telo, otvara interaktivni 3D model i može:

- da rotira model;
- da bira prednju ili zadnju stranu;
- da klikne region tela;
- da klikne konkretnu subregiju ili mišić;
- da vidi koje vežbe aktiviraju izabrani mišić;
- da otvori detalje vežbe.

3D model se trenutno prikazuje kroz WebView. Flutter učitava lokalnu web stranicu, a JavaScript šalje događaje nazad u Flutter preko handlera poput:

- `onMuscleSelected`;
- `onViewerError`.

Svaki klikabilni deo 3D modela treba da ima stabilan `mesh_name` koji se povezuje sa odgovarajućim zapisom mišića u Supabase bazi.

### Anatomija

Planirana hijerarhija:

```text
body_regions
  -> subregions
    -> muscles
```

Primeri glavnih regija:

- arms;
- shoulders;
- chest;
- back;
- legs;
- abs/core;
- opciono neck.

Primer podele ramena:

- anterior deltoid;
- lateral deltoid;
- posterior deltoid.

Primer detaljnije podele leđa:

- upper trapezius;
- middle trapezius;
- lower trapezius;
- latissimus dorsi;
- erector spinae;
- infraspinatus;
- teres major;
- teres minor.

Primer core podele:

- upper_abs;
- lower_abs;
- obliques;
- deep_core.

Primer glavnih mišića ruku:

- biceps long head;
- biceps short head;
- brachialis;
- triceps long head;
- triceps lateral head;
- triceps medial head;
- opciono mišići podlaktice.

Mišići, regije i subregije treba da imaju stabilne tehničke identifikatore kao što su `ref_id`, dok se prikazani nazivi mogu prevoditi i menjati.

### Vežbe

Svaka vežba može imati:

- tehnički `ref_id`;
- prikazani naziv;
- kratak opis;
- detaljna uputstva izvođenja;
- opremu;
- lokaciju izvođenja;
- težinu;
- tip pokreta;
- unilateral/bilateral oznaku;
- animaciju ili video;
- varijante vežbe;
- aktivirane mišiće;
- druge filtere.

Razlika:

- `description` objašnjava šta je vežba, čemu služi i šta korisnik treba da zna;
- `instructions` daju konkretne korake pravilnog izvođenja.

### Veza vežbe i mišića

Vežba može aktivirati više mišića različitim intenzitetom.

Planirana logika uloga:

- primary;
- secondary;
- third ili weak activation/stabilizer.

Početna ideja intenziteta:

- primary = 3;
- secondary = 2;
- third = 1.

Bolje dugoročno rešenje je posebna relaciona tabela, npr. `exercise_muscles`, u kojoj se čuvaju:

- `exercise_id`;
- `muscle_id`;
- `role`;
- opciono `intensity`.

Jedna vežba može imati više primarnih i više sekundarnih mišića.

### Oprema i lokacije

Vežbe treba da se filtriraju po opremi i mestu izvođenja.

Primeri opreme:

- barbell;
- dumbbell;
- cable;
- machine;
- bodyweight;
- resistance band;
- bench;
- pull-up bar.

Primeri lokacija:

- gym;
- home;
- outdoor.

---

## 3.2. Training Systems & Plans

Ovaj deo ima dva različita nivoa koji se ne smeju mešati.

### Nivo 1: sistem treninga / split

Primeri:

- Upper/Lower;
- Push/Pull/Legs;
- Full Body;
- Body Part Split;
- Bro Split;
- Arnold Split;
- drugi sistemi koji mogu biti dodati kasnije.

Ovo su kategorije ili načini organizovanja treninga, a ne konkretni planovi.

### Nivo 2: konkretan plan treninga

Za svaki split postoji više konkretnih planova.

Primer:

```text
Push/Pull/Legs
  -> PPL za početnike, 3 dana nedeljno
  -> PPL hipertrofija, 6 dana
  -> PPL snaga
  -> PPL kod kuće
```

Primer:

```text
Upper/Lower
  -> Upper/Lower za početnike
  -> Upper/Lower 4 dana
  -> Upper/Lower hipertrofija
  -> Upper/Lower strength + hypertrophy
```

Konkretan plan treba da može da sadrži:

```text
training_split
  -> training_plan
    -> plan_weeks
      -> workout_days
        -> workout_exercises
```

Plan može imati:

- nivo iskustva;
- cilj;
- broj dana nedeljno;
- trajanje u nedeljama;
- potrebnu opremu;
- mesto izvođenja;
- procenjeno trajanje treninga;
- dane odmora;
- pravila progresije;
- zamenske vežbe;
- deload logiku kasnije.

### Struktura treninga

Za vežbe unutar treninga već je planirana sledeća logika:

- `group_id`: povezuje vežbe koje se rade zajedno;
- `group_type`: `superset` ili `giant_set`;
- `set_type`: `straight`, `ladder`, `amrap`;
- `reps_per_set`: niz ponavljanja kada je `set_type = ladder`.

Obične samostalne vežbe imaju:

- `group_id = NULL`;
- `group_type = NULL`.

Vežbe istog superseta ili giant seta imaju isti `group_id`, ali samo unutar jednog konkretnog treninga.

Trening treba da podrži:

- obične serije;
- superserije;
- giant setove;
- AMRAP;
- ladder/piramide;
- broj ponavljanja;
- vremenske serije;
- odmor;
- tempo;
- opcione napomene.

---

## 3.3. Stretching & Mobility

Istezanje treba da bude zaseban modul, ne samo filter među običnim vežbama.

Moguće kategorije:

- full-body stretching;
- morning stretching;
- post-workout stretching;
- pre-run warm-up;
- post-run stretching;
- shoulder mobility;
- hip mobility;
- back and neck mobility;
- lower-body mobility;
- kratke rutine od 5, 10 ili 20 minuta;
- statičko istezanje;
- dinamičko istezanje.

Stretching rutina može imati:

- niz pokreta;
- trajanje svakog pokreta;
- stranu tela;
- broj ponavljanja;
- vreme odmora;
- region ili mišić;
- animaciju;
- instrukcije disanja;
- upozorenja.

Ovaj modul može deliti deo baze sa vežbama, ali ne treba silom koristiti potpuno istu workout logiku ako su trajanje i kontinuitet važniji od serija i ponavljanja.

---

## 3.4. Challenges

Challenge sekcija je posebna celina sa ciljevima i rezultatima.

Primeri:

- push-up pyramid;
- pull-up pyramid;
- plank challenge;
- squat challenge;
- 30-day challenges;
- AMRAP izazovi;
- timed challenges;
- staircase/pyramid challenges;
- streak izazovi;
- lični rekordi;
- kasnije izazovi između prijatelja.

Challenge može imati:

- naziv;
- opis;
- pravila;
- cilj;
- trajanje;
- maksimalno vreme;
- broj rundi;
- broj ponavljanja;
- nivo;
- potrebnu opremu;
- istoriju pokušaja;
- najbolji rezultat;
- završene dane;
- streak;
- achievement ili badge.

Challenge nije isto što i workout plan. Plan je strukturirani program, dok challenge ima jasan cilj, rezultat ili vremensko ograničenje.

---

# 4. Running modul

Running deo treba da funkcioniše skoro kao nezavisna aplikacija unutar glavne fitness platforme.

Glavne podsekcije mogu biti:

```text
Running
  -> Dashboard
  -> Start Run / Active Run
  -> History
  -> Run Details
  -> Statistics
  -> Routes
  -> Popular Routes
  -> Saved Routes
  -> Nearby Runners
  -> Connections
  -> Running Settings
```

## 4.1. GPS praćenje

Tehnološki izbor:

- `geolocator` za lokaciju;
- `drift` za lokalno čuvanje GPS tačaka;
- `flutter_map` za mapu;
- Mapbox Static Tiles API kao produkcioni tile servis;
- Supabase + PostGIS za server i geoprostorne podatke.

GPS tačke se lokalno čuvaju približno svake 2 sekunde.

Na server se ne šalje svaka sirova tačka odmah. Tokom trčanja podaci se čuvaju lokalno, a zatim se šalje završena i očišćena ruta.

Background tracking:

- Android koristi foreground location service i trajnu notifikaciju;
- iOS koristi Background Modes -> Location updates;
- obavezno je testiranje na realnim uređajima različitih proizvođača.

## 4.2. Podaci trčanja

Za svako trčanje treba čuvati:

- početak i kraj;
- ukupno vreme;
- moving time;
- kilometražu;
- prosečan tempo;
- prosečnu brzinu;
- maksimalnu brzinu;
- uspon;
- spuštanje;
- procenjene kalorije;
- prosečan puls;
- maksimalan puls;
- izvor podataka;
- GPS rutu;
- broj tačaka;
- status sinhronizacije.

Glavna ruta:

```text
geometry(LineStringZ, 4326)
```

`Z` koordinata može čuvati elevaciju.

Sirove tačke mogu se čuvati na serveru kratko, npr. 7–14 dana, radi replay-a i naknadne analize. Pojednostavljena ruta se čuva trajno.

## 4.3. Kalorije

Prioritet izvora:

1. sat je dao active energy;
2. postoji puls, ali nema active energy -> procena preko formule sa pulsom;
3. nema pulsa -> MET formula.

Uvek prikazivati:

```text
Procenjena potrošnja kalorija
```

Ne predstavljati broj kao potpuno precizan.

Težina, visina, datum rođenja i pol treba da budu privatni zdravstveni podaci, odvojeni od javnog profila.

## 4.4. Pametni satovi

MVP:

- `health` paket;
- import završenih treninga iz HealthKit-a i Health Connect-a;
- puls, kalorije i ruta nakon treninga, kada su dostupni.

Live senzori kasnije:

- Wear OS: Health Services / `ExerciseClient`;
- Apple Watch: `HKWorkoutSession` + `HKLiveWorkoutBuilder`;
- `watch_connectivity` za komunikaciju telefon-sat.

Live satovi zahtevaju posebne companion aplikacije i native Kotlin/Swift rad. To nije mali dodatak i treba ga ostaviti za kasniju fazu.

## 4.5. Popularne rute

Aplikacija treba anonimno da koristi završene GPS tragove korisnika koji su dali dozvolu kako bi otkrila gde mnogo različitih ljudi često trči.

Pipeline:

1. očistiti GPS greške;
2. ukloniti početak i kraj blizu kuće korisnika;
3. pojednostaviti LineString;
4. podeliti trasu na standardizovane 2D segmente;
5. brojati različite korisnike i različite dane;
6. segmente iznad praga popularnosti spojiti u graf;
7. napraviti `route_candidates`;
8. odobriti i objaviti kvalitetne rute.

DBSCAN može služiti za heatmap i detekciju gustih zona, ali ne kao glavni generator ruta.

Originalne korisničke rute ostaju privatne. Javno se prikazuju samo anonimizovani agregati i odobrene rute.

## 4.6. Show Me on Map

Korisnik može uključiti opciju `Show me on map`.

Važno pravilo:

- korisnik bira koliko daleko on želi da traži druge;
- korisnik ne bira jednostrano na kojoj udaljenosti drugi mogu da ga vide;
- privatnost se rešava isključivanjem opcije, blokiranjem ili skrivanjem lokacije od konkretnog prijatelja.

Mogući radijusi pretrage:

- 1 km;
- 5 km;
- 20 km;
- 50 km;
- neograničeno.

Pre konekcije se ne prikazuje tačna pozicija. Drugi korisnik dobija samo približnu prisutnost ili distance bucket.

Nepoznatom korisniku se ne vraća pravi stabilni UUID. Koristi se kratkotrajni `discovery_token` koji važi samo tokom trenutne presence sesije.

## 4.7. Mutual connection sistem

Ne postoji klasičan accept/decline request.

Logika:

1. korisnik A pošalje intent korisniku B;
2. B ne dobija obaveštenje;
3. ako B nezavisno pošalje intent korisniku A;
4. backend vidi oba intenta;
5. automatski kreira konekciju;
6. oba intenta se brišu.

Tek kada oba korisnika nezavisno pokažu interesovanje, oni postaju konektovani.

Za nepoznate korisnike intent se šalje preko `discovery_token`, a server interno pronalazi pravi korisnički ID.

## 4.8. Tačna lokacija prijatelja

Tačna lokacija se prikazuje samo kada:

- postoji konekcija;
- niko nije blokiran;
- korisnik trenutno deli lokaciju;
- presence nije istekla;
- korisnik nije sakrio lokaciju baš od tog prijatelja.

Konekcija ne znači trajni pristup lokaciji.

Početni intervali:

- lokalni GPS: svake 2 sekunde;
- tačna live lokacija konektovanih: svakih 10 sekundi;
- presence heartbeat: svakih 30 sekundi;
- presence expiration: 90 sekundi.

## 4.9. Hide i Block

### Hide location from friend

- korisnici ostaju konektovani;
- prijatelj ne vidi tačnu lokaciju;
- vidi samo približnu prisutnost kao nekonektovani korisnik;
- nema obaveštenja da je sakriven;
- može se kasnije ponovo uključiti.

### Block

- potpuna nevidljivost u oba smera;
- nema približne ni tačne prisutnosti;
- ne mogu slati intent;
- postojeća konekcija se uklanja;
- intenti se brišu;
- blokirani korisnik ne dobija obaveštenje.

Blokiranje mora biti moguće i za nekoga ko nije prijatelj, preko njegovog važećeg `discovery_token`.

---


## 4.10. Friend Run Finished Notifications

Kada korisnik završi trčanje, njegovi konektovani prijatelji mogu opciono da dobiju push notifikaciju, na primer:

```text
Milan je danas istrčao 5,5 km za 32:10.
```

Kontrolu imaju obe strane:

- trkač bira kojim prijateljima želi da pošalje obaveštenje o svom završenom trčanju;
- primalac bira od kojih prijatelja želi da prima ovakve notifikacije.

Za obe strane podržani su režimi:

- `all_friends`;
- `all_except_some`;
- `nobody`.

Sistem zato mora da podrži:

- `friend_run_notification_preferences`;
- `run_notification_share_exclusions`;
- `run_notification_receive_mutes`;
- `push_devices`;
- `notification_events`;
- `notification_deliveries`.

Pre slanja server proverava:

- da postoji konekcija;
- da nema blokiranja;
- podešavanja pošiljaoca;
- podešavanja primaoca;
- exclusion i mute liste;
- aktivne push uređaje;
- zaštitu od duplih notifikacija.

Preporučeni tok:

```text
finish_running_session()
        ↓
notification event
        ↓
Database Webhook
        ↓
Supabase Edge Function
        ↓
Firebase Cloud Messaging
        ↓
Android / iOS uređaj
```

Supabase ostaje glavni backend. Firebase se koristi samo za Cloud Messaging, ne za Auth, bazu ili Storage.

Podrazumevani tekst notifikacije treba da bude neutralan. Motivacioni ili takmičarski ton može kasnije biti opciono podešavanje.

---

# 5. Profil, statistika i zajednički sistemi

Svi moduli dele isti korisnički sistem.

## Javni profil

Može sadržati:

- username;
- display name;
- avatar;
- javne achievement-e;
- opciono javne statistike;
- datum kreiranja naloga.

## Privatni zdravstveni profil

Treba držati odvojeno:

- weight_kg;
- height_cm;
- birth_date;
- gender;
- druge zdravstvene podatke kasnije.

## Favorites

Korisnik može imati omiljene:

- vežbe;
- treninge;
- planove;
- stretching rutine;
- challenge-e;
- running rute.

## Statistika

Moguće zajedničke statistike:

- broj završenih treninga;
- ukupno trajanje treninga;
- nedeljni i mesečni streak;
- ukupan volumen;
- lični rekordi;
- broj završenih planova;
- broj završenih izazova;
- ukupna running kilometraža;
- prosečan tempo;
- najduže trčanje;
- ukupni uspon;
- kalorije kao procena.

---

# 6. Predlog glavne navigacije

Moguća glavna navigacija:

```text
Home
Exercises
Plans
Challenges
Running
Profile
```

Stretching može biti:

- zaseban tab;
- deo `Plans`;
- posebna velika sekcija unutar Home-a.

Running treba da se otvara kao poseban ozbiljan deo sa sopstvenom unutrašnjom navigacijom.

Moguća Home strana prikazuje:

- nastavi trenutni plan;
- sledeći trening;
- preporučenu vežbu;
- stretching rutinu;
- aktivni challenge;
- poslednje trčanje;
- nedeljni napredak.

---

# 7. Tehnička arhitektura

## Frontend

- Flutter;
- Android i iOS;
- WebView za postojeći 3D model;
- JavaScript komunikacija za izbor mesh-a;
- `drift` za lokalne podatke;
- `geolocator` za GPS;
- `flutter_map` za running mapu;
- `firebase_messaging` za push notifikacije.

## Backend

- Supabase Auth;
- PostgreSQL;
- PostGIS;
- Supabase Realtime;
- Postgres RPC funkcije;
- Row Level Security;
- privatna šema za osetljive podatke;
- periodični batch poslovi za popularne rute;
- Supabase Database Webhooks i Edge Functions za notification workflow;
- Firebase Cloud Messaging samo kao servis za dostavljanje push poruka.

## Bezbednosna pravila

- uključiti RLS na svim tabelama izloženim Data API-ju;
- korisnik čita i menja samo svoje originalne treninge i rute;
- zdravstveni podaci nisu javni;
- runner presence je u privatnoj šemi;
- location, block, hide i connection intent logika ide kroz kontrolisane RPC funkcije;
- ne izlagati `service_role` ključ Flutter aplikaciji;
- tačne lokacije se nikada ne čitaju direktnim klijentskim `select` upitom.

---

# 8. Glavne grupe tabela

Ovo nije kompletna finalna SQL šema, već pregled domena.

## Korisnici

- `profiles`;
- `user_health_profiles`;
- `user_settings`;
- premium/subscription tabele;
- achievements;
- favorites.

## Anatomija i vežbe

- `body_regions`;
- `subregions`;
- `muscles`;
- `exercises`;
- `exercise_muscles`;
- `equipment`;
- `locations`;
- varijante vežbi;
- media/animations.

## Trening sistemi i planovi

- `training_splits`;
- `training_plans`;
- `plan_weeks`;
- `workout_templates`;
- `workout_template_exercises`;
- korisnički aktivni planovi;
- završeni treninzi;
- exercise logs;
- sets.

## Stretching

- `stretching_routines`;
- `stretching_routine_items`;
- stretch/mobility movements;
- korisnički završene rutine.

## Challenges

- `challenges`;
- `challenge_steps`;
- `challenge_attempts`;
- `challenge_progress`;
- lični rekordi;
- streak podaci.

## Running

- `running_sessions`;
- `running_session_points`;
- `route_segments`;
- `route_candidates`;
- `runner_discovery_preferences`;
- `runner_presence`;
- `connection_intents`;
- `connections`;
- `user_blocks`;
- `location_hidden_users`;
- `friend_run_notification_preferences`;
- `run_notification_share_exclusions`;
- `run_notification_receive_mutes`;
- `push_devices`;
- `notification_events`;
- `notification_deliveries`.

---

# 9. Monetizacija

Osnovna ideja:

## Free

- osnovna baza vežbi;
- ograničen broj tutorijal animacija;
- osnovni planovi;
- osnovno praćenje treninga;
- osnovno GPS trčanje;
- ograničene statistike;
- deo izazova i istezanja.

## Premium

Moguće premium funkcije:

- sve animacije;
- svi napredni planovi;
- planovi po cilju i nivou;
- napredna statistika;
- više challenge sadržaja;
- napredne running analize;
- personalizovane preporuke;
- dodatne popularne i sačuvane rute;
- napredni filteri;
- cloud istorija i dodatni export;
- kasnije funkcije povezane sa satovima.

Tačna premium podela još nije zaključana i ne treba je smatrati finalnom dok se ne definiše MVP.

---

# 10. Redosled razvoja

Preporučen redosled:

1. stabilizovati anatomsku bazu i 3D izbor mišića;
2. završiti bazu vežbi i veze sa mišićima;
3. napraviti prikaz detalja vežbe;
4. definisati training splits i konkretne planove;
5. implementirati izvođenje i logovanje treninga;
6. dodati stretching modul;
7. dodati challenge modul;
8. implementirati osnovni running tracking;
9. povezati završene rute sa Supabase/PostGIS bazom;
10. dodati running history, statistiku i popularne rute;
11. tek kasnije dodati social location sloj;
12. dodati friend run push notifikacije;
13. dodati import sa HealthKit/Health Connect;
14. live satovi ostaviti za kasnu fazu.

Ne pokušavati da se sve završi odjednom. Svaki modul treba prvo da ima stabilnu osnovnu verziju.

Status 4. avgusta 2026: serverska osnova za korake 9, 11 i 12 već je implementirana u Supabase-u, ali odgovarajući Flutter ekrani, GPS tokovi, FCM registracija uređaja i end-to-end mobilni testovi još nisu završeni.

---

# 11. Trenutne važne odluke

- Flutter je glavni frontend framework.
- Supabase je implementirani glavni backend; Flutter povezivanje sa svim novim RPC funkcijama još traje.
- 3D model se učitava kroz WebView.
- Mišići se povezuju sa 3D mesh nazivima.
- Vežbe se povezuju sa mišićima preko relacione strukture.
- Training split nije isto što i konkretan plan.
- Stretching i Challenges su posebni moduli.
- Running je skoro nezavisna podaplikacija.
- Raw GPS tačke se prvenstveno čuvaju lokalno.
- Zdravstveni podaci moraju biti privatni.
- Tačne live lokacije ne smeju direktno biti izložene klijentu.
- Konekcija nastaje samo kada oba korisnika nezavisno pošalju intent.
- Hide location nije isto što i Block.
- Friend run notifikacije su opcione i kontrolišu ih i pošiljalac i primalac.
- Supabase ostaje glavni backend; FCM se koristi samo za dostavljanje push notifikacija.
- Popularne rute se formiraju iz anonimizovanih segmenata, ne iz javno izloženih originalnih ruta.

---

# 12. Način saradnje sa korisnikom

Korisnik preferira rad u malim, jasnim koracima.

Za coding zadatke:

1. prvo objasniti koja se funkcija implementira i zašto;
2. dati mali, konkretan korak;
3. navesti tačan fajl i mesto izmene;
4. ne menjati celu arhitekturu bez potrebe;
5. ne slati ogromne fajlove ako je dovoljan mali diff;
6. kod većih zadataka podeliti rad u faze;
7. jasno razdvojiti obavezno od opcionog;
8. ne uvoditi dodatne komplikacije koje korisnik nije tražio.

Korisnik želi praktične odgovore, jasna imena tabela i kolona i objašnjenje kako će nešto stvarno raditi u aplikaciji.

Kod pitanja o bazi treba posebno paziti na:

- relacije;
- foreign key pravila;
- cascade ponašanje;
- RLS;
- privatne podatke;
- upite koji će se stvarno koristiti;
- skaliranje i trošak.

---

# 13. Šta još nije finalno

Sledeće stvari još mogu da se menjaju:

- finalni vizuelni dizajn i navigacija;
- tačan premium model;
- tačan broj i sadržaj početnih planova;
- kompletan spisak stretching rutina;
- kompletan spisak challenge-a;
- finalna struktura svih workout log tabela;
- tačan 3D model i finalna imena svih mesh objekata;
- finalni tile provider i Mapbox plan pre produkcije;
- live integracije sa satovima;
- algoritam za automatsko spajanje popularnih route segmenata;
- pravila preporuka i personalizacije.
- Flutter FCM integracija i prikaz/obrada push notifikacija;
- iOS APNs konfiguracija i testiranje;
- end-to-end test friend-run notifikacije na stvarnim uređajima.

Budući AI ne treba da pretpostavi da su ove tačke završene. Treba da koristi postojeću ideju kao smer, a konkretne šeme i odluke potvrđuje kroz implementaciju.

---

# 14. Kratak opis aplikacije za prezentaciju

Ovo je sveobuhvatna fitness platforma koja kombinuje interaktivni 3D model tela i bazu vežbi, strukturirane programe treninga, rutine istezanja i mobilnosti, fitness izazove i napredan running sistem koji funkcioniše skoro kao zasebna aplikacija. Korisnik može da izabere mišić na 3D modelu, pronađe odgovarajuće vežbe i tutorijale, prati konkretan trening plan, beleži napredak, završava izazove i koristi GPS za praćenje trčanja. Running deo obuhvata istoriju ruta, tempo, elevaciju, kalorije, popularne rute, pametne satove i pažljivo dizajnirane društvene funkcije sa približnom i preciznom lokacijom, međusobnim povezivanjem, opcionim friend activity notifikacijama, blokiranjem i kontrolom privatnosti.

---

# 15. Jednorečenična suština

Aplikacija treba da bude jedinstveno mesto za vežbe, anatomiju, planiranje i praćenje treninga, istezanje, izazove i ozbiljno praćenje trčanja, sa jasnim putem od besplatnog korisnika do premium fitness iskustva.
