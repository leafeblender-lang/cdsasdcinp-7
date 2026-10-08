# Gde su slike, snimci i model

[Početna strana](README.md) · [Sledeće: trening](03_TRENING_KORAK_PO_KORAK.md)

## Da li postoji baza slika?

**Postoji lokalna SQLite baza `dataset/catalog.sqlite3`, dok su fotografije i dalje lokalni fajlovi.** Baza čuva putanje, klase, izvore, hash vrednosti i odluke o uključivanju/podeli; ne čuva same slike. Nema cloud bucket-a niti automatskog slanja fotografija. Komande za upravljanje su u [dokumentu 06](06_LOKALNA_BAZA.md).

Glavni folder sa podacima je:

```text
C:\Users\veljk\Desktop\FitnesApp\dataset
```

`raw_video` sadrži organizovane snimke, a `raw` sadrži fotografije koje trening stvarno čita. Folder klase predstavlja oznaku slike: slika u `raw/converging_chest_press` smatra se primerom te sprave. `catalog.sqlite3` se osvežava komandom `sync`, posle ekstrakcije videa i pre treninga; katalog i fotografije se ne prate u pozadini.

## Originalni snimci i njihove organizovane kopije

Originali se nalaze u:

```text
C:\Users\veljk\Desktop\converging chest press machine\
    20260901_201100.mp4
    20260901_201115.mp4

C:\Users\veljk\Desktop\plate-loaded seated row machine\
    20260901_200521.mp4
    20260901_200654.mp4
```

Kopije za postojeći postupak obrade nalaze se u:

```text
C:\Users\veljk\Desktop\FitnesApp\dataset\raw_video\gym01\
    converging_chest_press_s01.mp4
    converging_chest_press_s02.mp4
    plate_loaded_seated_row_s01.mp4
    plate_loaded_seated_row_s02.mp4
```

Ta četiri organizovana videa ukupno imaju približno 60,7 MB. Ovo ne uključuje originale, izdvojene slike, Python biblioteke ili aplikaciju.

Skripte automatski obrađuju `dataset/raw_video`; ne prate originalne desktop foldere. Ako dodaš novi video samo u originalni folder, treba ga kopirati i pravilno imenovati u `raw_video`.

## Gde su izdvojene fotografije

```text
C:\Users\veljk\Desktop\FitnesApp\dataset\raw\
├── converging_chest_press\       29 JPG slika
│   ├── gym01_s01_000015.jpg
│   └── ...
└── plate_loaded_seated_row\      68 JPG slika
    ├── gym01_s01_000015.jpg
    └── ...
```

Trening preko baze podržava JPG/JPEG, PNG i WebP direktno u folderu klase. Ne prolazi kroz dodatne podfoldere klase. HEIC prethodno konvertuj. Stari opcioni `--folders-only` tok koristi samo `.jpg` i zaobilazi odluke iz baze.

## Značenje naziva fajlova

Video se imenuje po obrascu:

```text
dataset/raw_video/<teretana>/<klasa>_<sesija>.mp4

Primer: dataset/raw_video/gym02/leg_press_s03.mp4
```

| Deo | Značenje |
|---|---|
| `gym02` | Identifikator druge teretane. |
| `leg_press` | Stabilan ID klase, malim slovima i sa donjim crtama. |
| `s03` | Treća odvojena sesija/snimak te klase u toj teretani. |

Za JPG kadrove obrazac je:

```text
dataset/raw/<klasa>/<teretana>_<sesija>_<broj_kadra>.jpg

Primer: dataset/raw/leg_press/gym02_s03_000120.jpg
```

Broj na kraju označava kadar; za ručno dodate fotografije može biti redni broj. Koristi dosledne ID-jeve teretana bez donjih crta, npr. `gym01`, `gym02`.

Za nove sesije koristi `s03`, `s04` itd. Ne zamenjuj drugi video istim nazivom klase/sesije, jer isti naziv određuje izlazne JPG fajlove. Skripta pri uobičajenom ponovnom pokretanju preskače sesiju za koju već postoje izdvojene slike.

