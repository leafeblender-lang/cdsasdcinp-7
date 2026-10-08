# Uvoz preuzetih ZIP-ova i novi model

[Početna strana](README.md) · [Licence](07_ROBOFLOW_DATASETOVI_I_LICENCE.md) · [SQLite komande](06_LOKALNA_BAZA.md)

Datum: 20. septembar 2026. Uvoz, oba treninga i ONNX izvoz su završeni. Bolji kandidat ima 29 opštih klasa i 82,7% tačnih prvih izbora na razvojnoj validaciji. **Model u postojećoj aplikaciji nije zamenjen.**

Naknadni EfficientNet eksperiment je završen i opisan u [09 — Novi trening i roman chair](09_NOVI_TRENING_I_ROMAN_CHAIR.md): 89,3% na validaciji, 90,1% na izdvojenom testu, ali `ima.jpg` i dalje nije prepoznata i strogi pragovi odbijaju još više poznatih slika. Nije promovisan u aplikaciju. Ispod je sačuvan opis prethodnih MobileNet pokušaja; njihov test je naknadno ocenjen radi poređenja u dokumentu 09.

## Gde su podaci

Originalni ZIP fajlovi ostali su netaknuti:

```text
C:\Users\veljk\Desktop\GymBro.v2i.coco.zip
C:\Users\veljk\Desktop\All Gym Equipment.v2i.coco.zip
```

Novi, odvojeni dataset je u:

```text
C:\Users\veljk\Desktop\FitnesApp\dataset\external\roboflow_v2_20260920\
    catalog.sqlite3         novi katalog
    raw\                    pripremljene slike po klasama
    label_mapping.json      tačno spajanje izvornih oznaka
    import_manifest.jsonl   izvor svake slike, okvir i izvršena obrada
    import_report.json      brojevi, hash ZIP-ova i ograničenja
    excluded_sources.json   izostavljeni izvori i razlozi
    ATTRIBUTION.md          autori, linkovi, licence i izmene
    provenance\             originalna obaveštenja iz oba ZIP-a
```

Stari `dataset/catalog.sqlite3` i `dataset/raw` nisu menjani. Novom skupu pridružene su kopije svih 97 tvojih postojećih slika, uz očuvanu podelu snimaka na trening i validaciju. Sačuvaj oba originalna ZIP-a: novi folder ne sadrži sve originalne fotografije/anotacije, već pripremljene primere i evidenciju uvoza.

## Koliko je uvezeno

| Mera | Broj |
|---|---:|
| Slike u izvornim ZIP-ovima, uključujući augmentacije | 15.655 |
| Slike u novom katalogu, uključujući tvoje kopije | 11.915 |
| Aktivne trening slike | 10.143 |
| Aktivne validacione slike | 1.196 |
| Izdvojene test slike | 543 |
| Klase uključene u model | 29 |
| Klase privremeno isključene zbog premalo podataka | 4 |

Preostale 33 pripremljene slike pripadaju isključenim klasama. Broj pripremljenih slika nije prost ostatak od broja ZIP fotografija: jedna fotografija može dati više isečaka različitih sprava. Ni 11.915 primera nije isto toliko nezavisnih snimanja.

## Šta je urađeno sa slikama

- Licence u `README.dataset.txt` i COCO metapodacima potvrđene su kao CC BY 4.0.
- Izostavljeno je sedam fajlova sa Shutterstock/Getty oznakama, jer njihovo zasebno poreklo/prava nisu potvrđeni. To ne predstavlja garanciju prava na sve ostale Internet fotografije.
- Grupe izvora prepoznate su po originalnim nazivima, uklanjanju `mirrored`/Roboflow sufiksa, identičnim fajlovima i jednakim perceptualnim hash-evima uz rotacije/ogledanje.
- Nađeno je 1.090 grupa koje se pojavljuju u više podela. Izostavljeno je 4.268 fotografija iz niže prioritetne podele. Prioritet je `test > validation > train`: originalne podele nisu nasumično preslagane, a pronađene kopije iz treninga nisu zadržane ako postoje u izdvojenom skupu.
- Iz COCO okvira napravljeni su kvadratni isečci cele označene sprave sa malo okoline, smanjeni na 384 × 384. Model iz njih koristi ulaz 224 × 224.
- Preskočeno je 414 premalih/neispravnih okvira i 585 identičnih gotovih isečaka.
- Originali nisu obrisani. Sve novonapravljene datoteke nalaze se u odvojenom folderu.

Grupisanje je konzervativno: slična generička imena mogu spojiti različite slike, a hash provera ne može garantovati pronalaženje svih skoro-identičnih fotografija. Nije dokazana nezavisnost celih teretana i fizičkih primeraka sprava. Zbog toga rezultate i dalje treba čitati kao razvojnu validaciju, ne kao garanciju rada u novoj teretani.

