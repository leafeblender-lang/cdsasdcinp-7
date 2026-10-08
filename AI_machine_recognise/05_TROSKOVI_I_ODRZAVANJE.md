# Troškovi, nalozi i održavanje

[Početna strana](README.md) · [Kako trenirati](03_TRENING_KORAK_PO_KORAK.md)

## Koje troškove trenutno ima ova funkcionalnost?

Za postojeće prepoznavanje nema plaćenog AI API-ja, servera za inference ili naplate po skeniranju. Trening se radi na računaru koji već poseduješ, a model se izvršava na telefonu korisnika.

| Stavka | Trenutno stanje |
|---|---|
| AI API pozivi | Ne koriste se; nema računa po pozivu. |
| Mesečna AI pretplata | Nije uvedena. |
| Iznajmljivanje GPU-a | Nije korišćeno; trening koristi tvoju RTX 3060. |
| AI backend server | Nije uveden. |
| Cloud baza slika/storage | Nije uvedena za ovu funkciju. |
| Lokalna SQLite baza | Besplatna; `dataset/catalog.sqlite3`, bez servera i naloga. |
| Dodatna plaćena softverska licenca | Nije kupljena niti aktivirana za ovu implementaciju. |
| Struja računara | Tvoj stvarni trošak rada računara tokom treninga. |
| Disk i backup | Lokalni prostor; dodatni disk/cloud backup može imati cenu ako ga izabereš. |
| Internet | Potreban za početna preuzimanja biblioteka/težina i distribuciju aplikacije. |
| Rad telefona | Procesor, memorija i baterija korisnikovog uređaja. |
| Prikupljanje i označavanje podataka | Tvoje vreme; eventualno tuđi rad ako ga angažuješ. |

„Bez plaćenog AI-ja“ ne znači da cela fitnes aplikacija nema nikakve druge troškove. Postojeći backend, mape, nalozi, distribucija aplikacije ili ostali servisi mogu imati sopstvene uslove i račune. Ovim radom za skener nisu dodati novi takvi servisi.

## Koliko košta jedno skeniranje?

U sadašnjoj arhitekturi nema fakturisanog AI poziva za jedno skeniranje. Sto, hiljadu ili više korisničkih skeniranja ne stvara naplatu GPU rada na tvom računaru, jer se inference obavlja na njihovim telefonima.

Trošak distribucije aplikacije/modela i održavanja ipak zavisi od načina na koji aplikaciju isporučuješ. Ako bi se model kasnije preuzimao sa tvog servera, trebalo bi posebno računati skladište i saobraćaj. Takav sistem trenutno nije implementiran.

## Kako da proceniš struju za trening

Računica je:

```text
energija u kWh = prosečna potrošnja celog računara u kW × trajanje treninga u satima
cena = energija u kWh × tvoja cena jednog kWh
```

Samo ilustracija: ako ceo računar prosečno troši 0,25 kW tokom 30 minuta, potrošnja je 0,125 kWh. Pomnoži to cenom sa svog računa. To nije izmerena potrošnja ovog treninga niti podatak o tvojoj tarifi. Ukupna potrošnja računara nije isto što i deklarisana snaga grafičke karte.

Na ovom veoma malom skupu lokalni trening je kratak nakon početnih preuzimanja. Vreme raste sa količinom podataka, augmentacijama i složenošću modela; nema fiksne cene ili garantovanog trajanja svakog budućeg treninga.

## Koji nalozi su potrebni?

Za sadašnji trening i korišćenje skenera nisu potrebni OpenAI nalog/API ključ, Colab prijava, nalog na servisu za trening modela niti nalog za cloud bazu slika.

Potrebni su instalirani alati na računaru i dozvola za kameru na telefonu. Početne težine i biblioteke preuzimaju se sa javnih izvora. Kada su lokalno dostupne, sam trening nad postojećim podacima i prepoznavanje mogu raditi bez novog AI mrežnog zahteva.

