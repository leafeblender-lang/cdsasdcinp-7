# Arhitektura Running modula – Flutter aplikacija (finalna verzija)

Ovaj dokument objedinjuje finalnu arhitekturu running dela aplikacije i dodatno odgovara na teme koje su naknadno otvorene: optimizacija geolokacijskih SQL upita u Supabase-u, smanjenje Mapbox API potrošnje u Flutteru i real-time ažuriranje lokacije uz minimalan battery drain. Dokument kombinuje zvaničnu dokumentaciju, GitHub diskusije i programerske izvore.[web:77][web:169][web:184]

## Status implementacije — 4. avgust 2026.

Ovaj dokument opisuje kompletnu ciljnu arhitekturu. Trenutno je završen i testiran njen Supabase/Firebase serverski deo, ali mobilni Flutter deo još nije kompletiran.

### Implementirano u Supabase projektu `FitnessApp`

- PostGIS i `pg_net` ekstenzije;
- `public` i neizložena `private` šema sa running, route, profile, social i notification tabelama;
- strani ključevi, prostorni/relacijski indeksi, `CHECK` ograničenja, RLS i ograničene dozvole;
- automatsko kreiranje `profiles` reda nakon registracije korisnika;
- kontrolisane RPC funkcije za zdravstveni profil, runner presence, discovery radius, približne i tačne lokacije, mutual connections, hide/block, notification preference i FCM uređaje;
- `finish_running_session()` sa idempotentnim kreiranjem jednog notification eventa;
- evidencija notification deliveries i zaštita od duplikata;
- Vault-zaštićen `pg_net` webhook;
- aktivna Edge Function `send-friend-run-notification`, uključujući deaktiviranje nevažećih FCM tokena;
- transakcioni testovi baze/RPC pravila i webhook autentifikacije.

### Implementirano u Firebase-u

- napravljen Firebase projekat;
- uključen Firebase Cloud Messaging API V1;
- service-account JSON dodat kao Supabase secret `FIREBASE_SERVICE_ACCOUNT_JSON`;
- potvrđeno da Edge Function učitava Firebase konfiguraciju i vraća uspešan testni odgovor.

### Još nije implementirano ili provereno

- Flutter `firebase_core` i `firebase_messaging` integracija;
- Android `google-services.json` i mobilna inicijalizacija Firebase-a;
- preuzimanje/osvežavanje FCM tokena i poziv `register_push_device()` RPC-a;
- foreground/background prikaz i obrada notifikacija u Flutteru;
- end-to-end push test na stvarnom Android uređaju;
- iOS APNs konfiguracija i test na iPhone uređaju;
- kompletan mobilni GPS/Drift/Mapbox/Realtime tok i pripadajući UI.

Zato se „serverska arhitektura završena“ ne sme tumačiti kao „ceo running modul završen“.

## 1. Konačan tehnološki izbor

Za mobilni deo najpraktičniji stack je `geolocator` za GPS, `drift` za lokalno čuvanje tačaka, `flutter_map` za prikaz mape, `health` za import podataka sa satova i opciono `watch_connectivity` za phone-watch komunikaciju.[web:47][web:92] Za backend je najprirodniji izbor Supabase sa Postgres + PostGIS + Auth + Realtime, jer podržava geoprostorne tipove, RPC funkcije i privatne realtime kanale u jednom sistemu.[web:77][web:96][web:99]

Za mapu treba koristiti `flutter_map + Mapbox Static Tiles API` kada se koriste Mapbox Studio stilovi kroz third-party tile URL, jer to nije isti model kao zvanični Mapbox mobilni SDK sa MAU obračunom.[web:154][web:155][web:163] Tačnu besplatnu kvotu i cenu uvek treba proveriti neposredno pred produkciju na aktuelnom Mapbox cenovniku, jer zavise od konkretnog endpointa i plana.[web:94]

## 2. GPS i background tracking

`geolocator` je dovoljan kao primarni paket za lokaciju, uključujući Android foreground service kroz `ForegroundNotificationConfig` i `AndroidSettings`.[web:47][web:53] Za Android 14 ključni su `FOREGROUND_SERVICE_LOCATION`, odgovarajuća lokacijska dozvola i `android:foregroundServiceType="location"`, dok `POST_NOTIFICATIONS` služi za pravilno prikazivanje trajne notifikacije, ali nije uslov da foreground service uopšte krene.[web:157]