## Koje klase se koriste

```text
ab_roller                 abdominal_crunch        assisted_dip_pullup
back_extension            barbell                 biceps_curl
cable_machine             chest_fly               chest_press
dumbbell                  hack_squat              hip_abduction_adduction
kettlebell                lat_pulldown            lateral_raise
leg_curl                  leg_extension           leg_press
leg_raise_tower            seated_dip              seated_row
shoulder_press            smith_machine           squat_rack
stepmill                  torso_rotation          treadmill
triceps_extension         weight_bench
```

Ovo su **opšti tipovi**, ne 29 preciznih modela/proizvođača. `converging_chest_press` iz tvojih slika u ovom kandidatu ulazi u širu klasu `chest_press`; `plate_loaded_seated_row` u `seated_row`. Ne tvrdi se da su sve tuđe chest-press sprave converging, niti da su sva veslanja plate-loaded.

Srodne izvorne oznake spojene su, npr. sedeći/ležeći leg curl → `leg_curl`, različita veslanja → `seated_row`, bench press/incline/obična klupa → `weight_bench`. Tačna mapa je u `label_mapping.json` i u skripti `tool/ai_roboflow_labels.json`.

`abdominal_bench`, `elliptical`, `ghd` i `stability_ball` privremeno su isključeni iz treninga jer posle filtriranja nemaju minimum 10 trening i 2 validaciona primera. Ti minimumi su samo prag uključivanja, ne dokaz da ima dovoljno podataka za pouzdan model. Posebno su male klase poput `stepmill` i `ab_roller` još slabo pokrivene.

## Pregled novog kataloga

Iz `fitness_plus`, u PowerShell-u:

```powershell
Set-Location -LiteralPath 'C:\Users\veljk\Desktop\FitnesApp\fitness_plus'
.\.venv-ai\Scripts\python.exe tool/ai_catalog.py --database '../dataset/external/roboflow_v2_20260920/catalog.sqlite3' stats
.\.venv-ai\Scripts\python.exe tool/ai_catalog.py --database '../dataset/external/roboflow_v2_20260920/catalog.sqlite3' images --label chest_press --limit 10
```

Iste komande za `set-enabled`, `source`, `backup` i `export` iz dokumenta 06 važe uz ovu eksplicitnu putanju baze. Bez `--database`, podrazumevani katalog je i dalje stari dataset sa tvoje dve klase.

## Trening i izlazne putanje

Prvi kandidat sa zamrznutim MobileNet osnovnim delom sačuvan je u:

```text
fitness_plus/artifacts/equipment_scan/candidates/roboflow_v2_20260920/
    reports/evaluation.json
    reports/head.pt
    mobile/equipment.onnx
    mobile/config.json
```

Na validaciji je imao 813/1.196 tačnih prvih izbora (oko 68%). Sa strogim pragovima prihvatio je 59 tačnih i jedan pogrešan rezultat, a odbio 1.136 poznatih primera. Zato nije postavljen u aplikaciju kao završen model.

Drugi postupak kratko prilagođava i osnovnu neuronsku mrežu, a zatim priprema klasifikacioni sloj i ONNX. Njegov odvojeni folder je:

```text
fitness_plus/artifacts/equipment_scan/candidates/roboflow_v2_finetuned_20260920/
    finetuned.pt            težine mreže i klasifikatora za nastavak obrade
    finetuned.json          tok dodatnog treninga i izbor epohe
    reports/               rezultat, podela podataka i pomoćne datoteke
    mobile/                ONNX, konfiguracija i atribucija izvora
```

Trening se radi na tvojoj RTX 3060. Nije korišćen Roboflow API/cloud trening niti kupljena pretplata ili kredit. Novi parametri omogućavaju paralelno učitavanje slika i najviše 64 trening predstavnika po klasi, da broj fotografija ne poveća mobilni model bez granice.

Pri izvozu je zatražena kratka, stratifikovana provera ONNX/PyTorch saglasnosti: po nekoliko slika svake klase, ne sve validacione slike. Tačan broj i greška nalaze se u `evaluation.json`. Validacija za izbor modela koristi sve 1.196 slika. Test skup od 543 slike nije korišćen za izbor epohe niti ovde automatski ocenjen.

### Konačni rezultat boljeg kandidata

Izvor: [evaluation.json](../fitness_plus/artifacts/equipment_scan/candidates/roboflow_v2_finetuned_20260920/reports/evaluation.json).

