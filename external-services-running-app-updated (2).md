# Spoljni servisi i planovi za running aplikaciju

Ovaj fajl je praktična checklista: gde treba da napraviš nalog, koji plan ima smisla za tvoj trenutni stage (MVP, par korisnika), i da li je sve što je do sada planirano u okviru free tier-a.

**Poslednje ažuriranje statusa: 4. avgust 2026.**

## 1. Supabase (backend, baza, auth, Realtime)

**Šta koristiš od Supabase-a:**
- Postgres baza sa PostGIS ekstenzijom (geolokacija, rute).[web:77]
- Auth (email/password ili OAuth).[web:226]
- Storage (eventualno za slike avatara, exporte).[web:34]
- Realtime (broadcast live lokacije kroz privatne kanale).[web:34]
- Edge Functions ili RPC funkcije za geoupite i social logiku.[web:34]

**Plan i trenutni status:**
- [x] Napravljena je Supabase organizacija i razvojni projekat `FitnessApp`.
- [x] Projekat koristi Free plan za trenutnu MVP fazu.
- [x] Postavljeni su Postgres/PostGIS, privatna šema, RLS, running/social/notification tabele, RPC funkcije, Vault, `pg_net` webhook i Edge Function.
- [x] Serverska arhitektura je proverena transakcionim i webhook testovima.
- [ ] Kasnije napraviti odvojen produkcioni projekat pre javnog lansiranja.
- Free plan trenutno obuhvata 500 MB baze, 1 GB storage-a, 50.000 MAU i 2 free projekta.[web:34][web:38]
- Ovo je sasvim dovoljno za rani razvoj, testiranje i mali broj realnih korisnika; sve što smo do sada dizajnirali staje u Free ako ne čuvaš sirove GPS tačke predugo.[web:38][web:236]

**Kad razmišljaš o Pro planu (~25 USD/month):**
- kada baza počne da prelazi 500 MB (više korisnika, mnogo istorije trčanja).[web:34][web:36]
- kada želiš ozbiljniju produkciju (više resursa, veće diskove, jače Realtime limite).

Trenutno: **Supabase nalog, organizacija, projekat i serverska arhitektura su završeni; sledeće je povezivanje Flutter klijenta sa RPC funkcijama i FCM registracijom uređaja.**

## 2. Mapbox (tiles za mapu)

**Šta koristiš od Mapbox-a:**
- Mapbox Studio za pravljenje **light/dark stilova**.[web:227]
- Static Tiles API za third-party integraciju sa `flutter_map` (URL za 256 tile-ove od Studio stila).[web:154][web:233]

**Plan za nalog:**
- Napravi **Mapbox nalog**.
- Koristi default **Free tier** – Mapbox ima besplatan nivo sa više ograničenja po API-ju.[web:108][web:238]
- Za Static / Raster / Vector tiles postoje različite free granice, npr. trenutno:
  - Static Tiles API do ~200.000 tile zahteva mesečno free.[web:238]
  - Raster Tiles API do ~750.000 tile zahteva mesečno free.[web:238]
- Tvoj MVP (par korisnika koji testiraju) je daleko ispod ovih limita, tako da je sve što smo planirali i dalje u okviru **Free tier-a**, pod uslovom da ne radiš masivni offline bulk download.

**Kasniji upgrade:**
- Ako ti Mapbox dashboard počne da pokazuje da prelaziš free granice, prelazak na plaćeni usage model je automatski – nema posebnog "Starter" plana, već plaćaš po potrošnji. To dolazi tek kada imaš dosta aktivnih korisnika.[web:108]

Za sada: **Mapbox nalog + Free usage**, vezi karticu samo ako moraju za API ključ, ali je praktično da imaš odmah.

## 3. Apple developer nalog (Apple Watch + iOS distribucija)

**Za razvoj na telefonu i emulatoru:**
- Za **lokalni razvoj** i testiranje na sopstvenom iPhone-u možeš da koristiš besplatan Apple developer nalog (Apple ID + Xcode), bez plaćanja godišnje članarine.[web:84]

**Za objavu aplikacije i distribuciju na App Store-u + ozbiljnu Watch integraciju:**
- Potreban je **Apple Developer Program** nalog.
- Cena je **99 USD godišnje**.[web:239]

Sve što si do sada planirao se može razvijati i testirati **besplatno** (lokalno, TestFlight preko shared naloga), ali za javnu objavu i ozbiljnu distribuciju planiraj da će ti trebati 99 USD/god.

Za sada: **kreiraj običan Apple ID, Xcode okruženje i besplatan developer nalog**; plaćeni Apple Developer Program može da sačeka dok ne budeš bliže objavi.

## 4. Google nalog (Android + Wear OS + Health Connect)

Za Android telefon, Wear OS sat i Health Connect:
- Potreban ti je **Google nalog** – verovatno ga već imaš.[web:83]
- Za objavu aplikacije na Play Store-u treba **Google Play Console nalog** (jednokratna naknada oko 25 USD), ali sama upotreba SDK-ova (Health Services, Health Connect) je besplatna.[web:83][web:58]

Za MVP i lokalni test:
- možeš sve da razvijaš i testiraš **besplatno** na lokalnim uređajima i emulatorima.

