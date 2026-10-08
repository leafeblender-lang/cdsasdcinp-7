# Novi datasetovi — v3 katalog i rezultat treninga

Datum: 20. septembar 2026.

## Šta je ubačeno

U novi, odvojeni katalog ubačena su tri ZIP-a iz `C:\Users\veljk\Desktop\nove_slike_za__model_1`:

- `All Gym Equipment.v2i.coco.zip` — FitFuel, CC BY 4.0. Ovo je ista verzija koja je već ranije obrađena; uključena je u provenance i globalno deduplikovana.
- `Gym Dataset.v2-dataset-v2.coco.zip` — Akash Edwin Samuel, CC BY 4.0.
- `Gym Equipment.v1i.coco.zip` — xiaoshis-workspace, CC BY 4.0.

Postojeći GymBro ZIP i raniji FitFuel ZIP takođe su zadržani u spajanju radi kontinuiteta. Novi katalog je:

```text
C:\Users\veljk\Desktop\FitnesApp\dataset\external\roboflow_v3_20260920\
```

Originalni ZIP-ovi, stari katalog i APK/model u aplikaciji nisu prepisani.

## Obrada i mapa naziva

Ukupno je pregledano 67.431 izvornih slika. Posle provere COCO anotacija, sigurnog čitanja ZIP-a, uklanjanja stock-naziva, duplikata i nižih kopija kroz splitove, sačuvano je 15.064 kvadratnih isečaka. 4.607 anotacija za dodatke/pomoćnu opremu namerno je preskočeno jer nisu relevantne klasifikacione klase (kablovi, trake, ploče, hvataljke i slično).

Korišćena su šira, stabilna imena:

| Izvorni naziv | Ciljna klasa |
|---|---|
| `Roman Chair` | `roman_chair` |
| `Chest Machine` | `chest_press` |
| `Leg Press` / `Leg Press Machine` | `leg_press` |
| `Hamstring Curl Machine` | `leg_curl` |
| `Arm Curl Machine` / `Concentrated Bicep Curls` | `biceps_curl` |
| `Cable Machine` / `Functional Trainer` | `cable_machine` |
| `Assisted Pull Up and Dip Machine` | `assisted_dip_pullup` |
| `Back Extension Machine` | `back_extension` |

Tačna mapa je [ai_roboflow_labels.json](../fitness_plus/tool/ai_roboflow_labels.json), a poreklo svake slike, bbox i hash su u `import_manifest.jsonl`.

## Roman Chair

Uvezeno je 6 Roman Chair isečaka iz dva nova izvora. Pošto oba originalna izvora imaju samo train slike, jedan izvor je ručno označen kao `train`, drugi kao `validation`, da klasa može da se proveri. To je eksperimentalna podela sa samo 3+3 slike, ne pouzdana procena. Zato treba dodati mnogo stvarnih, različitih fotografija pre oslanjanja na ovu klasu.

Posle podele katalog ima 30 aktivnih klasa za v3 trening. Ostale slabe klase ostaju u bazi, ali nisu uključene ako nemaju zaseban train i validation izvor.

## Rezultat v3 modela

Treniran je EfficientNet-B0 kandidat u:

```text
fitness_plus/artifacts/equipment_scan/candidates/efficientnet_b0_roboflow_v3_20260920/
```

Parametri: 10.085 train slika, 1.841 validation slika, 2.918 held-out test slika, 14 epoha, RTX 3060, whole-frame letterbox obrada. Rezultat:

- validation prvi izbor: **72,35%** (1.331 / 1.841);
- held-out test prvi izbor: **51,85%** (1.513 / 2.918);
- validation Roman Chair: **3 / 3**, ali na samo tri slike;
- test Roman Chair: nema test slika;
- `ima.jpg`: odbijena; najviši kandidat je pogrešno `assisted_dip_pullup` sa skorom 0,359 i sličnosti 0,237.

Ovo je lošiji kandidat od prethodnog 29-klasnog modela (90,06% na njegovom testu), verovatno zbog šireg spajanja, neujednačenih izvora i slabije nezavisnosti splitova. **Nije kopiran u aplikaciju.** Nije ispravno spustiti prag samo da bi `ima.jpg` dobila naziv.

ONNX kandidata ima oko 25,2 MB; SHA-256 je `215aec9eb30f55182ae0517476acdadcd0e69db46f2e9666b65f3330d3f37d5b`. Konfiguracija i model su samo eksperimentalni.

## Zaključak i sledeće opcije

Podaci jesu uspešno dodati i spremni su za ciljane eksperimente, ali v3 model nije dovoljno dobar za aplikaciju. Najbolje opcije su:

1. dodati 30–100 stvarnih Roman Chair fotografija iz više uglova i teretana, pa trenirati ciljanu verziju;
2. zadržati prethodni 29-klasni model kao osnovu i pažljivo dodavati samo dobro pokrivene klase;
3. kasnije preći na detektor objekata ako kamera treba da pronađe spravu u celom kadru, umesto klasifikatora koji očekuje jednu dominantnu spravu.

Ne menjati model u `assets/ml/equipment_scan` dok se ne uradi nova validacija na stvarnim fotografijama i negativnom skupu.
