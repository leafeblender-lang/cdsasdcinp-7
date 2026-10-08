# Ručni pregled i ispravka podataka

Datum: 21. septembar 2026.

## Zašto novi model nije postao bolji

Problem nije grafička kartica niti premalo epoha. Uvoz je obradio svih 67.431
izvornih zapisa, ali je stara logika preagresivno spajala slike po nazivu fajla
i davala prednost `test`, zatim `validation`, pa `train` varijanti. Zbog toga je
48.773 zapisa odbačeno pre pripreme za trening. Generički nazivi kao `images`,
`download-2` i `-0008` nisu pouzdan dokaz da je u pitanju ista fotografija.

Uz to:

- 6.517 izvornih slika nema COCO anotaciju, pa iz njih nema označenog objekta za crop;
- 1.611 anotacija je bilo premalo ili neispravno za upotrebljiv crop;
- 222 anotacije za `Elliptical`/`cross-trainer` nisu bile mapirane;
- različiti datasetovi sadrže transformisane kopije istih scena i poneke konfliktne oznake.

Zato broj od oko 14–16 hiljada pripremljenih slika nije predstavljao sve korisne
varijante iz ZIP-ova. Sledeći import mora da zadrži varijante za trening, a da celu
istu izvornu scenu drži u istom splitu.

## Prvi ručni batch: 100 slika

Pregledano je tačno 100 slika. Fajlovi pregleda su ovde:

```text
C:\Users\veljk\Desktop\FitnesApp\fitness_plus\artifacts\equipment_scan\manual_review_v4\
├── batch_0001.json
├── batch_0001_decisions.json
├── batch_0001_sheet_01.jpg
├── batch_0001_sheet_02.jpg
├── batch_0001_sheet_03.jpg
├── batch_0001_sheet_04.jpg
└── batch_0001_sheet_05.jpg
```

Rezultat pregleda:

- 90 slika može da zadrži trenutnu oznaku;
- 5 slika ima pogrešnu klasu i dobilo je ispravku;
- 5 slika je mutno, nejasno ili nema dovoljno izdvojen objekat i označeno je za isključenje;
- pronađeno je više grupa transformisanih kopija koje ubuduće moraju ostati u istom splitu.

Konkretne ispravke su zapisane u `batch_0001_decisions.json`. Najvažnije:

- cela kablovska sprava nije `cable_triceps_attachment`, već `cable_machine`;
- slika sa dve girje nije `exercise_ball`, već `kettlebell`;
- push-up diskovi na spornoj slici pripadaju klasi `sliders`;
- dve slike označene kao `roman_chair` zapravo prikazuju `captains_chair` / vertical-knee-raise spravu.

Stabilna pravila za sledeći uvoz su u `review_overrides.json`. Ona koriste
`dataset + originalni ZIP member + COCO annotation_id`, a ne promenljivi ID iz
SQLite baze. Novi importer ih prima preko `--review-overrides`, automatski radi
`relabel`/`exclude` i prekida rad ako neko pravilo nije pronađeno ili je duplirano.
Tako se ručne ispravke neće izgubiti pri pravljenju novog v5 kataloga.

## Roman chair

Svih šest postojećih `roman_chair` cropova potiče iz jedne iste izvorne scene,
iako su došli kroz dva ZIP-a. Na njima je captain's chair / vertical knee raise,
a ne roman-chair klupa za hiperekstenziju. Ranijih 3/3 na validaciji zato nije
validan rezultat: trening i validacija su sadržali prerade iste pogrešno označene scene.

Oba izvora su sada postavljena na `exclude` u v4 katalogu. Pre promene je napravljena
rezervna kopija:

```text
C:\Users\veljk\Desktop\FitnesApp\dataset\external\roboflow_v4_all_equipment_20260920\catalog_before_manual_review_20260921.sqlite3
```

Za pravi `roman_chair` treba dodati više nezavisnih fotografija stvarnih
hiperekstenzionih klupa, iz različitih uglova i teretana. Korisnikova `ima.jpg`
je dobar prvi primer, ali jedna fotografija nije dovoljna za trening i pošten test.

## Kako nastavljamo

Pregled ide u batch-evima od po 100 slika. Posle svakog batcha čuvaju se:

1. kontaktne table;
2. spisak svih pregledanih ID-jeva;
3. odluke `keep`, `relabel` ili `exclude`;
4. grupe vizuelnih kopija koje moraju biti u istom splitu.

Novi veliki trening se ne pokreće dok se ne napravi ispravljena podela podataka.
Model u Flutter aplikaciji nije menjan.

Importer je zatim ispravljen tako da više ne odbacuje train varijante zbog
Roboflow holdout prioriteta. Pouzdane varijante iste scene dobijaju isti,
deterministički split približno 80/10/10; generički nazivi fajlova se ne koriste
za grupisanje, a automatsko pHash spajanje je uklonjeno. Exact SHA duplikati se
i dalje bezbedno prepoznaju.

Potvrđene mape su takođe ispravljene:

- `Roman Chair` i `Leg Raise Tower` iz ovih izvora sada su `captains_chair`;
- neodređena klasa `Ball` se više ne uvozi naslepo i zahteva ručno razvrstavanje;
- `Cable Triceps Extension` se vodi kao cela `cable_machine`;
- mali broj vrsta kablovskih ručki objedinjen je u pouzdaniju klasu `cable_attachment`;
- `Elliptical` i `cross-trainer` koriste zajedničku klasu `elliptical_cross_trainer`.
