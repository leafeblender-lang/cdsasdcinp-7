# Novi trening i problem sa roman-chair fotografijom

Datum: 20. septembar 2026. Status: svih 14 epoha, ONNX izvoz i poređenje su završeni. Novi kandidat poboljšava ukupan prvi izbor, ali **i dalje ne prepoznaje `ima.jpg` i sa postojećim pragovima prihvata manje poznatih slika**. Nije promovisan u aplikaciju. Aplikacija još koristi prvobitni model za dve sprave.

## Rezultat novog treninga

| Mera | Prethodni MobileNet kandidat | Novi EfficientNet kandidat |
|---|---:|---:|
| Tačan prvi izbor, validacija | 989 / 1.196 = 82,69% | 1.068 / 1.196 = 89,30% |
| Tačan prvi izbor, izdvojeni test | 469 / 543 = 86,37% | 489 / 543 = 90,06% |
| Validacija: prihvaćeno tačno / pogrešno | 230 / 3 | 109 / 0 |
| Validacija: odbijeno poznatih | 963 | 1.087 |
| Test: prihvaćeno tačno / pogrešno | 137 / 1 | 68 / 0 |
| Test: odbijeno poznatih | 405 | 475 |
| `back_extension`, test pre pragova | 6 / 15 | 6 / 15 |
| Tvoja `ima.jpg` | Nije prepoznata | Nije prepoznata |

Izabrana je 12. epoha po najnižem validacionom gubitku; 14 epoha trajalo je oko 369 sekundi, ne računajući pripremu/izvoz. Test skup nije korišćen za izbor epohe niti podešavanje pragova. Posle zaključavanja novog modela, prethodni model je takođe ocenjen na istim test slikama radi poređenja. Izveštaji: [evaluation.json](../fitness_plus/artifacts/equipment_scan/candidates/efficientnet_b0_wholeframe_20260920_r1/evaluation.json) i [previous_model_test.json](../fitness_plus/artifacts/equipment_scan/candidates/efficientnet_b0_wholeframe_20260920_r1/previous_model_test.json).

Važno: 90,06% je tačnost prvog izbora među poznatim kategorijama, **nije procenat uspešnih skeniranja**. Sa prenetim pragovima novi model prihvata samo 68/543 = 12,52% test slika (stari 138/543 = 25,41%). Prag sličnosti 0,80 nije kalibrisan za novu reprezentaciju. Zato ovo nije spremna zamena skenera, iako se poboljšala ukupna klasifikacija. Nula pogrešno prihvaćenih u ovom skupu ne dokazuje bezbedno odbijanje nepoznatih sprava; nije prikupljen takav skup.

Na `ima.jpg` novi model najviše rangira pogrešnu kategoriju `lateral_raise`: skor 0,81439, sličnost 0,38864. Rezultat je odbijen. Ne spuštati prag da bi se ta fotografija nasilno prihvatila — prihvatio bi se pogrešan naziv. Sama zamena mreže nije popravila slabost `back_extension` kategorije. `barbell` takođe ostaje slaba klasa (1/7 na testu); pet kategorija u ovom test skupu nema nijedan primer. Internet test nije dokaz nezavisnosti fizičkih sprava/teretana.

ONNX ima 24.963.971 bajtova (oko 25 MB). Provera izvoza na po dve validacione slike svih 29 kategorija: 58 slika, najveća razlika skorova oko 0,00000608, iste odluke. SHA-256: `d0b220e878fb2707eebcebbb3972257aba30e74a806f9c423b38519f39b2b1e9`. Ovo je kratka numerička provera izvoza, ne veliko testiranje aplikacije. Novi APK nije pravljen.

## Zašto baza ima 33, a model 29 kategorija

Tabela `classes` čuva sve uvezene oznake, uključujući isključene. Prethodni importer je zahtevao bar 10 trening i 2 validacione slike da uključi klasu u model. To je samo tehnički minimum, ne garancija dobrog prepoznavanja.

| Kategorija van modela | Trening slike | Validacione slike |
|---|---:|---:|
| `abdominal_bench` | 0 | 0 |
| `elliptical` | 7 | 2 |
| `ghd` | 4 | 4 |
| `stability_ball` | 6 | 1 |