Za sada: dovoljno je da imaš regularan Google nalog. Play Console može da sačeka dok ne budeš spreman za objavu.

## 5. Firebase Cloud Messaging (push notifikacije)

Firebase se u ovom projektu ne koristi kao glavni backend. Supabase ostaje zadužen za bazu, Auth, RPC funkcije, Edge Functions, Realtime i kompletnu social logiku.

Firebase se koristi samo za **Firebase Cloud Messaging (FCM)**, odnosno kao transportni servis koji dostavlja sistemske push notifikacije telefonu kada je aplikacija u pozadini ili ugašena.

**Šta koristiš:**
- FCM za dostavljanje push notifikacija na Android;
- FCM kao zajednički aplikativni kanal za iOS, uz povezivanje sa Apple Push Notification service-om (APNs);
- `firebase_messaging` Flutter paket za registraciju uređaja, osvežavanje tokena i prijem poruka.

**Tok slanja:**

```text
Supabase Postgres / RPC
        ↓
notification event
        ↓
Supabase Database Webhook
        ↓
Supabase Edge Function
        ↓
Firebase Cloud Messaging
        ↓
Android / iOS uređaj
```

FCM se koristi za funkciju u kojoj konektovani prijatelji mogu da dobiju obaveštenje kada korisnik završi trčanje, uz globalno i selektivno uključivanje ili isključivanje.

Ne koriste se:
- Firebase Authentication;
- Firestore ili Realtime Database;
- Firebase Storage;
- Firebase kao zamena za Supabase.

Za iOS će kasnije biti potrebno povezati APNs podatke sa Firebase projektom i uključiti push capabilities u Apple projektu.

**Trenutni status Firebase/FCM integracije:**

- [x] Napravljen je Firebase projekat.
- [x] Firebase Cloud Messaging API V1 je uključen.
- [x] Generisan je Firebase service-account JSON.
- [x] JSON je bezbedno sačuvan kao Supabase Edge Function secret `FIREBASE_SERVICE_ACCOUNT_JSON` i nije dodat u Flutter kod ili repozitorijum.
- [x] Supabase webhook uspešno poziva Edge Function i funkcija prepoznaje Firebase konfiguraciju (`200 OK`).
- [ ] Dodati `firebase_core` i `firebase_messaging` u Flutter.
- [ ] Dodati Android aplikaciju u Firebase i postaviti `google-services.json` u odgovarajuće Flutter/Android mesto.
- [ ] Preuzimati i osvežavati FCM token i slati ga u Supabase kroz `register_push_device()`.
- [ ] Uraditi end-to-end test stvarne notifikacije na fizičkom Android uređaju.
- [ ] Kasnije povezati APNs za iOS i testirati na iPhone uređaju.

## 6. Mobilni SDK-ovi i Flutter paketi

Ovo su **biblioteke**, ne spoljni SaaS servisi – ne treba poseban nalog ili plan:

- `geolocator` – paket sa pub.dev, besplatan.[web:47]
- `drift` – ORM za lokalnu bazu, open-source.[web:52]
- `flutter_map` – open-source.[web:113]
- `flutter_map_tile_caching` / `flutter_map_cache` – dodatak za keširanje tile-ova, open-source.[web:184][web:190]
- `health` – paket za HealthKit + Health Connect, besplatan.[web:16]
- `watch_connectivity` – paket za Apple Watch/phone komunikaciju, besplatan.[web:92]
- `firebase_messaging` – Flutter paket za registraciju uređaja i prijem FCM push notifikacija.

Ovde nema naloga ni planova – samo ih dodaješ u `pubspec.yaml`.

## 7. Ostali potencijalni servisi (za kasnije)

Pored FCM-a, za ovaj projekat zasad **nemaš obavezne** dodatne SaaS servise kao što su Sentry, LogRocket ili Stripe. Ako ih kasnije dodaš:

- **Crash reporting / monitoring** (Sentry, Firebase Crashlytics) – oba imaju free tier za male projekte.[web:241]
- **Plaćanja** (Stripe) – nema mesečne naknade, već se naplaćuje po transakciji.

To je opcioni deo za kasnije.

## 8. Rezime – trenutni status i sledeći koraci

Status servisa:

1. **Supabase** — [x] razvojni projekat i serverska running/social/notification arhitektura završeni.
2. **Firebase Cloud Messaging** — [x] projekat, FCM V1, service account i Supabase secret završeni; [ ] Flutter klijent i stvarni push test nisu završeni.
3. **Mapbox** — koristiti Free usage za Static Tiles API + Studio stilove; konkretan Flutter/map status proveriti odvojeno.
4. **Apple** — besplatan developer nalog je dovoljan za rani razvoj; APNs i plaćeni program ostaju za kasnije.
5. **Google/Play Console** — običan Google nalog je dovoljan za razvoj; Play Console je potreban tek za objavu.

Sve što je do sada planirano je u praksi i dalje u okviru **free / vrlo niskog troška**, pod uslovom da ne preteraš sa retention-om sirovih GPS tačaka i da ne praviš masivni offline download tile-ova. Supabase ostaje glavni backend, dok FCM služi samo kao kanal za dostavljanje push notifikacija.