Na iOS-u treba uključiti *Background Modes → Location updates*, ali i dalje očekivati ozbiljno testiranje jer sistem može suspendovati aplikaciju ili promeniti ponašanje u pozadini zavisno od režima rada.[web:179] Praktična strategija za potrošnju baterije je da se aktivno trčanje prati preciznijim update-ima, a server i social sloj dobijaju ređe, agregirane update-e umesto svake GPS tačke.[web:179]

## 3. Lokalno čuvanje i sinhronizacija

Sirove GPS tačke treba prvo čuvati lokalno u Drift bazi na oko 2 sekunde, a na server slati završenu ili periodično očišćenu rutu, čime se štede i baterija i Supabase storage.[web:179] Iako kompletan proizvod može imati i serversku tabelu sirovih tačaka za replay i naprednu analizu, najvažnije je da se na server ne gura svaka lokacijska promena u realnom vremenu bez razloga, jer to brzo povećava i troškove i veličinu baze.[web:184][web:185]

Preporučena početna strategija retencije je: lokalno na telefonu dok se sinhronizacija ne potvrdi, serverske raw tačke 7–14 dana, a pojednostavljeni `LineString` trajno. To ostavlja mogućnost za replay, puls-po-ruti i naknadnu analizu, ali sprečava da free tier baze prerano pukne.

## 4. Auth i profiles

Supabase već ima internu tabelu `auth.users`, pa svoju korisničku tabelu ne treba zvati `users` nego `public.profiles`, vezanu foreign key-em na `auth.users(id)`.[web:97] To je i zvanični preporučeni obrazac, jer `auth` šema nije namenjena direktnom klijentskom radu preko public API-ja.[web:97][web:106] Još čistije je odvojiti javne profil podatke od zdravstvenih podataka: `public.profiles` za javne informacije koje aplikacija sme da prikaže drugim korisnicima, a `private.user_health_profiles` za težinu, visinu, datum rođenja i pol, uz stroge privatne politike i bez direktnog izlaganja drugim korisnicima kroz Data API.[web:159][web:204]

```sql
create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  username text unique,
  display_name text,
  avatar_url text,
  created_at timestamptz not null default now()
);

create table private.user_health_profiles (
  user_id uuid primary key references public.profiles(id) on delete cascade,
  weight_kg real,
  height_cm real,
  birth_date date,
  gender text,
  updated_at timestamptz not null default now()
);
```

## 5. Baza – running_sessions

Za glavnu tabelu trčanja bitno je da kolone imaju jasne jedinice u nazivu, jer to kasnije sprečava greške u API-ju, UI-u i analitici. `ended_at` mora biti nullable, pošto sesija može biti tek pokrenuta ili pauzirana dok još nema vreme završetka.

```sql
create table public.running_sessions (
  id bigserial primary key,
  user_id uuid not null references public.profiles(id) on delete cascade,
  started_at timestamptz not null,
  ended_at timestamptz,

  distance_meters integer,
  duration_seconds integer,
  moving_duration_seconds integer,

  average_pace_seconds_per_km integer,
  average_speed_mps real,
  max_speed_mps real,

  elevation_gain_meters real,
  elevation_loss_meters real,

  average_heart_rate_bpm integer,
  max_heart_rate_bpm integer,

  calorie_estimation_method text,
  estimated_calories_kcal real,

  status text not null default 'recording'
    check (status in ('recording', 'paused', 'completed', 'discarded')),

  sync_status text not null default 'pending'
    check (sync_status in ('pending', 'syncing', 'synced', 'failed')),

  source text,
  route_point_count integer,
  synced_at timestamptz,

  route_geom geometry(LineStringZ, 4326)
);

create index running_sessions_route_idx
  on public.running_sessions
  using gist (route_geom);
```

`geometry(LineStringZ, 4326)` je dobar izbor za rutu, jer čuva i elevaciju kao Z koordinatu i praktičan je za sečenje, pojednostavljivanje i dalju obradu rute.[web:77] Dužina rute može se računati kroz `ST_Length(route_geom::geography)` kada je potreban metrički rezultat preko zemljine površine.[web:77]
Nad `public.running_sessions` obavezno uključi RLS tako da korisnik može da čita, menja i briše samo svoje sesije, tipično kroz politike zasnovane na `auth.uid() = user_id`.[web:208][web:209] `public` šema je izložena Data API-ju, pa bez RLS-a drugi autentifikovani korisnici ne smeju imati pristup tuđoj istoriji kretanja.[web:159][web:204]


