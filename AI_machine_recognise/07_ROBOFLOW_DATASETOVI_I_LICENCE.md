# Roboflow datasetovi: licence i sledeći korak

[Početna strana](README.md) · [Lokalna baza](06_LOKALNA_BAZA.md)

Provereno: 20. septembra 2026. **Ažuriranje: korisnik je preuzeo oba ZIP-a; potvrđene su njihove licence, završen je uvoz u zaseban SQLite katalog, lokalni trening i ONNX izvoz.** Detalji i stvarni brojevi su u [dokumentu 08](08_UVOZ_I_TRENING_ROBOFLOW.md). Model u aplikaciji i originalnih 97 lokalnih slika nisu promenjeni. Ostatak ovog dokumenta čuva prethodnu proveru uslova i postupak preuzimanja.

## Koji skupovi su pregledani

| Dataset | Objavljeni autor | Objavljena licenca | Podaci sa stranice |
|---|---|---|---|
| [All Gym Equipment, verzija 2](https://universe.roboflow.com/fitfuel/all-gym-equipment/dataset/2) | FitFuel | CC BY 4.0 | V2: 4.717 slika, uključujući augmentacije; train 4.341, valid 334, test 42. |
| [GymBro, verzija 2](https://universe.roboflow.com/rizzlabzz/gymbro/dataset/2) | RizzLabzz | CC BY 4.0 | V2: 10.938 slika, uključujući augmentacije; train 9.561, valid 907, test 470. |

[Pregled FitFuel projekta](https://universe.roboflow.com/fitfuel/all-gym-equipment) navodi 29 klasa i 1.947 slika u projektu. [Pregled GymBro projekta](https://universe.roboflow.com/rizzlabzz/gymbro) navodi 23 klase i 4.564 slike u projektu. Broj slika u projektu i broj u izvezenoj verziji nisu ista mera: verzije mogu koristiti podskup i umnožavati trening primere augmentacijama.

Obe navedene verzije imaju tri izlaza po trening primeru. Zato zbir od 15.655 izvezenih slika **nije 15.655 nezavisnih fotografija**, a moguće preklapanje između projekata još nije provereno.

## Da li moraš nešto da platiš autorima?

Prema objavljenoj **CC BY 4.0 licenci**, dozvoljeno je kopiranje, prilagođavanje i komercijalno korišćenje. Licenca je royalty-free: ne zahteva procenat zarade niti naknadu autoru. Potrebno je sačuvati odgovarajuće navođenje autora/izvora, link licence i napomenu o izmenama. Ne sme se sugerisati da autori podržavaju tvoju aplikaciju. [Sažetak licence](https://creativecommons.org/licenses/by/4.0/) i [pravni tekst, odeljak 2](https://creativecommons.org/licenses/by/4.0/legalcode.en).

Ako se licencirani materijal dalje deli, treba poštovati i zabranu dodatnih ograničenja prava koja licenca daje. CC BY nije CC BY-NC niti CC BY-SA; sama po sebi ne traži procenat prihoda ili objavljivanje celog izvornog koda aplikacije. Ovo nije pravna garancija da je objavljivač imao sva prava na svaku fotografiju: licenca ne daje takvu garanciju, a privatnost i druga prava mogu biti odvojena. Ako se pri pregledu utvrdi sporno poreklo, te primere treba izostaviti. [Uslovi i ograničenja](https://creativecommons.org/licenses/by/4.0/).

Prilikom preuzimanja proveriti i sačuvati priložene README/licencne fajlove. Ako se njihovi uslovi razlikuju od prikazane licence ili se traži naknada, ne nastavljati sa korišćenjem bez razjašnjenja.

## Preuzimanje naspram plaćenih Roboflow usluga

Roboflow opisuje preuzimanje javnih Universe datasetova pomoću besplatnog naloga. [Zvanično uputstvo](https://blog.roboflow.com/deploy-custom-model-weights-roboflow/). Aktuelni cenovnik ima Public plan bez plaćanja i bez kreditne kartice; druge usluge i planovi imaju svoja ograničenja i troškove. [Cenovnik](https://roboflow.com/pricing).

Nama trebaju **slike i anotacije**, ne njihov gotov model, API ili cloud trening. Trening ostaje lokalno na RTX 3060, prepoznavanje na telefonu. Ne treba kupovati pretplatu/kredite, unositi karticu, uključivati automatsku naplatu, preuzimati plaćene težine modela niti slati tvoje privatne slike u javni Roboflow projekat.

Automatski preglednik je pri pokušaju preuzimanja prikazao Cloudflare proveru „Verify you are human“. Provera nije zaobilažena. Nije potvrđen kompletan download kroz korisnički nalog niti je bilo šta plaćeno.

## Šta sada treba da uradiš

1. Otvori svaki od linkova za verziju 2 u svom pregledniku i, ako se traži, prijavi se besplatnim Roboflow nalogom.
2. Izaberi **Download / Download Dataset**, zatim **Download dataset**, format **COCO JSON** i **Download as ZIP**. Nemoj birati cloud trening ili preuzimanje njihovog gotovog modela. [Zvaničan primer COCO ZIP preuzimanja](https://blog.roboflow.com/zero-shot-vs-fine-tuned-models/).
3. Sačuvaj ZIP fajlove npr. u `C:\Users\veljk\Downloads`, ili u zaseban `FitnesApp\dataset\external\downloads` folder koji napraviš. Pošalji njihove lokalne putanje. Ne treba slati lozinku ili privatni API ključ.
4. Ako bilo koji korak traži novac, karticu ili kupovinu kredita, stani i pošalji prikaz poruke. Nema saglasnosti za plaćanje.

COCO izvoz zadržava nazive klasa i okvire objekata. Ne raspakuj ove slike direktno preko našeg `dataset/raw`: trenutna skripta očekuje drugačiji raspored i imenovanje izvora.

## Šta treba prilagoditi pre treninga

Oba skupa su za detekciju objekata, dok je trenutni model klasifikator centralnog kadra. Treba pročitati anotacije i pregledati primere, pa pripremiti odgovarajuće fotografije/isečke za klasifikaciju. Prisustvo više sprava u jednoj fotografiji ne sme nasumično postati jedna oznaka za celu sliku.

FitFuel ima opšte oznake `Chest Press Machine` i `Seated Row Machine`, a GymBro `chest-press`, `back-row-machine` i `seated-cable-row`. One **nisu automatski isto** što i naše specifične klase `converging_chest_press` i `plate_loaded_seated_row`. Potreban je izbor: proširiti model na opšte tipove sprava ili ručno izdvojiti samo potvrđene primere naših podtipova. Ne treba slepo preimenovati tuđe opšte klase u naše podtipove niti ih bez pregleda označiti kao `unknown`.

Plan pripreme kada ZIP fajlovi budu dostupni:

- Sačuvati originalne arhive, licence i podatke o izvoru odvojeno od aktivnog dataseta.
- Pregledati oznake, kvalitet, okvire objekata i duplikate između oba izvora i postojećih slika.
- Očuvati podele train/valid/test i držati varijante iste originalne slike zajedno. Ne mešati augmentovane kopije kroz podele; test ne koristiti za izbor epohe.
- Evidentirati autora, URL, verziju, licencu i stvarne izmene u katalogu/pratećem manifestu.
- Trenirati odvojenog kandidata lokalno, bez prepisivanja postojećeg modela aplikacije.

## Navođenje autora kada podaci budu korišćeni

Ovo su predlošci za navođenje autora. Model koji je već u aplikaciji još ne koristi ove datasetove; stvarna atribucija za novi lokalni kandidat sačuvana je u `dataset/external/roboflow_v2_20260920/ATTRIBUTION.md`:

> All Gym Equipment dataset v2 — FitFuel, via Roboflow Universe. CC BY 4.0. Source: https://universe.roboflow.com/fitfuel/all-gym-equipment/dataset/2 . License: https://creativecommons.org/licenses/by/4.0/ . Changes: [navesti stvarno izvršene izmene].

> GymBro dataset v2 — RizzLabzz, via Roboflow Universe. CC BY 4.0. Source: https://universe.roboflow.com/rizzlabzz/gymbro/dataset/2 . License: https://creativecommons.org/licenses/by/4.0/ . Changes: [navesti stvarno izvršene izmene].

Pri stvarnom uvozu dopuniti predloške, zadržati i eventualna dodatna postojeća obaveštenja iz arhiva i uključiti atribuciju u dokumentaciju modela i odgovarajući odeljak izvora/licenci aplikacije. Tvoje originalne fotografije ostaju zaseban izvor.