Ove kategorije/slike nisu obrisane. Njihovi izvori su isključeni iz treninga oznakom `split='exclude'`; pojedinačna polja `enabled` ostala su uključena. Dodavanje reda u `classes` ne dodaje prepoznavanje već napravljenom modelu. Novi trening takođe koristi 29 aktivnih kategorija. Nisu premeštane test slike u trening radi povećanja brojeva. Brojevi u gornjoj tabeli potiču iz izveštaja uvoza, pre postavljanja `exclude`.

U DB Browser for SQLite, `Execute SQL`:

```sql
SELECT label,
       COUNT(*) AS sve_slike,
       SUM(enabled=1 AND valid=1 AND missing=0 AND split='train') AS aktivni_train,
       SUM(enabled=1 AND valid=1 AND missing=0 AND split='validation') AS aktivna_validacija,
       SUM(enabled=1 AND valid=1 AND missing=0 AND split='test') AS aktivni_test
FROM image_catalog
GROUP BY label
ORDER BY label;
```

## Zašto prethodni model nije prepoznao ima.jpg

Direktno su proverena oba postojeća modela. Onaj u aplikaciji poznaje samo `converging_chest_press` i `plate_loaded_seated_row`; roman chair nije njegov mogući izlaz. Širi MobileNet kandidat ima `back_extension`, ali na `ima.jpg` pogrešno najviše rangira `triceps_extension` (skor oko 0,302), dok `back_extension` ima oko 0,00170. Nijedan model nije prihvatio rezultat.

Stara obrada od fotografije 831 × 1801 zadržava centralni kvadrat između y=485 i y=1316. Tako se gube delovi sprave. Međutim, proba cele fotografije sa istim starim modelom takođe nije uspela: problem nije samo isecanje niti samo prag prihvatanja.

Pregled trening podataka otkrio je važnije ograničenje:

- `back_extension` ima 170 trening slika, ali to nisu 170 različitih sprava.
- 168 dolazi iz GymBro, a samo dve iz FitFuel skupa.
- Dve FitFuel slike su varijante istog izvora sa roman-chair klupom.
- Ostala 31 grupisana izvora u vizuelnom pregledu prikazuju sedeće back-extension sprave, uključujući mnogo kadrova iste sprave/teretane.
- Zato ova objedinjena kategorija nije dobro pokrila roman chair iz drugih uglova/proizvođača.

Pregled izvora: [back_extension_train_sources_20260920.jpg](../fitness_plus/artifacts/equipment_scan/roboflow_review/back_extension_train_sources_20260920.jpg). Skripta `tool/ai_review_class.py` pravi ovaj pregled samo iz trening podele, bez promene baze.

## Šta novi postupak menja

- EfficientNet-B0 umesto MobileNetV3-Small, sa zvaničnim početnim ImageNet težinama iz torchvision-a.
- Do 14 epoha: prva prilagođava klasifikator, ostale celu mrežu. Prekida se ranije ako validacioni gubitak ne napreduje.
- Čuva celu fotografiju i proporcije, uz dopunu ivica (`letterbox`) umesto centralnog isecanja. Ulaz ostaje 224 × 224.
- Umerenije uravnotežuje zastupljenost kategorija i smanjuje uticaj višestrukih varijanti istog grupisanog izvora.
- Menja osvetljenje, veličinu objekta u kadru, ugao i perspektivu tokom treninga.
- RTX 3060, PyTorch, mixed precision. Baza i prethodni modeli se ne prepisuju.
- I dalje nema pouzdano kalibrisane provere za nepoznate sprave. Pragovi ostaju 0,88 / 0,40 / 0,80, bez podešavanja prema `ima.jpg`.

Trening koristi isti katalog: `dataset/external/roboflow_v2_20260920/catalog.sqlite3`, istih 10.143 trening i 1.196 validacionih slika. Test skup od 543 slike čita se tek posle izbora konačnih težina. `ima.jpg` nije deo treninga/validacije; posle izvoza proverava se kao korisnička fotografija. Pošto je ta fotografija već ukazala na problem obrade kadra, ona nije slepi nezavisni benchmark.

## Putanje i ponavljanje

Završen pokušaj:

```text
fitness_plus/artifacts/equipment_scan/candidates/efficientnet_b0_wholeframe_20260920_r1/
    best.pt                 najbolji checkpoint po validacionom gubitku
    history.json            rezultati svake završene epohe
    split_manifest.json     tačna podela primera
    evaluation.json         konačni rezultat, test i proba ima.jpg
    previous_model_test.json  poređenje prethodnog modela na istom test skupu
    mobile/                 ONNX, konfiguracija, atribucija
```