## 6. Baza – sirove tačke

Ako želiš replay i ozbiljniju naknadnu analizu, možeš odmah imati i serversku tabelu sirovih tačaka. Za nju su važniji `session_id`, `seq` i `recorded_at` nego prostorni indeks nad svakom tačkom, jer se replay skoro uvek čita po redosledu unutar sesije, ne prostorno po svim tačkama iz cele baze.

```sql
create table public.running_session_points (
  id bigserial primary key,
  session_id bigint not null references public.running_sessions(id) on delete cascade,
  seq integer not null,
  geom geometry(PointZ, 4326),
  recorded_at timestamptz not null,
  elevation_meters real,
  heart_rate_bpm integer,
  accuracy_meters real,
  unique (session_id, seq)
);

create index running_session_points_session_seq_idx
  on public.running_session_points (session_id, seq);
```

GiST indeks nad svakom sirovom tačkom ima smisla samo ako planiraš mnogo prostornih upita direktno nad tim tačkama. Za standardni replay i većinu trening prikaza dovoljan je `(session_id, seq)` indeks. Za `public.running_session_points` RLS treba da proverava vlasništvo preko povezane `running_sessions` sesije, tako da korisnik može pristupiti samo tačkama svojih treninga.[web:208][web:209]

## 7. Popularne rute

DBSCAN ne treba koristiti kao jedini generator ruta, jer ne razume redosled kretanja, smer, raskrsnice ni topologiju puta.[web:32][web:35] Bolji pristup je da se završene rute očiste, pojednostave, podele na standardizovane segmente i onda broji koliko različitih korisnika i dana prolazi kroz svaki segment. Za završnu korisničku rutu je i dalje koristan `LineStringZ`, ali za prepoznavanje preklapanja popularnih deonica praktičnije je držati segmente kao 2D `LineString`, jer bučna GPS elevacija ne treba da razdvoji dve realno iste ulične deonice.[web:207]

```sql
create table public.route_segments (
  id bigserial primary key,
  geom geometry(LineString, 4326),
  unique_user_count integer,
  unique_day_count integer,
  average_elevation_meters real,
  average_slope_percent real,
  last_updated timestamptz
);

create index route_segments_geom_idx
  on public.route_segments using gist (geom);

create table public.route_candidates (
  id bigserial primary key,
  geom geometry(LineStringZ, 4326),
  popularity_score real,
  name text
);

create index route_candidates_geom_idx
  on public.route_candidates using gist (geom);
```

Praktični pipeline je: očisti GPS greške → ukloni deonice blizu kuće → pojednostavi rutu → segmentiši → agregiraj popularnost → spajaj susedne popularne segmente u kandidatske rute. DBSCAN može ostati kao pomoćni alat za heatmap zone, ali ne za glavnu logiku. U sistem za popularne rute treba da ulazi samo očišćena ruta bez početne i završne deonice i bez `user_id` u javno čitljivim rezultatima, dok originalne korisničke rute i dalje ostaju privatne.

## 8. Social sloj – finalna struktura

Za "show me on map" deo treba odvojiti trenutno prisustvo, preference pretrage i relacije među korisnicima. Najosetljiviji podaci ne treba da budu direktno čitljivi klijentu, nego samo kroz kontrolisane RPC funkcije i privatne realtime kanale.[web:96][web:99][web:159]

### Prisustvo

Na početku migracija dodaj i privatnu šemu, uz pravilo da `private` ne bude izložena kroz Supabase Data API.[web:195][web:204]

```sql
create schema if not exists private;

create table private.runner_presence (
  user_id uuid primary key references public.profiles(id) on delete cascade,
  running_session_id bigint references public.running_sessions(id) on delete cascade,
  current_location geography(Point, 4326),
  discovery_token uuid not null unique default gen_random_uuid(),
  show_me_on_map boolean not null default false,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  expires_at timestamptz not null,
  accuracy_meters real
);

create index runner_presence_location_idx
  on private.runner_presence using gist (current_location);
```

