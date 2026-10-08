# Pregled "Run" modula — primedbe

Datum: 2026-08-26
Obuhvat: `fitness_plus/lib/features/running/**`, Android native kanali, Supabase (`running_sessions`,
`running_session_points`, `sync_completed_running_session`, `restore_running_sessions`), integracija sa
`progress` i `social`.

Stanje testova: `flutter test test/features/running` → **225/225 prolazi**. Sve što je niže navedeno je
kod koji testovi ne pokrivaju ili pokrivaju pod pretpostavkom koja slučajno sakriva grešku.

Oznake: **P0** = kvari podatke ili gasi funkcionalnost · **P1** = trošak / performanse · **P2** = doslednost i higijena.

---

## P0-1. Haversine koristi stepene umesto radijana → distanca je pogrešna, zavisno od geografske širine

`lib/features/running/domain/run_tracking.dart:691-696`

```dart
final a =
    sinLatitude * sinLatitude +
    math.cos(firstLatitude) *          // <-- STEPENI
        math.cos(secondLatitude) *     // <-- STEPENI
        sinLongitude *
        sinLongitude;
```

`firstLatitudeRadians` i `secondLatitudeRadians` su izračunati tri reda iznad, ali se koriste **samo** za
`latitudeDelta`. U `cos()` ulaze sirovi stepeni. Rezultat: istok-zapad komponenta svake distance množi se
faktorom `cos²(lat_u_stepenima) / cos²(lat_u_radijanima)`, koji je praktično nasumičan broj između 0 i ~1.5
i menja se na svakih ~3° geografske širine.

### Izmereno (pokrenuto kroz sam kod aplikacije)

Korak od 10 m tačno na istok:

| lat | prijavljeno | stvarno | odnos |
|---|---|---|---|
| 42.0 | 5.38 m | 9.99 m | 0.54 |
| 44.0 | 13.88 m | 9.99 m | **1.39** |
| 44.8125 (Beograd) | 9.63 m | 9.99 m | 0.95 |
| 45.25 (Novi Sad) | 4.24 m | 9.99 m | **0.42** |
| 45.55 | 0.04 m | 9.99 m | **0.004** |
| 47.0 | 14.53 m | 9.99 m | **1.45** |
| 52.0 | 2.64 m | 9.99 m | 0.27 |

Ceo hod od 838.6 m pravo na istok, 1 Hz, savršeni fiksevi, kroz `RunMetricsAccumulator`:

| lat | bez Doppler brzine | sa Doppler brzinom |
|---|---|---|
| 44.8 | 800.6 m (−4.5%) | 802.0 m (−4.4%) |
| 45.25 | **348.6 m (−58%)** | 772.1 m (−7.9%) |
| 47.0 | **1212.7 m (+45%)** | 912.1 m (+8.7%) |

### Zašto do sada nije primećeno

1. `test/features/running/domain/run_tracking_accuracy_test.dart:55` hardkoduje `latitude = 44.8` — jedna
   od retkih širina gde je greška samo ~5%, i to je jedina širina na kojoj se accuracy test ikad izvršava.
2. `speedBudget` (Doppler integral) delimično spasava totale — on je *pod* i *plafon* oko sirove distance,
   pa kad telefon daje pouzdanu brzinu, ukupna kilometraža se vraća u ±9%. Zato su runovi u bazi
   (Beograd, ~44.8°) izgledali normalno.

### Šta sve trpi, pored totala

- `_movementThreshold` poređenje (`run_tracking.dart:497`) — prag kretanja poredi pogrešnu distancu.
- Detekcija teleporta `impliedSpeed = rawDistance / elapsedSeconds` (`:409`, `:465`) — na 44° i 47°
  distanca je naduvana ~45%, pa se legitimni koraci proglašavaju `implausibleJump` i run se cepa na
  segmente; na 45.55° distanca je ~0, pa se pravi teleport nikad ne detektuje.