Koristi istu širinu brojeva, npr. `s01` do `s99`, ili dosledno `s001`, `s002`, ... od početka većeg skupa. Podrazumevana validaciona sesija bira se tekstualnim sortiranjem, ne po datumu snimanja iz videa. Baza omogućava i eksplicitno označavanje celih izvora za trening, validaciju, nezavisni test ili isključivanje.

## Kako dodavati samostalne fotografije

Možeš dodavati i prave fotografije, bez izvlačenja iz videa. Na primer:

```text
dataset/raw/leg_press/gym02_s01_000001.jpg
dataset/raw/leg_press/gym02_s01_000002.jpg
dataset/raw/leg_press/gym02_s02_000001.jpg
```

`s01` treba da označava jednu stvarnu sesiju fotografisanja, a `s02` drugu. Nemoj podeliti susedne kadrove istog snimka u lažne sesije samo da prođe uslov treninga: tada će rezultat validacije biti previše optimističan.

Ako fotografije već staviš u `raw`, za njih ne pokrećeš video-ekstrakciju. Pokreni `tool/ai_catalog.py sync`, pa trening; trening i sam osvežava bazu. PNG nije potrebno konvertovati kada koristiš bazu. HEIC konvertuj u podržani format; samo menjanje nastavka fajla nije konverzija.

## Kakvi podaci su korisni

Praktičan početni cilj za sledeće prikupljanje je nekoliko odvojenih snimanja svake klase, iz različitih uglova i po mogućnosti različitih dana. Za prepoznavanje čitavog tipa sprave treba postepeno uključiti različite primerke/proizvođače i teretane. To je smer za prikupljanje, a ne garantovani broj slika za određenu tačnost.

Snimi celu spravu sporo, tako da se vide sedište, ručke, oslonci i okvir. Uključi prednji pogled, bočni pogled i ugao između. Ne moraju svi kadrovi biti savršeni, ali ne treba da budu uglavnom pod, plafon ili druga sprava.

Više različitih snimanja obično donosi korisniju novu informaciju od velikog broja gotovo identičnih kadrova. Skripta zato uzorkuje oko dva kadra u sekundi, izbacuje jako zamućene i veoma slične kadrove, a dužu ivicu smanjuje na najviše 1024 piksela.

Model koristi centralni kvadrat slike. Ako se ceo važan deo sprave stalno nalazi van tog kvadrata, trening i kasnije skeniranje mogu biti loši iako cela originalna fotografija izgleda dobro.

## Šta pripada klasi `unknown`

Za sada `unknown` slike nisu prikupljene. Kasnije koristi:

```text
dataset/raw_video/gym01/unknown_s01.mp4
dataset/raw_video/gym01/unknown_s02.mp4
```

U njih mogu da uđu druge sprave koje model još ne prepoznaje, prazni prolazi, pod, zid i predmeti koji se često pojavljuju pri skeniranju. Najvredniji su i slični, zbunjujući primeri drugih sprava, a ne samo očigledno različit zid.

Poznate sprave ne treba da budu glavni objekat `unknown` slike. Kada kasnije neku spravu dodaš kao zasebnu poznatu klasu, pregledaj stare `unknown` primere i ispravi oznake da ista sprava ne ostane označena na dva suprotna načina.

Nasumične boje i šare nisu korišćene kao zamena za ove stvarne negativne primere.

## Model, izveštaji i Python okruženje