`geography(Point, 4326)` je dobar izbor za live lokaciju i radius upite u metrima, što je i obrazac koji se često koristi u Supabase/PostGIS primerima.[web:77][web:172] `expires_at` služi da prisustvo automatski zastari ako telefon prestane da šalje lokaciju. `discovery_token` treba da bude kratkotrajan token vezan za trenutno prisustvo, tako da nepoznati trkači ne dobijaju pravi `profiles.id` i ne mogu dugoročno da prate istu osobu preko stabilnog identifikatora.

### Preference pretrage

```sql
create table public.runner_discovery_preferences (
  user_id uuid primary key references public.profiles(id) on delete cascade,
  search_radius_km integer check (
    search_radius_km is null
    or search_radius_km in (1, 5, 20, 50)
  ),
  updated_at timestamptz not null default now()
);
```

Vrednosti mogu biti `1`, `5`, `20`, `50` ili `NULL` za neograničeno. Ovaj radius utiče samo na to koliko daleko korisnik traži druge, ne na to ko njega može videti.

### Osetljive relacije

```sql
create table private.connection_intents (
  from_user_id uuid references public.profiles(id) on delete cascade,
  to_user_id uuid references public.profiles(id) on delete cascade,
  created_at timestamptz default now(),
  primary key (from_user_id, to_user_id),
  check (from_user_id <> to_user_id)
);

create table public.connections (
  user_low_id uuid references public.profiles(id) on delete cascade,
  user_high_id uuid references public.profiles(id) on delete cascade,
  connected_at timestamptz default now(),
  primary key (user_low_id, user_high_id),
  check (user_low_id < user_high_id)
);

create table private.user_blocks (
  blocker_id uuid references public.profiles(id) on delete cascade,
  blocked_id uuid references public.profiles(id) on delete cascade,
  created_at timestamptz default now(),
  primary key (blocker_id, blocked_id),
  check (blocker_id <> blocked_id)
);

create table private.location_hidden_users (
  owner_user_id uuid references public.profiles(id) on delete cascade,
  hidden_from_user_id uuid references public.profiles(id) on delete cascade,
  created_at timestamptz default now(),
  primary key (owner_user_id, hidden_from_user_id),
  check (owner_user_id <> hidden_from_user_id)
);
```

`private.connection_intents`, `private.user_blocks` i `private.location_hidden_users` treba da ostanu privatni, jer korisnik ne sme direktno da sazna da ga je neko blokirao, sakrio ili poslao intent. `public.connections` može ostati javna uz RLS koji svakom korisniku prikazuje samo njegove konekcije.

## 9. Social pravila ponašanja

Finalna pravila su sledeća:

- **Show me on map OFF** → niko ga ne vidi.
- **Show me on map ON** → mogu ga pronaći korisnici čiji `search_radius_km` doseže do njega.
- **Nisu povezani** → vidi se samo približna prisutnost.
- **Povezani** → vidi se tačna lokacija dok je deljenje aktivno.
- **Povezani, ali sakriveni** → vidi se samo približna prisutnost.
- **Blokirani** → potpuna nevidljivost i zabrana povezivanja.

Ovakvo ponašanje sprečava da korisnik lako zaključi da je baš posebno sakriven, jer i dalje vidi približnu prisutnost kao običan nepovezani korisnik.

## 10. RPC funkcije i bezbednost

`get_nearby_runners()` ne treba da prima radius kao parametar. Bezbednije je da sama funkcija čita `search_radius_km` iz `public.runner_discovery_preferences`, da klijent ne može ručno proslediti nenormalno veliku vrednost mimo svojih podešavanja. Pored toga, i kod `NULL = neograničeno` treba uvesti `LIMIT`, paginaciju ili cursor.

`get_nearby_runners()` interno koristi tačnu lokaciju, ali ne treba da vraća pravi korisnički UUID nepoznatim trkačima. Umesto toga treba da vrati `discovery_token`, `distance_bucket` i eventualno ograničene javne profil podatke. `send_connection_intent(target_discovery_token)` tada treba da prima `discovery_token`, a server interno prevodi token u stvarni `user_id`. Kada korisnik ugasi deljenje ili mu prisustvo istekne, token prestaje da važi. Token treba rotirati pri svakoj novoj `Show me on map` sesiji. `get_connected_runner_locations()` vraća tačne lokacije samo kada postoji konekcija, nema blokiranja, prisustvo nije isteklo, korisnik trenutno deli lokaciju i nije sakrio tačnu lokaciju baš od tražioca.