- `sharpensProvisionalAnchor` i "stoji u mestu" grana (`:419`, `:439`) porede `rawDistance` sa `accuracy`.
- Plafon max brzine u analizi (`run_analysis.dart:564`).
- `active_run_social_controller.dart:167` — udaljenost do drugog trkača, od koje zavisi da li se prelazi
  na `closePublishInterval` (10 s umesto 30 s). Znači i **direktno na Supabase trošak**.

### Popravka

```dart
final a =
    sinLatitude * sinLatitude +
    math.cos(firstLatitudeRadians) *
        math.cos(secondLatitudeRadians) *
        sinLongitude *
        sinLongitude;
```

I obavezno: accuracy test parametrizovati po širini (bar 42, 45.25, 45.55, 47, 52), inače se ista klasa
greške vraća neprimećeno.

---

## P0-2. ~12.5% runova bez pauze nikad ne stigne na server (RPC ih trajno odbija)

Klijent šalje (`data/remote/supabase_run_sync_gateway.dart:40-45`):

```dart
'p_duration_seconds': endedAt.difference(session.startedAt).inSeconds,  // seče naniže
'p_moving_duration_seconds': activeSeconds.round(),                     // zaokružuje
```

Server (`sync_completed_running_session`) diže izuzetak:

```sql
or p_moving_duration_seconds > p_duration_seconds
   then raise exception 'Invalid run metrics' using errcode = '22023';
```

Za run bez ijedne pauze važi `activeDuration ≈ endedAt − startedAt` **tačno**, jer se `startedAt` prepisuje
na zidni sat prvog fiksa (`active_run_controller.dart:861`), a `_activeSegmentStartedElapsed` na monotoni
sat istog trenutka (`:857`).

Dodatno, Drift po defaultu čuva `DateTime` kao **Unix sekunde** (nema `store_date_time_values_as_text` u
projektu) — potvrđeno i u bazi: svi `started_at`/`ended_at` imaju 0 mikrosekundi. Znači `startedAt` i
`endedAt` se pri čitanju iz lokalne baze seku na celu sekundu, a `activeDuration` ostaje u milisekundama.

Neka je `D = (E−S)/1000 = n + f`, i `s` = razlomljeni deo `S/1000`:

- `p_duration_seconds = n + [s+f ≥ 1]`
- `p_moving_duration_seconds = n + [f ≥ 0.5]`

Odbijanje se dešava kad `f ≥ 0.5` **i** `s + f < 1`:
`P = ∫₀.₅¹ (1−f) df = 0.125` → **1 od 8 runova bez pauze**.

Posledice: run ostaje `syncStatus = failed`, `_scheduleRetry` ga pokušava svakih 30 s i pri svakom
pokretanju aplikacije — **zauvek**, sa istim ishodom. Beskonačni potrošeni zahtevi + korisnik trajno gubi run
sa naloga.

Indicija u bazi: `running_sessions.id` ima rupe na 1,2,3,4,7,8,9. Sequence vrednosti se troše i kad
transakcija padne, pa je to konzistentno sa odbačenim uploadima (nije dokaz — mogu biti i stare iteracije).

### Popravka (klijent, najmanja izmena)

```dart
final durationSeconds =
    endedAt.difference(session.startedAt).inSeconds.clamp(0, 2147483647);
...
'p_duration_seconds': durationSeconds,
'p_moving_duration_seconds':
    activeSeconds.round().clamp(0, durationSeconds),
```

Bolje dugoročno: slati milisekunde (`p_moving_duration_ms`) i pustiti server da zaokružuje, ili u RPC-u
dozvoliti toleranciju od 1 s. I: dodati test koji vozi ceo `uploadCompletedRun` sa `startedAt`/`endedAt`
razmaknutim za npr. 485.7 s.

---

## P0-3. Obaveštenje "prijatelj je završio run" nikad se ne šalje

