# Kako da nastaviš trening i ubaciš novi model u aplikaciju

[Početna strana](README.md) · [Podaci i imenovanje](02_PODACI_I_FOLDERI.md) · [Sledeće: tok aplikacije](04_TOK_APLIKACIJE_I_OGRANICENJA.md)

Sve komande u ovom dokumentu su za **PowerShell na tvom trenutnom računaru** i predstavljaju uputstvo za budući rad. Naknadno je uvedena [lokalna SQLite baza](06_LOKALNA_BAZA.md) i provereno je učitavanje podataka bez ponovnog treninga. Postojeći model nije promenjen.

## 1. Otvori pravi folder

```powershell
Set-Location -LiteralPath 'C:\Users\veljk\Desktop\FitnesApp\fitness_plus'
```

Koristi Python iz `.venv-ai`. Nije potrebno da aktiviraš okruženje ako ga pozivaš punom relativnom putanjom kao u primerima. Postojeći Flutter SDK je na `C:\development\flutter`, zato komande ispod koriste baš tu putanju.

## 2. Dodaj nove podatke, zadržavajući stare

Za dodatne snimke postojećih klasa dodaj, na primer:

```text
dataset/raw_video/gym01/converging_chest_press_s03.mp4
dataset/raw_video/gym01/plate_loaded_seated_row_s03.mp4
```

Ove putanje su u odnosu na `FitnesApp`, dakle `dataset` je pored `fitness_plus`.

Za drugu teretanu koristi `gym02`. Za drugi nezavisni snimak iste sprave koristi drugi broj sesije. Samostalne JPG fotografije možeš direktno dodati u `dataset/raw/<klasa>`, ali moraju pratiti imenovanje iz dokumenta 02.

**Ne izbacuj stare klase i slike zato što dodaješ novu spravu.** Skripta pri svakom pokretanju uzima prethodno naučeni osnovni MobileNet i iznova trenira novi završni sloj na celom tada dostupnom datasetu. Ona trenutno ne nastavlja trening iz prethodnog `equipment.onnx` ili `head.pt` fajla.

Ako novom treningu daš samo nove slike, stare slike nisu uključene u taj trening. Za jedan zajednički model svih sprava čuvaj podatke za sve te klase u `raw`.

## 3. Izvuci JPG kadrove iz novih videa

```powershell
.\.venv-ai\Scripts\python.exe tool/ai_extract_frames.py
```

Skripta čita `../dataset/raw_video` i piše u `../dataset/raw`. Već obrađene sesije obično preskače; za novi video koristi novi naziv sesije.

Po završetku automatski osvežava postojeći `catalog.sqlite3`. Za direktno dodate fotografije možeš sam pokrenuti `.\.venv-ai\Scripts\python.exe tool/ai_catalog.py sync`. Loš primer možeš isključiti bez brisanja komandom `set-enabled`, opisano u dokumentu 06.

Trenutni podrazumevani parametri su oko 2 kadra/s, duža ivica do 1024 px, filter jakog zamućenja i uklanjanje veoma sličnih kadrova unutar snimka. Zato 60 sekundi videa ne znači obavezno 120 sačuvanih slika.

Pregledaj slike u folderima. Pogrešnu spravu ne ostavljaj u folderu poznate klase samo zato što potiče iz njenog videa. Ukloni takav primer iz radnog dataseta ili ga pravilno označi, uz očuvanje originalnog snimka.

Za rutinsko dodavanje snimaka ne treba `--force`. Ta opcija ponavlja izdvajanje i može prepisati istoimene JPG fajlove, ali ne čisti automatski sve ranije izdvojene kadrove. Ako menjaš pravila ekstrakcije, koristi odvojen dataset i pregledaj rezultat pre zamene postojećeg.

## 4. Razumi kako se podaci dele

Trening automatski nalazi SQLite bazu pored `raw`, osvežava je i poštuje isključenja i podele izvora iz nje. Fotografije i dalje čita iz `raw`. **Za ovaj tok ne treba pokretati `ai_split_dataset.py` niti Colab notebook.** To su ranije pripremljene, odvojene opcije.

Ako su izvori ostavljeni na podrazumevanom `auto`, za svaku klasu skripta:

1. Iz imena JPG fajla čita izvor, npr. `gym01_s02`.
2. Tekstualno sortira jedinstvene izvore.
3. Poslednji izvor koristi za validaciju.
4. Sve ostale izvore te klase koristi za trening.

Primer: sa `gym01_s01`, `gym01_s02` i `gym01_s03`, `s03` postaje validacija, a `s01` i `s02` trening. To je izbor po imenu, ne po datumu u EXIF-u ili datumu izmene fajla.