Za `send_connection_intent(target_discovery_token)` treba koristiti jednu atomsku RPC funkciju koja u jednoj transakciji prevodi token u korisnika, proverava self-connect, blokiranja, postojeći intent, obrnuti intent i po potrebi kreira konekciju, pa briše oba intenta. Za već postojećeg prijatelja može postojati posebna funkcija bazirana na UUID-u, ali za korisnike otkrivene na mapi klijent ne treba da dobije pravi UUID. Za `block_user(blocked_user_id)` u istoj transakciji treba dodati blok, obrisati intente, obrisati konekciju i obrisati eventualne hide redove između ta dva korisnika, ali **ne treba brisati ceo `runner_presence` red korisnika**, jer bi to sakrilo lokaciju i svim ostalim dozvoljenim korisnicima.

Za Supabase RPC funkcije koje pristupaju privatnim tabelama treba koristiti `security definer`, prazan `search_path`, puna imena šema i eksplicitno ograničene `EXECUTE` dozvole, jer Supabase dokumentacija preporučuje eksplicitno grant/revoke upravljanje za sigurniji Data API pristup.[web:159][web:164][web:167]

## 11. Kako optimizovati SQL upite za geolokaciju u Supabase-u

Za radius pretrage i sortiranje po udaljenosti u Supabase-u najpraktičniji obrazac je PostGIS + RPC funkcija, a ne običan `.select()` filter iz klijenta.[web:172][web:174] U GitHub diskusijama i issue-jima oko Supabase Flutter-a često se preporučuje upravo `.rpc()` za prostorne upite, jer tako možeš koristiti `ST_DWithin`, `ST_Distance`, sortiranje po udaljenosti i dodatna security pravila na serverskoj strani.[web:169][web:172]

Glavne optimizacije su:

- koristi `geography(Point, 4326)` za live lokacije kada su bitni metri i radius upiti;[web:77]
- koristi GiST indeks nad geolokacijom;[web:176]
- ograniči broj vraćenih rezultata (`LIMIT`, cursor);[web:176]
- osloni se na autovacuum kao podrazumevani mehanizam održavanja, a ručni `VACUUM`, `ANALYZE` ili `REINDEX` radi samo kada monitoring i `EXPLAIN ANALYZE` pokažu realan problem.[web:196][web:198][web:194]

Kod popularnih ruta i složenijih prostornih obrada korisno je držati `geometry(LineStringZ, 4326)` za samu rutu, dok se live prisustvo drži kao `geography(Point, 4326)`. Ta podela smanjuje komplikacije i omogućava da svaki tip podataka radi ono u čemu je najbolji.

## 12. Strategije za smanjenje potrošnje Mapbox API kredita u Flutteru

Najvažnija strategija je da ne koristiš mapu kao da je GPS stream – mapa ne mora da refresha tile-ove svaki put kada dobiješ novu lokaciju. Kada korisnik trči, njegova pozicija i polilinija mogu da se osvežavaju lokalno u Flutter widget stablu, dok se tile pozadina učitava samo kada se viewport zaista pomeri ili kada korisnik zumira.

Programerski izvori i GitHub diskusije oko `flutter_map` ukazuju na nekoliko korisnih optimizacija:

- koristi tile cache; `flutter_map` ima caching podršku i ekosistem plugin-a oko cache-a, uključujući `flutter_map_tile_caching` za napredniji offline i disk cache pristup.[web:186][web:184]
- koristi dobar `TileProvider` i razumno keširanje da iste tile-ove ne tražiš stalno ponovo.[web:185]
- debounce ili optimizuj tile update ponašanje; u community iskustvima je pomagalo npr. `tileUpdateTransformer` da smanji blink i nepotrebne refreshe.[web:180]
- nemoj imati nepotrebno agresivno centriranje mape na svaku GPS promenu; centriraj samo kad korisnik prati sebe, a i tada sa throttling-om.

Važna praktična napomena: plugin-i za bulk/offline cache moraju se koristiti u skladu sa pravilima samog tile provajdera. To što `flutter_map_tile_caching` tehnički može preuzimati regione ne znači da svaki provider dozvoljava isto ponašanje, pa Mapbox i OSM pravila treba proveriti odvojeno.[web:184][web:48]