`running_shell.dart:160`:

```dart
unawaited(coordinator.syncCompletedRun(session.localId));   // notifyFriends podrazumevano false
```

To je **jedini** poziv `syncCompletedRun` u celom projektu. `notifyFriends` nigde nije `true`, pa je
`p_emit_notification` uvek `false`, pa se `finish_running_session` nikad ne poziva iz sync putanje.

Potvrda iz baze (`private.notification_events`):

| event_type | count |
|---|---|
| friend_run_started | 17 |
| friend_run_finished | **0** |

…uz 13 uspešno sinhronizovanih završenih runova.

Cela `finish_running_session` funkcija i deo `get_friend_run_notification_settings` /
`set_friend_run_notification_preferences` / `replace_run_notification_receive_mutes` površine su mrtvi:
korisnik podešava preferencije za notifikaciju koja se ne emituje.

Popravka: proslediti `notifyFriends: true` (uslovljeno korisničkim `share_mode`-om), ili — ako je odluka
da se ova notifikacija ne šalje — ukloniti podešavanje iz UI-ja.

---

## P0-4. `route_geom` je NULL za svaki run koji ima pauzu

`sync_completed_running_session`:

```sql
when count(*) >= 2 and count(distinct point.segment_index) = 1
  then extensions.st_makeline(point.geom order by point.seq)
else null
```

3 od 13 runova u bazi nemaju `route_geom`. Danas je bezopasno (vidi P1-3 — kolona se nigde ne čita), ali
ako se ikad uvede prostorni upit nad rutama, polovina istorije će nedostajati bez ikakvog signala.
Ispravno bi bilo `ST_Collect` / `MultiLineString` po segmentima, ili izbaciti kolonu.

---

## P1-1. `restore_running_sessions` povlači celu istoriju pri svakom hladnom startu

`run_sync_coordinator.dart:84-95` → `_handleAuthChanged` okida `restoreAccountRuns` čim se `userId` promeni
sa `null` na vrednost. To je **svako** pokretanje aplikacije.

`restore_running_sessions(p_limit DEFAULT 200)` vraća do 200 sesija sa **svim** tačkama rute, kao jedan
jsonb. Klijent zatim čita `getKnownRunIds()` i baca 99% dobijenog.

Izmereno na trenutnoj bazi: **41 kB za 13 kratkih runova (166 tačaka)**. Projekcija za realnog korisnika
sa 200 runova × ~35 tačaka (≈7 tačaka/km × 5 km) ≈ **1.4 MB nekompresovanog JSON-a po pokretanju, po korisniku**.
Na 10.000 korisnika koji otvore aplikaciju jednom dnevno → reda veličine 3–4 GB egress-a dnevno, za podatke
koji su već lokalno.

Predlozi (po ceni implementacije):

1. Preskočiti restore kad lokalna baza već ima runove tog korisnika — restore je za reinstall/novi telefon.
   Jedan `getKnownRunIds()` **pre** mrežnog poziva, i ako nije prazno, samo lagani "koliko ih ima" upit.
2. Dodati `p_known_client_run_ids uuid[]` parametru RPC-a i vraćati samo ono što nedostaje.
3. Ili dvofazno: prvo RPC koji vraća samo `client_run_id[]`, pa drugi koji povlači tačke za razliku.

---

## P1-2. Redundantan indeks na `running_session_points`

```
running_session_points_session_id_seq_key   UNIQUE btree (session_id, seq)   idx_scan = 0
running_session_points_session_seq_idx      btree (session_id, seq)          idx_scan = 378
```

Dva identična indeksa nad istim kolonama. Planer koristi običan, ali UNIQUE nosi constraint i pokriva iste
upite. **Obrisati `running_session_points_session_seq_idx`** — jedan indeksni upis manje po svakoj tački
rute, i manje diska.

## P1-3. `route_geom` + GiST indeks se nikad ne čitaju