Komandom `source --split` možeš postaviti `train`, `validation`, `test` ili `exclude` za celu sesiju. Ako postoji eksplicitna validacija, preostali `auto` izvori idu u trening. `test` se ne koristi za trening ni izbor modela; ostaje u manifestu, bez automatskog ocenjivanja. Detaljne komande su u dokumentu 06.

Svaka uključena klasa mora imati bar **dva stvarno odvojena izvora**. To važi i za `unknown` kada ga dodaš. Potrebne su bar dve poznate klase. Ovo su tehnički minimumi da skripta radi, a ne garancija dovoljnih podataka.

Skripta proverava i identične bajtove slike između treninga i validacije. Ipak, skoro isti kadar, ponovo kompresovana kopija ili ista sprava iz iste teretane i dalje mogu učiniti rezultat optimističnim.

Podela nije automatski podela po celim teretanama: ako ima više snimaka poslednje teretane, neki mogu ostati u treningu. Za proveru nove teretane eksplicitno izdvoji sve njene izvore. Slike za nezavisnu završnu proveru označi kao `test` u bazi ili ih drži van `raw`, npr. u `dataset/field_test`; trening skripta ih još ne ocenjuje automatski.

## 5. Preporučeni način: napravi kandidata za novi model

Ovaj postupak trenira u odvojene izlazne foldere i ostavlja trenutni model aplikacije na mestu.

```powershell
Set-Location -LiteralPath 'C:\Users\veljk\Desktop\FitnesApp\fitness_plus'

$aiRunName = Get-Date -Format 'yyyyMMdd_HHmmss'
$aiCandidateDir = Join-Path (Get-Location).Path "artifacts/equipment_scan/candidates/$aiRunName"
$aiCandidateModel = Join-Path $aiCandidateDir 'mobile'
$aiCandidateReports = Join-Path $aiCandidateDir 'reports'

.\.venv-ai\Scripts\python.exe tool/ai_train_local.py --output "$aiCandidateReports" --assets-dir "$aiCandidateModel"
if ($LASTEXITCODE -ne 0) { throw 'Trening ili izvoz nije uspeo. Nemoj zameniti model aplikacije.' }
```

Skripta sama pravi izlazne foldere. Na uspešnom završetku piše `Exported verified model:` i putanju. `mobile` sadrži ONNX i konfiguraciju, a `reports` izveštaj, podelu slika i pomoćne fajlove.

Pomenute promenljive važe dok je otvoren isti PowerShell prozor. Ako ga zatvoriš, sačuvaj ispisanu putanju i kasnije ponovo postavi `$aiCandidateModel` i `$aiCandidateReports` na konkretne foldere.

Kandidati se ovde čuvaju ispod `artifacts/equipment_scan`, koji je već izuzet iz Git praćenja. Nisu automatski upakovani u aplikaciju.

Za brzi ponovni trening postoji i kraća komanda:

```powershell
.\.venv-ai\Scripts\python.exe tool/ai_train_local.py
```

**Kraća komanda piše direktno preko modela u `assets/ml/equipment_scan` i standardnih izveštaja.** Ne pravi istoriju verzija. Izvoz modela prethodi završnim proverama, pa neuspešan izvoz ne treba koristiti za build. Zato je odvojeni kandidat bolji uobičajeni postupak.

## 6. Pogledaj rezultat i probaj novu fotografiju

Iz istog PowerShell prozora:

```powershell
Get-Content -LiteralPath (Join-Path $aiCandidateReports 'evaluation.json') -Encoding UTF8
.\.venv-ai\Scripts\python.exe tool/ai_predict.py --model-dir "$aiCandidateModel" --image 'C:\putanja\do\nove_fotografije.jpg'
```

Putanju fotografije zameni pravom. Koristi fotografiju koja nije već iskorišćena za trening. CLI ispisuje `recognized`, prihvaćeni naziv ili `null`, skor klase i vizuelnu sličnost. Skor nije kalibrisan procenat tačnosti.

Za postojeći model aplikacije izostavi `--model-dir`:

```powershell
.\.venv-ai\Scripts\python.exe tool/ai_predict.py --image 'C:\putanja\do\nove_fotografije.jpg'
```

Ne moraš sada pokretati veliki test paket. Kratka ručna proba nekoliko novih fotografija poznatih i drugih sprava je korisna pre zamene modela. Sama automatska provera ONNX izvoza je već deo trening skripte.

## 7. Sačuvaj staru verziju i ubaci novu

Prvo proveri da su oba fajla kandidata prisutna i da mu konfiguracija odgovara:

```powershell
$aiCandidateOnnx = Join-Path $aiCandidateModel 'equipment.onnx'
$aiCandidateConfig = Join-Path $aiCandidateModel 'config.json'
$aiConfig = Get-Content -LiteralPath $aiCandidateConfig -Raw -Encoding UTF8 | ConvertFrom-Json
$aiHash = (Get-FileHash -LiteralPath $aiCandidateOnnx -Algorithm SHA256).Hash.ToLowerInvariant()
if ($aiHash -ne $aiConfig.model_sha256) { throw 'Model i konfiguracija nisu iz istog izvoza.' }
```

Zatim napravi backup trenutnog para i kopiraj oba nova fajla:

```powershell
$aiBackupDir = Join-Path 'C:\Users\veljk\Desktop\FitnesApp\AI_model_backups' (Get-Date -Format 'yyyyMMdd_HHmmss')
New-Item -ItemType Directory -Path $aiBackupDir -ErrorAction Stop | Out-Null
Copy-Item -LiteralPath 'assets/ml/equipment_scan/equipment.onnx','assets/ml/equipment_scan/config.json' -Destination $aiBackupDir

Copy-Item -LiteralPath $aiCandidateOnnx -Destination 'assets/ml/equipment_scan/equipment.onnx'
Copy-Item -LiteralPath $aiCandidateConfig -Destination 'assets/ml/equipment_scan/config.json'
```

Ove poslednje dve komande namerno zamenjuju model aplikacije. ONNX i JSON prenosi zajedno, jer JSON definiše redosled klasa i pragove za taj model. Sačuvaj i odgovarajući izveštaj ako želiš da kasnije porediš verzije.

## 8. Napravi APK i instaliraj ga

```powershell
& 'C:\development\flutter\bin\flutter.bat' build apk --debug
```

Rezultat je:

```text
C:\Users\veljk\Desktop\FitnesApp\fitness_plus\build\app\outputs\flutter-apk\app-debug.apk
```

Prebaci APK na Android telefon i instaliraj. Za direktno pokretanje na već povezanom uređaju, uz uključen USB debugging:

```powershell
& 'C:\development\flutter\bin\flutter.bat' devices
& 'C:\development\flutter\bin\flutter.bat' run -d ID_UREDJAJA
```

`ID_UREDJAJA` zameni ID-jem koji prethodna komanda ispiše. Kamera na emulatoru potvrđuje da funkcionalnost može da se pokrene, ali za prepoznavanje stvarne sprave treba proba odgovarajućom fotografijom ili kamerom fizičkog telefona.

Ovo je razvojni debug APK. Objavljivanje potpisanog release izdanja u prodavnici je zaseban postupak. Promena modela na računaru ne menja već instaliranu aplikaciju: korisnik mora dobiti novu verziju aplikacije sa novim modelom.

## Dodavanje potpuno nove sprave

Primer za leg press:

```text
dataset/raw_video/gym01/leg_press_s01.mp4
dataset/raw_video/gym01/leg_press_s02.mp4
```

Pokreni ekstrakciju, pa trening kandidata. Klase se automatski otkrivaju iz foldera `raw`, a novi `config.json` sadržaće i `leg_press`. Flutter čita nazive iz konfiguracije, pa ne moraš dodavati novu klasu u neki fiksni Dart enum.

Za lepši naziv možeš dodati jednu stavku u postojeći `NAMES` rečnik u `tool/ai_train_local.py`, čuvajući postojeće stavke:

```python
"leg_press": "Leg press machine",
```

Ako to ne uradiš, naziv se pravi od ID-ja zamenom donjih crta razmacima.

Tekst `equipmentTrialNote` u `lib/l10n/app_en.arb` trenutno izričito pominje samo dve početne sprave. Kada proširiš model, ažuriraj i taj opis, pa generiši prevode pre build-a:

```powershell
& 'C:\development\flutter\bin\flutter.bat' gen-l10n
```

Nemoj ručno menjati redosled oznaka u JSON-u niti preimenovati ID-jeve postojećih klasa bez jasne potrebe.

## Dodavanje stvarnih `unknown` primera

Dodaj bar dva odvojena pravilno označena snimka, npr. `unknown_s01.mp4` i `unknown_s02.mp4`, pa prođi isti postupak. Folder sa izdvojenim slikama mora se zvati tačno `unknown`.

Model tada uči dodatnu eksplicitnu klasu. Aplikacija rezultat `unknown` tretira kao odbijanje, a ne kao naziv uspešno prepoznate sprave. I dalje ostaju pragovi i uzastopne potvrde. Prisustvo ove klase ne znači automatski da je model pouzdan za sve preostale sprave.

U izveštaju je `unknown_validation_available` informacija da `unknown` klasa postoji. Polje `unknown_rejection_calibrated` u konfiguraciji ostaje `false`: skripta ne radi automatsku kalibraciju nepoznatih sprava.

## Podešavanja treninga