| Mera | Rezultat |
|---|---:|
| Tačan prvi izbor, pre pragova | 989 / 1.196 = 82,69% |
| Prihvaćeni tačni rezultati | 230 |
| Prihvaćeni pogrešni rezultati | 3 |
| Odbijeni poznati primeri | 963 |
| Veličina ONNX modela | 7.748.456 bajtova, oko 7,75 MB |
| Proverene ONNX/PyTorch validacione slike | 86, iz svih 29 klasa |
| Najveća razlika skorova pri toj proveri | Oko 0,00000399 |

Izabrana je peta od šest epoha fine-tuning-a, a zatim četvrta epoha dodatne obrade klasifikacionog sloja. Oba izbora koriste validaciju, ne test. Nisu sprovedeni veliki Flutter test paket, novi APK build niti proba na fizičkom telefonu.

Pragovi ostaju 0,88 za skor, 0,40 za razliku od sledeće klase i 0,80 za sličnost. Sa njima model prihvata samo oko 19,5% ovih poznatih fotografija. Među prihvaćenim rezultatima 230/233 je tačno, ali to nije izmerena pouzdanost prema nepoznatim spravama. **82,7% ne znači da skener uspešno prepozna 82,7% fotografija posle svih pragova.** Pre primene treba raditi na odzivu i stvarnim negativnim primerima; pragove ne treba spuštati samo da bi ekran češće prikazivao rezultat.

SHA-256 starog modela aplikacije ponovo je potvrđen kao nepromenjen, a stari katalog i dalje sadrži 97 aktivnih slika.

## Kako ponoviti ili nastaviti trening

Ne moraš ponovo uvoziti iste ZIP-ove. Možeš koristiti postojeći novi katalog. Svaki novi pokušaj sačuvaj u novom folderu:

```powershell
$aiNextRun = Join-Path 'artifacts/equipment_scan/candidates' ('roboflow_' + (Get-Date -Format 'yyyyMMdd_HHmmss'))
.\.venv-ai\Scripts\python.exe tool/ai_train_local.py --database '../dataset/external/roboflow_v2_20260920/catalog.sqlite3' --checkpoint 'artifacts/equipment_scan/candidates/roboflow_v2_finetuned_20260920/finetuned.pt' --output "$aiNextRun/reports" --assets-dir "$aiNextRun/mobile" --views 1 --epochs 120 --workers 4 --max-prototypes 64 --onnx-check-samples 64
```

Ova komanda učitava već prilagođenu osnovnu mrežu i trenira završni sloj; ona ne ponavlja fine-tuning osnovne mreže. Redosled klasa u checkpoint-u mora odgovarati katalogu. Ako dodaš nove klase, ne koristi stari checkpoint sa drugačijim završnim slojem ovom komandom; potrebno je prilagoditi prenos težina ili krenuti bez `--checkpoint`.

Za ponovno kratko prilagođavanje mreže na istim klasama, uz novi izlazni fajl:

```powershell
.\.venv-ai\Scripts\python.exe tool/ai_finetune_local.py --database '../dataset/external/roboflow_v2_20260920/catalog.sqlite3' --initial-head 'artifacts/equipment_scan/candidates/roboflow_v2_20260920/reports/head.pt' --output "$aiNextRun/finetuned.pt" --epochs 6 --workers 2
```

Ova skripta polazi od ImageNet osnove i početnog klasifikacionog sloja navedenog u komandi; nije nastavak iz proizvoljnog ONNX modela. Potom izvozu prosledi novi `--checkpoint`. Originalni importer `tool/ai_import_roboflow.py` odbija već postojeći izlazni folder, da ne bi prepisao podatke ili napravio duple unose.

## Proba i buduće uključivanje u aplikaciju

Fotografiju koja nije trening primer možeš probati ovako:

```powershell
.\.venv-ai\Scripts\python.exe tool/ai_predict.py --model-dir 'artifacts/equipment_scan/candidates/roboflow_v2_finetuned_20260920/mobile' --image 'C:\putanja\do\nove_fotografije.jpg'
```

Novi model još ne sadrži pravilno prikupljene `unknown` primere. Pragovi su početni, a skor nije kalibrisan procenat tačnosti. Druga sprava može biti pogrešno prihvaćena, a poznata odbijena. Slike za trening uglavnom su izdvojeni objekti; kamerom je potrebno držati ciljnu spravu u centralnom kvadratu.

Model iz `mobile/` **nije automatski upakovan u stari APK**. Pre eventualnog uključivanja treba sačuvati stari par ONNX/JSON, proveriti nekoliko novih poznatih/nepoznatih fotografija i ažurirati tekst podržanih sprava i vidljivo navođenje autora/licenci u aplikaciji. Sa modelom zadržati `ATTRIBUTION.md` i originalna obaveštenja iz `provenance/`. Zatim se radi novi APK prema dokumentu 03.