`running_sessions_route_idx` (GiST): `idx_scan = 0`. Provera svih `sql`/`plpgsql` funkcija u `public` i
`private`: `route_geom` se pominje **samo** u `sync_completed_running_session`, i to samo pri upisu.
Klijent rutu čita iz `running_session_points`.

Znači: `ST_MakeLine` se računa pri svakom uploadu, GiST indeks se održava pri svakom uploadu, i ništa to
ne koristi. Kandidat za brisanje (kolona + indeks).

## P1-4. Mrtve kolone na `running_sessions`

Nikad ih ne piše `sync_completed_running_session` niti bilo ko drugi:
`max_speed_mps`, `elevation_gain_meters`, `elevation_loss_meters`, `average_heart_rate_bpm`,
`max_heart_rate_bpm`, `calorie_estimation_method`, `estimated_calories_kcal`, i
`running_session_points.heart_rate_bpm`.

Uz to — nova migracija `running_session_summary_metrics` (20260826120000) uvodi `summary_metrics.max_speed_mps`
**iako `max_speed_mps` kolona već postoji i prazna je**. Ili napuniti postojeću kolonu (jeftinije za upite),
ili je obrisati da ne postoje dva izvora istine.

Isto važi za `elevation_gain/loss`: `RunAnalyzer` ih izračuna, ekran ih prikaže, ali se nikad ne šalju — pa
restore-ovan run gubi elevaciju i pored toga što za nju postoje kolone.

## P1-5. Lokalna baza raste bez ograničenja i bez čišćenja

`local_run_points` čuva **svaki** opaženi fiks — i prihvaćene i odbijene — zauvek, i posle uspešnog
uploada.

Konkretno merenje iz baze: run `id = 16` traje 1678 s (≈1678 fikseva na 1 Hz), a na server je otišlo
**49 tačaka**. Odnos ≈ **34 : 1**. Sat vremena trčanja ≈ 3600 redova ≈ 300 kB lokalno; 200 takvih runova
≈ 60 MB, plus WAL.

Nema ni GC-a ni prikaza zauzeća. Predlog: posle `syncStatus == synced` i nakon što je `summary_metrics`
zapisan, prorediti sirove tačke starijih runova (npr. zadržati RDP-pojednostavljenu rutu + altitudu), ili
bar odbijene fikseve (`accepted = false`) brisati posle N dana — oni ne ulaze ni u jednu meru, samo u
brojač `observedPointCount`.

Sitno vezano: `getRecoverableRun()` uzima `ORDER BY started_at DESC LIMIT 1`. Ako ikad nastanu dva reda u
statusu `recording`/`paused` (npr. `_safelyDiscardOrphanedSession` padne), stariji postaje trajno nevidljiv
— nije u istoriji, nije u recovery-ju, a nosi sve svoje tačke.

## P1-6. Kadenca objave prisustva tokom runa

`active_run_social_controller.dart` — 30 s u proseku, 10 s kad je konekcija blizu. Za run od 40 min to je
~80 RPC poziva **po runu po korisniku**, plus isto toliko čitanja markera.

Kadenca ne zavisi od toga da li uopšte ima koga na mapi. Predlog: ako poslednja objava vrati 0 markera,
usporiti na 60–120 s i vratiti se na 30 s čim se neko pojavi. Za korisnika koji trči sam (a to je većina)
to je 2–4× manje zahteva bez ikakve razlike u doživljaju.

## P1-7. Keš mapa može zauzeti 200+ MB

`core/map/cached_map_tile_provider.dart:19` — `maxNrOfCacheObjects: 2000`, uz `512@2x` pločice (1024×1024 px).
Pri ~100 kB po pločici to je ~200 MB na disku, bez gornje granice po veličini. Vredi spustiti broj ili preći
na `flutter_cache_manager` konfiguraciju sa limitom po bajtovima.