## 13. Implementacija real-time ažuriranja lokacije uz minimalan battery drain

Za running app nije dobro slati svaku GPS tačku svim drugim korisnicima u realnom vremenu. Praktičan početni standard je sledeći:

- lokalni GPS zapis: svake 2 sekunde,
- tačna live lokacija konektovanih korisnika: svakih 10 sekundi,
- `runner_presence` update: svakih 30 sekundi,
- `runner_presence.expires_at`: 90 sekundi od poslednjeg update-a.

Ovakva podela smanjuje i mrežni saobraćaj i Realtime poruke, a i dalje daje korisniku osećaj da sistem "radi uživo". Odnos 30 sekundi heartbeat i 90 sekundi expiration znači da jedan propušten mrežni update neće odmah ukloniti korisnika sa mape. Supabase Realtime broadcast sa privatnim kanalima je bolji izbor za češće evente nego sirovo oslanjanje na open table changes, posebno kada želiš fino kontrolisanu autorizaciju.[web:96][web:99][web:187]

Na mobilnoj strani, preporuka je da se lokacija lokalno bufferuje, a da se mreža koristi ređe i pametnije. U stručnim primerima background geolocation implementacija često se kombinuju lokalni buffer, foreground service na Androidu i promena strategije kada app ode u pozadinu, upravo radi baterije i pouzdanosti.[web:179]

## 14. Satovi i live puls

Za MVP je dovoljno čitati završene treninge preko `health` paketa iz HealthKit-a i Health Connect-a. To ti daje puls, kalorije i eventualno rutu nakon treninga, ali ne live stream senzora.[web:16][web:19]

Za live puls tokom treninga potrebna je posebna watch integracija:

- Wear OS: Health Services / `ExerciseClient`.[web:79][web:83]
- Apple Watch: `HKWorkoutSession` + `HKLiveWorkoutBuilder`.[web:58]
- komunikacija telefon ↔ sat: `watch_connectivity`.[web:92]

To je besplatno sa strane SDK-a, ali Apple distribucija i dalje traži Apple Developer Program članstvo za javnu objavu aplikacije.[web:84]

## 15. Kalorije

Hijerarhija izvora za kalorije treba da bude:

1. sat je dao active energy,
2. nema active energy, ali postoji puls → formula sa pulsom,
3. nema pulsa → MET formula.

Kalorije treba uvek prikazivati kao procenu. To je tačnije i korektnije prema korisniku od predstavljanja jedne cifre kao apsolutne istine.

## 16. Friend Run Finished Notifications

Ovaj modul omogućava da konektovani korisnici dobiju push notifikaciju kada njihov prijatelj završi trčanje. Funkcija je opciona i kontrolišu je obe strane:

1. korisnik bira kojim prijateljima želi da se pošalje obaveštenje o njegovom završenom trčanju;
2. primalac bira od kojih prijatelja želi da prima ovakva obaveštenja.

Primer neutralne notifikacije:

```text
Milan je danas istrčao 5,5 km za 32:10.
```

Podrazumevani tekst treba da bude neutralan. Motivacioni ili takmičarski ton može kasnije biti dodat kao posebno podešavanje.

### 16.1. Podešavanja korisnika

Svaki korisnik ima dve nezavisne postavke:

- `share_mode` — ko može da dobije obaveštenje kada korisnik završi trčanje;
- `receive_mode` — od kojih prijatelja korisnik želi da prima obaveštenja.

Podržani režimi:

- `all_friends` — svi konektovani prijatelji;
- `all_except_some` — svi osim izabranih prijatelja;
- `nobody` — potpuno isključeno.

```sql
create table public.friend_run_notification_preferences (
  user_id uuid primary key
    references public.profiles(id)
    on delete cascade,

  share_mode text not null default 'all_friends'
    check (share_mode in ('all_friends', 'all_except_some', 'nobody')),

  receive_mode text not null default 'all_friends'
    check (receive_mode in ('all_friends', 'all_except_some', 'nobody')),

  updated_at timestamptz not null default now()
);
```

Primer:

```text
Milan:
Share my finished runs → All friends except Marko

Petar:
Receive friend run notifications → All except Milan
```

