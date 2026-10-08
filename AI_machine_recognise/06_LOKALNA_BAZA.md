# Lokalna SQLite baza za slike

[Početna strana](README.md) · [Slike i imenovanje](02_PODACI_I_FOLDERI.md) · [Trening](03_TRENING_KORAK_PO_KORAK.md)

## Šta je napravljeno

Napravljena je **besplatna lokalna SQLite baza**, bez servera, naloga ili pretplate:

```text
C:\Users\veljk\Desktop\FitnesApp\dataset\catalog.sqlite3
```

Upisano je 97 postojećih slika: 29 za `converging_chest_press` i 68 za `plate_loaded_seated_row`, iz četiri snimka. Pri uvozu nema oštećenih/nedostajućih slika niti identičnih duplikata. Provereni su integritet baze i učitavanje podele 46 trening / 51 validaciona slika. Model nije ponovo treniran niti zamenjen.

Fotografije ostaju u `dataset/raw/<klasa>/`. Baza čuva njihove putanje i podatke, **ne kopije fotografija**. Ne pomera niti briše originalne slike.

SQLite odgovara lokalnom katalogu sa 10.000+ zapisa; nije potreban MySQL/PostgreSQL server. Broj slika nije unapred ograničen na 10.000. Brzina indeksiranja zavisi od diska i veličine fotografija; nije rađeno veliko merenje sa 10.000 slika. [SQLite: preporučene primene](https://www.sqlite.org/whentouse.html).

Koristi se Python-ov ugrađeni `sqlite3`, u ovom okruženju SQLite 3.37.2. Nije instaliran dodatni server. [Python sqlite3 dokumentacija](https://docs.python.org/3.10/library/sqlite3.html).

## Dodavanje novih slika

Stavi označene slike neposredno u folder klase, npr.:

```text
dataset/raw/leg_press/gym02_s01_000001.jpg
dataset/raw/leg_press/gym02_s01_000002.jpg
dataset/raw/leg_press/gym02_s02_000001.jpg
```

Podržani su JPG/JPEG, PNG i WebP. HEIC prethodno konvertuj; preimenovanje ekstenzije nije konverzija. Naziv prati `<teretana>_sNN_<broj>`. ID teretane koristi slova, cifre i eventualno crticu, bez donje crte. Koristi dosledno dvocifrene ili trocifrene sesije (`s01` ili `s001`). Svaka sesija predstavlja jedan stvarni video ili fotografisanje, ne veštačku podelu susednih kadrova.

Pokreni u PowerShell-u:

```powershell
Set-Location -LiteralPath 'C:\Users\veljk\Desktop\FitnesApp\fitness_plus'
.\.venv-ai\Scripts\python.exe tool/ai_catalog.py sync
.\.venv-ai\Scripts\python.exe tool/ai_catalog.py stats
```

Neispravna imena/oštećene slike se evidentiraju kao nevažeći primeri i ne koriste za trening. Podfolderi unutar klase i nepodržani formati se ne uvoze. Oznake sprava moraš sam proveriti.

Za nove videe ostaje `.\.venv-ai\Scripts\python.exe tool/ai_extract_frames.py`: ekstrakcija sada automatski osvežava postojeću bazu. Trening takođe prvo osveži katalog. **Nema pozadinskog praćenja foldera:** promene ulaze u bazu pri `sync`, ekstrakciji ili treningu.

Ponovljeni `sync` ne duplira zapise i čuva ručna isključenja/podele. Neizmenjene slike prepoznaje preko veličine i vremena izmene. `sync --rehash` ponovo čita sve fajlove. Nestali fajlovi dobijaju oznaku `missing`, bez brisanja zapisa.

## Šta se čuva u bazi

| Tabela/pregled | Sadržaj |
|---|---|
| `classes` | ID i naziv klase/sprave. |
| `sources` | Teretana, sesija, podela skupa, video ako postoji, URL, licenca i beleška. |
| `images` | Relativna putanja, SHA-256, dimenzije, veličina, vreme izmene, ispravnost, prisutnost i uključenost. |
| `image_catalog` | Spojen pregled za čitanje i CSV izvoz. |

URL/licenca se ne pronalaze automatski. Komanda `source` podržava `--url`, `--license` i `--notes`; podatke upisuj tek kada proveriš izvor. Poreklo je na nivou sesije: razdvoji izvore sa različitim autorima/uslovima; autora navedi u belešci. `display_name` u bazi još ne određuje naziv u aplikaciji: za njega koristi `NAMES` u trening skripti, kao u dokumentu 03.

## Pregled i isključivanje loših primera

```powershell
.\.venv-ai\Scripts\python.exe tool/ai_catalog.py images --limit 20
.\.venv-ai\Scripts\python.exe tool/ai_catalog.py images --status invalid
.\.venv-ai\Scripts\python.exe tool/ai_catalog.py duplicates
```

Za isključivanje slike zameni `123` stvarnim ID-jem iz pregleda:

```powershell
.\.venv-ai\Scripts\python.exe tool/ai_catalog.py set-enabled --id 123 --value 0
```

Za vraćanje koristi `--value 1`. Fotografija ostaje na disku u oba slučaja.

Duplikati se prepoznaju po identičnim bajtovima, ne po vizuelnoj sličnosti. Komanda za pregled uključuje i isključene kopije. Trening identične aktivne kopije iste klase/podele koristi samo jednom. Ako ista aktivna slika pripada različitim klasama ili podelama, trening staje da ne bi koristio pogrešne oznake ili proveravao sam sebe. Isključi nepravilno smeštenu kopiju i pokušaj ponovo.

## Podela trening / validacija / test

Svaki izvor ima `split`: `auto`, `train`, `validation`, `test` ili `exclude`. Podela važi za celu sesiju jedne klase, nikada nasumično za susedne kadrove istog videa.

Početni uvoz ostavio je sve na `auto`: poslednji izvor po tekstualnom sortiranju teretane/sesije ide u validaciju, ostali u trening. Ako postoji eksplicitna validacija, ostali `auto` izvori te klase idu u trening. Za stabilnu validaciju pri budućem dodavanju snimaka možeš pokrenuti:

```powershell
.\.venv-ai\Scripts\python.exe tool/ai_catalog.py source --label converging_chest_press --gym gym01 --session s02 --split validation
.\.venv-ai\Scripts\python.exe tool/ai_catalog.py source --label plate_loaded_seated_row --gym gym01 --session s02 --split validation
```

Ove komande su primeri, nisu primenjene prilikom uvoza. `exclude` isključuje celu sesiju. `test` je zadržava u manifestu, ali ne koristi za trening niti izbor epohe; automatsko ocenjivanje test skupa još nije implementirano. Svaka trenirana klasa mora imati zasebne aktivne trening i validacione izvore; potrebne su bar dve poznate klase.

## Trening sa bazom

Postojeće komande treninga nastavljaju da rade: baza se automatski pronalazi i poštuju se njena isključenja/podele. Za novi model koristi odvojenog kandidata iz [dokumenta 03](03_TRENING_KORAK_PO_KORAK.md).

Kratka provera podele bez treninga i zamene modela:

```powershell
.\.venv-ai\Scripts\python.exe tool/ai_train_local.py --inspect-only --output artifacts/equipment_scan/catalog_inspect
```

Manifest sada sadrži i `catalog_image_id` i SHA-256 slike. `test` slike su u manifestu, ali nisu na kontakt-listu trening/validacionih slika.

Za drugi komplet podataka treningu možeš dati `--database 'D:\MojDataset\catalog.sqlite3'`; slike tada moraju biti u `D:\MojDataset\raw`. Komanda kataloga prihvata `--database` **pre** podkomande, npr. `tool/ai_catalog.py --database 'D:\MojDataset\catalog.sqlite3' sync`. `--folders-only` na treningu zaobilazi bazu i njena isključenja/podele: nemoj ga koristiti u redovnom toku; taj stari tok podržava samo `.jpg`.

**Baza nije model.** Telefon koristi samo ONNX i konfiguraciju, ne ovaj katalog ni trening fotografije. Novi podaci utiču na aplikaciju tek posle novog treninga i isporuke modela. Raznovrsnost i tačne oznake važnije su od samog broja slika. Sadašnji model čuva vektore trening slika, pa veći skup povećava model i obradu; kapacitet SQLite baze i brzina modela su odvojene stvari.

## Backup i izvoz

```powershell
$aiCatalogStamp = Get-Date -Format 'yyyyMMdd_HHmmss'
.\.venv-ai\Scripts\python.exe tool/ai_catalog.py backup --out "../dataset/backups/catalog_$aiCatalogStamp.sqlite3"
.\.venv-ai\Scripts\python.exe tool/ai_catalog.py export --out "../dataset/backups/catalog_$aiCatalogStamp.csv"
```

Postojeći odredišni fajl neće biti prepisan. CSV je za pregled, ne zamena za bazu; CSV import nije implementiran. Backup koristi [SQLite backup mehanizam](https://www.sqlite.org/backup.html), a ne kopiranje otvorenog fajla naslepo.

**Backup baze ne sadrži slike.** Sačuvaj i `raw/`, `raw_video/`, modele i izveštaje, po mogućnosti na drugi disk. Kopija u `dataset/backups` nije zaštita od kvara istog diska. Pri prenosu zadrži raspored: baza pored `raw/`. Za doslednu kopiju podataka zaustavi uvoz/menjanje slika dok praviš i prenosiš backup.

Baza koristi WAL: `catalog.sqlite3-wal` i `catalog.sqlite3-shm` su normalni pomoćni fajlovi; ne briši ih dok je baza otvorena. Koristi backup komandu iznad. Aktivna baza treba da bude na lokalnom disku, ne kao istovremeno otvoren deljeni fajl preko mreže/cloud sinhronizacije.

## Opcioni program sa prozorom i troškovi

Besplatan grafički preglednik je [DB Browser for SQLite](https://sqlitebrowser.org/). **Nije instaliran ovim radom.** U njemu možeš otvoriti bazu i pogled `image_catalog`. Za promene koristi navedene komande; ne menjaj ručno ID-jeve, putanje, hash vrednosti ili strukturu tabela.

Trošak baze je **0 za softver/server/nalog**. Troši prostor na tvom disku. Struja, dodatni disk i eventualni izabrani cloud backup su odvojeni troškovi.
