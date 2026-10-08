# V4 — sve relevantne sprave, rekviziti i nastavci

Datum: 20. septembar 2026.

## Šta je promenjeno

Napravljen je novi odvojeni katalog:

```text
C:\Users\veljk\Desktop\FitnesApp\dataset\external\roboflow_v4_all_equipment_20260920\
```

Za razliku od prethodnog importa, nisu preskočene kategorije kao:

- kablovski nastavci i hvataljke (`cable_attachment`, `rope_attachment`, `vbar_attachment`, `wide_grip_attachment`...);
- ploče (`weight_plate`);
- trake (`resistance_bands`);
- vratila/paralelne šipke (`pullup_bar`, `parallel_bars`);
- lopte, foam roller, plyometric box, sliders, punching bag;
- bicikli, stair climber i ostala oprema.

Ukupno je pregledano 67.431 izvornih slika, sačuvano 16.677 pripremljenih isečaka, a v4 trening koristi 11.282 train slike, 1.998 validation slika i 3.092 test slike. Katalog ima 43 aktivne klase. Slabe klase bez odvojenog train/validation izvora ostaju zabeležene u bazi, ali nisu ubačene u model.

Sva tri nova ZIP-a su obrađena, uključujući `All Gym Equipment.v2i.coco.zip`, koji je ranije već postojao i sada je globalno deduplikovan. Originalni ZIP-ovi, v2/v3 katalozi i aplikacioni model nisu menjani.

## Rezultat v4 treninga

Kandidat je u:

```text
fitness_plus/artifacts/equipment_scan/candidates/efficientnet_b0_roboflow_v4_all_equipment_20260920/
```

- 43 klase;
- EfficientNet-B0, 14 epoha, RTX 3060;
- validation prvi izbor: **74,27%**;
- held-out test prvi izbor: **52,75%**;
- validation prihvaćeno: 394 tačna / 4 pogrešna;
- test prihvaćeno: 315 tačnih / 1 pogrešno;
- Ranije prijavljen Roman Chair rezultat 3/3 nije validan: svih 6 cropova su prerade jedne iste scene i zapravo prikazuju `leg_raise_tower`, pa su naknadno isključeni iz kataloga;
- Roman Chair nema test primere;
- `ima.jpg`: odbijena; najviši kandidat je pogrešno `dumbbell` (skor 0,252, sličnost 0,212).

Rezultat je lošiji od prethodnog 29-klasnog modela. Više slika nije automatski poboljšalo model jer su klase sada heterogene, neki izvori imaju različite definicije opreme, a postojeći test split nije dovoljno nezavisan. To nije razlog da se podaci izbace; znači da treba promeniti strategiju modela i označavanje.

## Status aplikacije

V4 model **nije ubačen u aplikaciju**. Aplikacija i dalje koristi prethodni model za dve početne sprave. V4 ONNX je samo eksperimentalni fajl; veličina je oko 28 MB. Model koristi `letterbox` preprocessing, dok aplikacija trenutno koristi staru centralnu obrezanu sliku.

## Sledeći tehnički korak

Za realan proizvod bolji je dvostepeni sistem:

1. detektor pronađe spravu/rekvizit u celom kadru;
2. klasifikator prepozna samo izabrani objekat.

Za sada su svi dodatni podaci sačuvani i dostupni za novi trening. Ne treba samo spustiti prag, jer bi aplikacija češće prikazivala pogrešne nazive.