Kada Milan završi trčanje, Marko ne dobija notifikaciju jer ga je Milan isključio, Petar je ne dobija jer je utišao Milana, a ostali dozvoljeni prijatelji je dobijaju.

### 16.2. Izuzeci na strani pošiljaoca

Tabela predstavlja izbor: „Obavesti sve moje prijatelje osim ovih korisnika.”

```sql
create table private.run_notification_share_exclusions (
  owner_user_id uuid not null
    references public.profiles(id) on delete cascade,
  excluded_friend_id uuid not null
    references public.profiles(id) on delete cascade,
  created_at timestamptz not null default now(),
  primary key (owner_user_id, excluded_friend_id),
  check (owner_user_id <> excluded_friend_id)
);
```

- `owner_user_id` je korisnik koji je završio trčanje.
- `excluded_friend_id` je prijatelj kojem se ne šalje obaveštenje.

Tabela ostaje u `private` šemi kako isključeni prijatelj ne bi mogao da sazna da je posebno izostavljen.

### 16.3. Utišani prijatelji na strani primaoca

Tabela predstavlja izbor: „Ne želim da dobijam obaveštenja o trčanjima ovih prijatelja.”

```sql
create table private.run_notification_receive_mutes (
  owner_user_id uuid not null
    references public.profiles(id) on delete cascade,
  muted_friend_id uuid not null
    references public.profiles(id) on delete cascade,
  created_at timestamptz not null default now(),
  primary key (owner_user_id, muted_friend_id),
  check (owner_user_id <> muted_friend_id)
);
```

- `owner_user_id` je primalac notifikacije.
- `muted_friend_id` je prijatelj čije su running notifikacije utišane.

Utišani prijatelj ne dobija informaciju da je utišan.

### 16.4. Push uređaji

Jedan korisnik može imati više uređaja, pa push token ne treba čuvati direktno u `profiles`.

```sql
create table private.push_devices (
  id bigserial primary key,
  user_id uuid not null
    references public.profiles(id) on delete cascade,
  provider text not null default 'fcm'
    check (provider in ('fcm')),
  push_token text not null,
  platform text not null
    check (platform in ('android', 'ios')),
  is_active boolean not null default true,
  last_seen_at timestamptz not null default now(),
  disabled_at timestamptz,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (provider, push_token)
);
```

Flutter aplikacija koristi Firebase Cloud Messaging. Kada se FCM token promeni, aplikacija šalje novi token serveru. Ako FCM prijavi da token više nije validan, uređaj se označava kao neaktivan. Push tokeni ostaju u privatnoj tabeli.

### 16.5. Notification event i tok slanja

Notifikaciju ne treba slati direktno iz Flutter aplikacije. Kada se trčanje završi, serverska funkcija kreira poseban event:

```sql
create table private.notification_events (
  id bigserial primary key,
  event_type text not null
    check (event_type in ('friend_run_finished')),
  actor_user_id uuid not null
    references public.profiles(id) on delete cascade,
  running_session_id bigint not null
    references public.running_sessions(id) on delete cascade,
  created_at timestamptz not null default now(),
  unique (event_type, running_session_id)
);
```

Jedinstveno ograničenje sprečava da isto završeno trčanje kreira više identičnih događaja.

Preporučeni tok:

```text
finish_running_session()
        ↓
running_sessions.status = completed
        ↓
kreira se notification_event
        ↓
Database Webhook
        ↓
Supabase Edge Function
        ↓
Firebase Cloud Messaging
        ↓
push notifikacija na uređaju
```

Event se kreira iz serverske funkcije `finish_running_session()`, a ne na osnovu proizvoljnog klijentskog update-a.

### 16.6. Evidencija isporuka i zaštita od duplikata

```sql
create table private.notification_deliveries (
  id bigserial primary key,
  event_id bigint not null
    references private.notification_events(id) on delete cascade,
  recipient_user_id uuid not null
    references public.profiles(id) on delete cascade,
  status text not null default 'pending'
    check (status in ('pending', 'sent', 'failed', 'skipped')),
  sent_at timestamptz,
  failure_reason text,
  created_at timestamptz not null default now(),
  unique (event_id, recipient_user_id)
);
```

Ova tabela sprečava da isti primalac dobije istu notifikaciju više puta, čak i ako se webhook ili Edge Function ponovo izvrše.