Nalozi i troškovi prodavnica aplikacija, ako ih budeš koristio za objavljivanje, odvojeni su od AI funkcije. Ovde nisu navedene njihove cene jer nijedan takav nalog/plan nije uveden ovim radom i njihove aktuelne uslove treba proveriti pri objavljivanju.

## Koliko prostora zauzima?

Trenutni model ima približno **3,8 MB**, a četiri organizovana videa približno **60,7 MB**. Tome se dodaju JPG kadrovi, originalne kopije snimaka, biblioteke, keševi i razvojni fajlovi.

Napravljen kompletan debug APK ima 270.930.826 bajtova, približno **271 MB**. To je cela postojeća fitnes aplikacija sa Flutter/native bibliotekama i ostalim resursima, a ne veličina AI modela. Optimizacija veličine release izdanja je zaseban posao; nije obavljena u okviru ove funkcionalnosti.

Ako čuvaš svaku verziju modela i mnogo videa, skladište će postepeno rasti. Čuvaj verzije koje su ti važne i vodi poreklo podataka. Ne briši jedini primer trening skupa zato što model već postoji: model nije zamena za originalne podatke potrebne za budući trening.

SQLite katalog čuva metapodatke i ne duplira fotografije. Za 10.000 slika prostor prvenstveno zavisi od samih fajlova: prosečno 0,5 MB po slici znači oko 5 GB fotografija, bez videa i backup-a. To je primer računice, ne izmerena veličina budućeg dataseta. Za backup baze i slika vidi [dokument 06](06_LOKALNA_BAZA.md).

## Održavanje koje ti ostaje

Pri novom snimanju proveri klasu i naziv sesije, pregledaj izdvojene fotografije i napravi backup važnih podataka. Pri novom treningu čuvaj ceo dataset poznatih klasa, izveštaj i par ONNX/JSON koji mu odgovara.

Pre zamene modela dovoljno je za početak napraviti nekoliko smislenih proba novim fotografijama, uključujući druge sprave. Veće testiranje ostavljeno je za kasnije po dogovoru, ali rezultat treninga ne treba predstavljati kao dokaz pouzdanosti na neviđenim spravama.

Kada dodaš nove klase, ažuriraj tekst u aplikaciji koji opisuje podržane sprave. Kada promeniš model ili pragove, napravi i isporuči novu verziju aplikacije. Korisničko skeniranje samo po sebi ne dopunjava dataset.

Prilikom nadogradnje Flutter-a ili ONNX paketa proveri ponovo da se Android build pravi. Sadašnji APK je uspešno napravljen na AGP 9; Flutter alat je ispisao i upozorenje vezano za detekciju Kotlin Gradle plugina. To nije sprečilo ovaj build, ali ne treba unapred pretpostaviti kompatibilnost sa svakom budućom verzijom.

## iPhone i druge platforme

Izabrani paket u trenutnoj konfiguraciji zahteva iOS 16+, što je podešeno u projektu. Na ovom Windows računaru nije napravljen iOS build. Za izgradnju/proveru na iPhone-u potreban je odgovarajući Mac/Xcode postupak; eventualni nabavljeni hardver ili izabrani build servis bili bi odvojeni troškovi.

Ako je podrška za iOS 13–15 obavezna, treba promeniti nativnu integraciju/model runtime pre izdavanja za te uređaje. Dokumentacija ne tvrdi da sadašnja aplikacija za skeniranje radi na tim starijim verzijama.

## Ako kasnije odlučiš da uvedeš cloud

Moguće buduće stavke bile bi skladište slika, kontrola pristupa, prenos podataka, GPU vreme za trening, čuvanje verzija modela i njihovo preuzimanje. To su posebne odluke i nisu preduslov da trenutni skener radi.

Jednostavno arhiviranje podataka na drugi disk može biti dovoljno za početak. Cloud ima smisla kada želiš saradnju više ljudi, kontrolisano prikupljanje novih primera ili automatizovanu distribuciju modela; sam po sebi neće popraviti pogrešno označene ili nedovoljno raznovrsne slike.