(Za pohvalu: `tileDimension: 512` + `zoomOffset: -1` je tačno prava odluka za Mapbox naplatu po zahtevu.)

---

## P1-8. Tri puna rebuild-a u sekundi tokom runa

Potvrđeno čitanjem `drift-2.34.3/lib/src/runtime/executor/stream_queries.dart:327-346` — `fetchAndEmitData`
**ne poredi** novi rezultat sa prethodnim, emituje bezuslovno pri svakoj promeni tabele.

`saveObservedPoint` (`running_database.dart:330`) u transakciji radi `UPDATE local_runs` **pri svakom fiksu**,
dakle ~1 Hz. Zbog toga se, po sekundi:

1. `_enqueuePoint` → `_emit` → `notifyListeners()` — (1)
2. `watchRecentCompletedRuns(limit: 20)` u `ActiveRunController` ponovo izvrši upit, mapira 20 redova u
   `RunSession` i emituje → `_emit(recentRuns:)` → `notifyListeners()` — (2)
3. `_uiTimer` 1 Hz → `refreshElapsedTime` → `_emit` → `notifyListeners()` — (3)
4. plus `ProgressController` ima **svoju** pretplatu `watchRecentCompletedRuns(limit: 1)`
   (`progress_controller.dart:108`) — još jedan SQL upit u sekundi (rebuild ne izaziva; 2 s debounce ga
   spasava od sweep-a).

`ActiveRunScreen` je jedan `AnimatedBuilder` oko **celog** stabla (`active_run_screen.dart:245`), pa svaki
od ta tri notify-ja rekonstruiše i `RunMapView`. A `RunningDashboardScreen` ispod (`MaterialPageRoute`,
`maintainState: true`) takođe se rebuild-uje i layout-uje iako se ne crta.

Predlozi:
- `.distinct()` (ili poređenje po `localId`+`updatedAt`) na `watchRecentCompletedRuns` u
  `drift_run_store.dart:80`, ili pauzirati tu pretplatu dok je `phase` u `recording`/`acquiringFix`.
- Suziti `AnimatedBuilder` na instrumente; mapu izvući iznad njega i hraniti je zasebnim
  `ValueNotifier<List<List<RunPoint>>>` koji se menja samo kad se ruta zaista produži.
- `_startUiTimer` i `_emit` iz `refreshElapsedTime` menjaju samo `activeDuration` — može zaseban notifier
  za sat.

## P1-9. `RunMapView` gradi celu polyline listu pri svakom build-u

`run_map_view.dart:199-217` — `routeSegments.map(...).toList()` na svaki build. Uz P1-8 to je N `LatLng`
alokacija u sekundi (N = broj sidrišta rute, raste kroz ceo run). Memoizovati po identitetu
`widget.routeSegments`.

## P1-10. Ekran statistike učitava **sve** tačke **svih** runova odjednom

`run_statistics_screen.dart:49-58`:

```dart
final runs = await widget.controller.loadCompletedRuns();          // bez limita
final analyses = await widget.controller.loadRunAnalyses(runs, ...); // Future.wait preko svih
```

`loadRunAnalyses` (`active_run_controller.dart:102`) radi `Future.wait` — za svaki run `getObservations()`
povlači **svaki** red iz `local_run_points`, i sve analize se drže u memoriji istovremeno. Za 300 runova
× ~1500 redova = ~450.000 redova mapiranih u objekte na UI izolatu.

A od svega toga statistici trebaju samo `elevationGain` i `estimatedCalories` — distanca, vreme i tempo
dolaze iz `session.metrics`. Predlozi:
- Sračunati `elevationGainMeters`/`elevationLossMeters` jednom pri `finish()` i upisati ih u `local_runs`
  (kolone na serveru već postoje, vidi P1-4), pa statistika ne dira `local_run_points` uopšte.
- Ili filtrirati po periodu **pre** učitavanja opservacija.