Prvi pokušaj bez `_r1` prekinuo se pre treninga zbog Python/Windows sertifikata pri preuzimanju težina. Težine su zatim preuzete sa istog zvaničnog HTTPS izvora bez isključivanja TLS provere; SHA-256 je `7f5810bc96def8f7552d5b7e68d53c4786f81167d28291b21c0d90e1fca14934`.

Za novi pokušaj izaberi novo ime foldera:

```powershell
Set-Location -LiteralPath 'C:\Users\veljk\Desktop\FitnesApp\fitness_plus'
$aiRun = 'artifacts/equipment_scan/candidates/efficientnet_' + (Get-Date -Format 'yyyyMMdd_HHmmss')
.\.venv-ai\Scripts\python.exe tool/ai_train_v2.py --database '../dataset/external/roboflow_v2_20260920/catalog.sqlite3' --output $aiRun --epochs 14 --workers 4 --batch-size 48 --probe-image 'C:\Users\veljk\Desktop\ima.jpg'
```

`ai_train_v2.py` odbija postojeći izlazni folder. Za svako novo treniranje koristi novo ime. `best.pt` nije kompletan zapis optimizatora za tačan nastavak prekinute epohe. Trening se ne naplaćuje po slikama/pozivima; koristi lokalni računar, struju i prostor. Nisu otvoreni nalozi, plaćeni servisi ni novi cloud resursi.

Lokalna proba gotovog modela:

```powershell
.\.venv-ai\Scripts\python.exe tool/ai_predict.py --model-dir 'artifacts/equipment_scan/candidates/efficientnet_b0_wholeframe_20260920_r1/mobile' --image 'C:\Users\veljk\Desktop\ima.jpg'
```

`ai_predict.py` sada čita način obrade iz konfiguracije; ostaje kompatibilan sa starim modelima. **Novi model se ne sme samo prekopirati u staru aplikaciju:** Flutter trenutno uvek centralno iseca kadar, pa za `letterbox` treba uskladiti obradu kamere/fotografije, zatim uključiti model, vidljivu atribuciju i napraviti novi APK. To nije automatski urađeno ovom skriptom.

## Ako roman chair i dalje ne radi

Najkorisnije je dopuniti stvarne podatke, ne ponavljati isti trening u nedogled:

1. Snimiti celu roman-chair spravu iz više uglova, u bar dva odvojena snimanja; poželjno i više fizičkih primeraka/proizvođača. Jedan snimak ide u trening, drugi ostaje za proveru. Desetine raznovrsnih slika vrede više od hiljada susednih video-kadrova.
2. Razdvojiti `roman_chair` od sedeće `back_extension_machine` ako aplikacija treba da ih razlikuje; to zahteva pregled i tačno označavanje, ne samo preimenovanje kategorije.
3. Za kameru koja vidi više sprava razviti detektor koji prvo nalazi okvir objekta, a zatim prepoznaje tip. To je zaseban korak sa proverom podataka, licence konkretnog modela i brzine na telefonu.

Pre promovisanja bilo kog novog modela potrebni su i stvarni negativni primeri (nepodržane sprave, prazna teretana i slični objekti), da se pragovi podese za izabranu mrežu. U ovom pokušaju nisu menjani prema izdvojenom testu niti korisničkoj slici.

Ovaj novi trening je ograničen pokušaj poboljšanja, ne garancija prepoznavanja svih sprava. Visok rezultat na delovima istog internet skupa ne dokazuje rad u novoj teretani.

## Tehnički izvori

- [Torchvision EfficientNet-B0](https://docs.pytorch.org/vision/stable/models/generated/torchvision.models.efficientnet_b0.html): model i početne težine. Novi postupak namerno koristi sopstvenu celokadarsku obradu i dodatni trening, ne neizmenjenu ImageNet obradu.
- [PyTorch transfer learning](https://docs.pytorch.org/tutorials/beginner/transfer_learning_tutorial.html): prilagođavanje prethodno trenirane mreže.
- Licence konkretnih datasetova i poreklo ostaju u [07](07_ROBOFLOW_DATASETOVI_I_LICENCE.md), [08](08_UVOZ_I_TRENING_ROBOFLOW.md) i dataset `ATTRIBUTION.md`.