### 16.7. Pravila slanja

Prijatelj dobija notifikaciju samo kada su ispunjeni svi uslovi:

1. trčanje je prvi put uspešno prešlo u `completed`;
2. postoji aktivna konekcija između korisnika;
3. nema blokiranja ni u jednom smeru;
4. pošiljalac ima `share_mode != 'nobody'`;
5. ako je `share_mode = 'all_except_some'`, primalac nije u `run_notification_share_exclusions`;
6. primalac ima `receive_mode != 'nobody'`;
7. ako je `receive_mode = 'all_except_some'`, pošiljalac nije u `run_notification_receive_mutes`;
8. primalac ima najmanje jedan aktivan push uređaj;
9. za kombinaciju eventa i primaoca još ne postoji poslata isporuka.

Za MVP nije potrebno uvoditi minimalnu kilometražu, dnevni limit, cooldown ili digest. To se može dodati kasnije ako notifikacije postanu prečeste.

### 16.8. Sadržaj notifikacije

Podrazumevani neutralni tekst:

```text
Milan je danas istrčao 5,5 km za 32:10.
```

Drugi primeri:

```text
Milan je upravo završio trčanje: 8,2 km.
```

```text
Tvoj prijatelj Milan završio je današnje trčanje.
```

Kasnije se može dodati podešavanje tona:

- `neutral`;
- `motivational`;
- `competitive`.

Primer motivacionog tona:

```text
Milan je danas istrčao 5,5 km. Tvoj red!
```

Takmičarski tekst poput „Nemoj da zaostaješ” ne treba da bude podrazumevan.

Push payload može sadržati:

```text
notification_type
running_session_id
actor_display_name
distance_meters
duration_seconds
```

Ne treba slati GPS koordinate, celu rutu, zdravstvene podatke ili privatne profilne podatke. Klik na notifikaciju može otvoriti sažetak trčanja samo ako je trkač dozvolio prijateljima da vide te podatke.

### 16.9. UI podešavanja

Predložena sekcija:

```text
Running
  → Settings
    → Social notifications
```

Opcije:

```text
Share my finished runs
  ○ All connected friends
  ○ All except selected friends
  ○ Nobody
```

```text
Friend run notifications
  ○ All connected friends
  ○ All except selected friends
  ○ Nobody
```

Kada korisnik izabere `All except selected friends`, prikazuje se lista njegovih konektovanih prijatelja. Promene se čuvaju preko kontrolisanih RPC funkcija, a ne direktnim upisom u privatne tabele.

### 16.10. Konačan tehnički izbor za notifikacije

```text
Supabase Postgres
+ privatne notification tabele
+ finish_running_session() RPC
+ Database Webhook
+ Supabase Edge Function
+ Firebase Cloud Messaging
+ firebase_messaging Flutter paket
```

Server-side pristup pre slanja proverava konekcije, blokiranja, podešavanja pošiljaoca i primaoca, mute/exclusion liste, aktivne uređaje i zaštitu od duplih notifikacija.

Ovo podržava:

```text
Share with all friends except {A, B, C}
```

kao i:

```text
Receive from all friends except {X, Y, Z}
```

bez otkrivanja korisnicima da su posebno isključeni ili utišani.

---

## 17. Konačan zaključak

Finalni proizvodni stack za running deo aplikacije je:

| Sloj | Preporuka |
|---|---|
| GPS | `geolocator` |
| Lokalna baza | `drift` |
| Mapa | `flutter_map + Mapbox Static Tiles API` |
| Backend | `Supabase + Postgres + PostGIS + Auth + Realtime` |
| Import sa satova | `health` |
| Live satovi | Wear OS Health Services / Apple Watch HealthKit workout API |
| Social bezbednost | privatne tabele + RPC funkcije + RLS + privatni realtime kanali |
| Push notifikacije | Firebase Cloud Messaging + Supabase Edge Function |
| Popularne rute | PostGIS segmentacija + batch job |

Najvažnija arhitekturna odluka je da se sistem ne zasniva na direktnom izlaganju lokacija klijentu, već na privatnim tabelama i kontrolisanim serverskim funkcijama. To daje bolju bezbednost, lakše održavanje i više prostora da kasnije dodaš naprednije funkcije bez rušenja osnovnog modela.[web:96][web:159]