## P1-11. Istorija bez paginacije i bez lazy liste

`run_history_screen.dart:37` → `getCompletedRuns()` bez limita, a `_HistoryList` koristi
`ListView(children: [...])` — svaki red za svaki run se gradi odmah. Za `ListView.builder` / `SliverList`
ovde nema prepreke.

Isto: `RunningStatistics.calculate` se poziva unutar `build()` (`run_statistics_screen.dart:102`), a
`_buildTrend` je O(broj_kanti × N) sa `where` prolazom po kanti.

## P1-12. Detaljni prikaz crta nepojednostavljenu rutu

`run_details_screen.dart:202` prosleđuje `analysis.routeSegments` — puni lokalni trag (hiljade tačaka),
dok isti run restore-ovan sa naloga ima ~34× manje. `selectRouteIndices` iz
`domain/route_simplification.dart` već postoji; primeniti ga i pri crtanju. Manje posla i vizuelno
konzistentno između lokalnog i restore-ovanog runa.

---

## P2-1. Podešavanje km/milje se ignoriše u celom Run modulu

`RunnerDistanceUnit` postoji, ima RPC `set_runner_distance_unit`, ali se koristi **samo** za prikaz
radijusa u `account_privacy_sheets.dart`. `presentation/run_formatters.dart` je hardkodovan na km,
min/km, km/h i metre — distanca, tempo, brzina, elevacija, splitovi, notifikacija na zaključanom ekranu.
Korisnik može da izabere milje i ništa se ne promeni.

## P2-2. Tekst geolocator notifikacije zaobilazi lokalizaciju

`data/location/geolocator_location_source.dart:77-78` — `'Fitness Plus'` / `'Tracking your run in the
background'` su hardkodovani engleski stringovi, van `app_en.arb`. Sve ostalo na kartici runa ide kroz ARB
(`RunNotificationLabels`). Trenutno je samo `en` lokalizacija, pa je posledica kozmetička, ali je to
jedina poruka koju modul emituje mimo l10n pipeline-a.

## P2-3. `com.example.fitness_plus` svuda

`applicationId`, `android:label="fitness_plus"`, `userAgentPackageName` u oba `TileLayer`-a, imena svih
method channel-a. Sa `com.example.*` se ne može objaviti na Play Store, a promena package-a posle prve
objave znači novu aplikaciju. Vredi rešiti pre prvog release-a.

## P2-4. `_WeekSummary` broji samo poslednjih 20 runova

`running_dashboard_screen.dart:386` računa nedeljni zbir iz `state.recentRuns`, koji je
`watchRecentCompletedRuns(limit: 20)`. Korisnik sa više od 20 runova u 7 dana dobija manji zbir nego što
jeste. Retko, ali tiho pogrešno.

## P2-5. Prag elevacije od 1.5 m nad sirovom GPS visinom

`run_analysis.dart:55` — `_minimumElevationChangeMeters = 1.5`. GPS visina (i sa `useMSLAltitude`) ima
korelisan šum reda ±5–15 m. Sa uzorkom po sidrištu (≈1 Hz) i pragom od 1.5 m, ravan run od 30 min može
akumulirati stotine metara fantomskog uspona. **Ovo nisam izmerio** — `elevation_gain_meters` se ne šalje
na server pa nema podataka. Predlog: uporediti prijavljeni uspon na jednom ravnom runu sa stvarnim, i ako
se potvrdi, podići prag na 3–5 m ili provući visinu kroz klizni prosek pre poređenja.

## P2-6. RLS dozvoljava zaobilaženje validacije u RPC-u