| Sadržaj | Lokacija u odnosu na `FitnesApp` | Da li ide u aplikaciju? |
|---|---|---|
| Organizovani video-snimci | `dataset/raw_video/` | Ne. |
| Trening/validacione slike | `dataset/raw/` | Ne. |
| SQLite katalog slika i izvora | `dataset/catalog.sqlite3` | Ne. |
| Model | `fitness_plus/assets/ml/equipment_scan/equipment.onnx` | Da. |
| Nazivi i pragovi | `fitness_plus/assets/ml/equipment_scan/config.json` | Da. |
| Rezultati po slici | `fitness_plus/artifacts/equipment_scan/evaluation.json` | Ne. |
| Podela podataka | `fitness_plus/artifacts/equipment_scan/split_manifest.json` | Ne. |
| Pregled izabranih fotografija | `fitness_plus/artifacts/equipment_scan/contact_sheet.jpg` | Ne. |
| Težine završnog sloja | `fitness_plus/artifacts/equipment_scan/head.pt` | Ne. |
| Fotografije za opcioni integracioni test, u Base64 obliku | `fitness_plus/artifacts/equipment_scan/smoke_inputs.json` | Ne u normalnom APK-u. |
| Python okruženje | `fitness_plus/.venv-ai/` | Ne. |

Preuzete početne MobileNet težine su u:

```text
C:\Users\veljk\.cache\torch\hub\checkpoints\mobilenet_v3_small-047dcff4.pth
```

To je osnovni model pre novog treninga, a ne rezultat treninga tvojih sprava. `head.pt` takođe nije samostalna zamena za ceo mobilni model: ne sadrži kompletan eksportovani model i sve njegove vizuelne predstavnike. Za aplikaciju čuvaj **ONNX i odgovarajući JSON zajedno**.

## Da li aplikacija skuplja nove podatke sama?

Ne. Skeniranjem na telefonu model se ne dopunjava i ne trenira. Kadrovi uživo obrađuju se u memoriji i ne šalju se u cloud. Odabrana fotografija takođe se ne dodaje automatski u dataset na računaru.

Sistemska kamera/image picker mogu imati privremeni lokalni fajl fotografije. To nije uvedena baza trening slika. Za novi trening trenutno ti ručno biraš i kopiraš nove primere na računar.

## Backup

Najvredniji sadržaj za čuvanje je `dataset`, zajedno sa modelima koje želiš da zadržiš i izvornim skriptama. Originalni desktop snimci i njihove organizovane kopije su na istom računaru: to štiti od pojedinačne greške pri radu, ali nije backup protiv otkaza diska.

Sačuvaj i SQLite katalog: u njemu su ručna isključenja i podele. Za konzistentnu kopiju baze koristi `tool/ai_catalog.py backup --out <nova-putanja>`, opisano u dokumentu 06. Sama baza ne sadrži fotografije, pa obavezno kopiraj i `raw`/`raw_video`.

`.venv-ai` i `artifacts/equipment_scan` izuzeti su iz Git praćenja. Dataset je izvan `fitness_plus` Git foldera. Nemoj pretpostaviti da su snimci i izveštaji sačuvani samo zato što je kod aplikacije sačuvan u Gitu. Kopiraj važne podatke na drugi disk ili u skladište koje sam izabereš.

## Mogući javni izvori slika za kasnije

Trenutni dataset sadrži tvoje snimke; za njega nisu preuzimane internet fotografije.

Ako kasnije želiš dodatne izvore, [Open Images](https://storage.googleapis.com/openimages/web/factsfigures.html) je opšti javni skup slika. Treba proveriti da li konkretni primeri i oznake odgovaraju tvojim vrstama sprava; nije već pripremljena baza baš za ove dve klase.

[Wikimedia Commons](https://commons.wikimedia.org/wiki/Commons:Reusing_content_outside_Wikimedia) omogućava pronalaženje fotografija sa navedenim uslovima korišćenja. Proveravaj uslove konkretnog fajla i sačuvaj izvor, autora i licencu uz preuzeti primer. Dostupnost slike na internetu sama po sebi nije dozvola za njeno korišćenje.

Internet fotografije ne treba nasumično mešati i deliti u trening i validaciju: iste ili skoro iste fotografije mogu se pojaviti na više mesta. Za realnu proveru tvoje aplikacije i dalje su potrebni novi snimci kakve će korisnici stvarno praviti.