| Opcija | Podrazumevano | Kada se koristi |
|---|---|---|
| `--raw` | `../dataset/raw` | Ako treniraš iz odvojenog skupa JPG slika. |
| `--database` | Automatski `catalog.sqlite3` pored `raw` | Eksplicitni katalog; slike moraju biti u `raw` pored te baze. |
| `--folders-only` | Isključeno | Zaobilazi bazu i njena isključenja/podele; nije redovan tok. |
| `--workers` | 0 | Paralelno učitavanje fotografija; za veći skup korišćeno 4. |
| `--max-prototypes` | 0, svi primeri | Ograničava broj predstavnika po klasi u mobilnom modelu; za novi skup 64. |
| `--onnx-check-samples` | 0, sve validacione slike | Za kratak pregled izvoza koristi stratifikovan uzorak; ne menja validaciju za izbor modela. |
| `--checkpoint` | Nema | Učitava prilagođenu osnovu i sloj iz `ai_finetune_local.py`; zahteva iste klase/redosled. |
| `--output` | `artifacts/equipment_scan` | Za odvojene izveštaje/verzije. |
| `--assets-dir` | `assets/ml/equipment_scan` | Za izvoz kandidata bez zamene modela u aplikaciji. |
| `--epochs` | `120` | Broj prolaza za završni sloj. |
| `--views` | `8` | Broj augmentovanih prikaza svake trening slike. |
| `--inspect-only` | Isključeno | Samo podela podataka i pregled slika, bez treninga i izvoza. |

Veći `--epochs` ili `--views` nisu automatski bolji model. Ako prepoznaje pogrešnu spravu, prvo proveri oznake, raznovrsnost i raspored podataka. Podešavanja pragova opisuje dokument 04.

## Okruženje i česte prepreke

Na ovom računaru biblioteke su već pripremljene. Za obnovu dodatnih Python biblioteka:

```powershell
.\.venv-ai\Scripts\python.exe -m pip install -r tool/ai_local_requirements.txt
```

Ako je obrisano samo `.venv-ai`, a globalni PyTorch/torchvision su i dalje dostupni:

```powershell
python -m venv --system-site-packages .venv-ai
.\.venv-ai\Scripts\python.exe -m pip install -r tool/ai_local_requirements.txt
```

Na drugom računaru ovo samo po sebi nije dovoljno: treba posebno obezbediti kompatibilan Python, PyTorch/torchvision i, za GPU trening, NVIDIA drajver/CUDA podršku. Ne treba kopirati `.venv-ai` kao prenosivu instalaciju. Aktuelno PyTorch uputstvo za tu mašinu treba proveriti pri njenom podešavanju.

| Simptom | Šta proveriti |
|---|---|
| `need at least two source videos` | Klasa nema dva izvora prepoznatljiva iz imena JPG fajlova. Dodaj drugi stvarni snimak/sesiju. |
| `Cannot identify source video` | JPG naziv ne prati obrazac `gym01_s01_000001.jpg`. |
| `Identical image content found` | Ista slika postoji u treningu i validaciji; ispravi kopije i poreklo. |
| Ekstrakcija kaže `already extracted` | Sesija već ima JPG fajlove. Za novi snimak daj nov naziv sesije. |
| CUDA nije dostupna | Trening automatski koristi CPU; proveri da pokrećeš pravi Python ako želiš GPU. |
| `ModuleNotFoundError` | Koristi `.venv-ai\Scripts\python.exe` i navedeni requirements fajl. |
| `flutter` nije prepoznat | Koristi punu putanju `C:\development\flutter\bin\flutter.bat`. |
| ONNX i PyTorch odluke se razlikuju | Ne promoviši taj izvoz. Sačuvaj grešku i proveri izvoz/numeričku preciznost. |
| U aplikaciji se prikazuje stari model | Potrebni su novi APK i instalacija; samo trening na PC-ju nije ažuriranje telefona. |

Prilikom prve instalacije postojao je Python problem sa čitanjem Windows TLS sertifikata. Osnovne težine su zato preuzete pomoću `curl.exe` sa zvaničnog PyTorch URL-a, uz proveru SHA256 prefiksa. Fajl je sada u lokalnom kešu, pa normalan novi trening ne treba ponovo da ga preuzima. Ako se problem ponovi posle brisanja keša, ne isključuj proveru TLS sertifikata; popravi preuzimanje/okruženje.

## Vraćanje prethodnog modela

Iz konkretno izabranog backup foldera vrati i `equipment.onnx` i `config.json` u `assets/ml/equipment_scan`, pa napravi i instaliraj novi APK. Vraćanje samo `head.pt` nije dovoljno, a prethodno instalirani APK može poslužiti za probu stare verzije samo ako je kompatibilan sa potpisom i podacima aplikacije.