`running_sessions_insert_own` / `running_sessions_update_own` dozvoljavaju `authenticated` klijentu da
direktno piše i menja svoje redove. Sve provere iz `sync_completed_running_session` (razumna distanca,
`moving ≤ duration`, validne koordinate, zabrana anonimnog naloga) se time zaobilaze običnim PostgREST
pozivom. Danas nije problem jer runovi ne hrane ništa zajedničko; postaje problem u trenutku kad se uvedu
rang liste ili izazovi nad `distance_meters`. Razmisliti o ukidanju direktnog `INSERT`/`UPDATE` grant-a i
ostavljanju samo RPC putanje.

## P2-7. Sitnice

- `RunMetricsAccumulator._copyWith` (`run_tracking.dart:558`) koristi `??` za svako polje, pa se nijedno
  polje ne može vratiti na `null`. Posledica: `pendingJumpPoint` se nikad ne briše posle uspešnog
  re-sidrenja. Trenutno bezopasno jer ga `maximumReliableSampleGap` provera odbija kao ustajalog, ali je
  zamka za sledeću izmenu.
- `run_analysis.dart` — `averageSpeedMps` nema donji prag, dok `averagePaceSecondsPerKilometer` traži
  ≥100 m. Run od 11 m prikazuje brzinu `0.7 km/h` i tempo `--:--` jedno pored drugog
  (run `id = 12` u bazi je upravo takav).
- `drift_run_store.dart:206-224` — `insertRestoredRun` ne prenosi `speedAccuracyMps` (restore-ovane tačke
  ionako nemaju brzinu, ali je asimetrija u odnosu na `saveObservation`).
- `running_database.dart:458` — pruning interrupcija briše `detectedAt < boundary.detectedAt`; kod
  jednakih vremena zadrži više od `keepMostRecent` redova.
- `RunNotificationChannel.registerDismissalReceiver` (Kotlin) registruje `BroadcastReceiver` na
  application context i nikad ga ne odjavljuje.
- `_DistanceBarsPainter.shouldRepaint` poredi liste operatorom `!=` — Dart liste nemaju value equality, pa
  se uvek prijavljuje "promenjeno". Sitno, ali `RunningStatistics.calculate` pravi novu listu na svakom
  build-u pa se chart uvek preslikava.

---

## Šta je dobro rešeno (da se ne pokvari pri refaktoru)

- `speedBudget` (Doppler integral kao dvostrana granica nad razlikom pozicija) je ozbiljno i tačno rešenje
  problema pristrasnosti GPS-a; on je jedini razlog što P0-1 nije bio katastrofalan.
- WAL + `synchronous = NORMAL` na lokalnoj bazi — tačna odluka za 1 Hz upis sa ugašenim ekranom.
- 512 px pločice sa `zoomOffset: -1` + trajni disk keš — direktno smanjuje Mapbox račun.
- Sekvencijalni `_writeQueue` sa generacijama i `_transitionCapturedElapsed` — recovery i prelazi su
  pažljivo urađeni, tu nisam našao rupu.
- `ACCESS_BACKGROUND_LOCATION` namerno izostavljen, sa obrazloženjem u manifestu — izbegava Play review.
- Anon `EXECUTE` je revoke-ovan na sva tri RPC-a; RLS politike su korektne.
- Interrupcioni log + battery advisor: dijagnostika koja se ne meša u snimanje.

---

## Predloženi redosled

1. P0-1 (jedan red koda, najveći uticaj na tačnost) + parametrizovati accuracy test po širini.
2. P0-2 (`clamp` na `p_moving_duration_seconds`) — trenutno tiho gubi ~12% runova sa naloga.
3. P0-3 (odluka: uključiti notifikaciju ili skloniti podešavanje).
4. P1-1 (restore samo kad treba) — najveća ušteda po korisniku.
5. P1-8 / P1-9 (`.distinct()` + suziti `AnimatedBuilder`) — najveća ušteda baterije tokom runa.
6. P1-2 / P1-3 / P1-4 (jedna migracija: obrisati suvišan indeks, `route_geom` + GiST, mrtve kolone).
7. P1-10 (elevacija na `local_runs` pri `finish()`), pa ostalo.
