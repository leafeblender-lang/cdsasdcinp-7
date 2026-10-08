# Ceo tok: od treninga do odluke na telefonu

[Početna strana](README.md) · [Trening](03_TRENING_KORAK_PO_KORAK.md) · [Sledeće: troškovi](05_TROSKOVI_I_ODRZAVANJE.md)

## Dva odvojena procesa

**Trening se radi na računaru.** Tu se čitaju tvoje označene slike, izračunavaju karakteristike, trenira klasifikacioni sloj i pravi verzija modela.

**Prepoznavanje se radi na telefonu.** Telefon dobija gotov model kroz instaliranu aplikaciju i koristi ga na novim kadrovima. Za to ne poziva tvoj računar, GPU u računaru, Colab ili AI servis.

SQLite baza je katalog podataka za rad na računaru. Telefon je ne preuzima i ne čita pri skeniranju. Nova slika dodata u bazu utiče na aplikaciju tek nakon novog treninga i isporuke odgovarajućeg modela.

Računar može biti ugašen dok korisnik skenira. Telefon ne mora imati NVIDIA grafičku. ONNX Runtime omogućava lokalno izvršavanje modela na mobilnim uređajima; naš kod ne zahteva GPU na telefonu. [Dokumentacija ONNX Runtime-a](https://onnxruntime.ai/docs/tutorials/mobile/).

## Put podataka pri treningu

1. Video dobije ID klase i sesije kroz ime i folder.
2. Iz videa se izdvoje odabrani JPG kadrovi.
3. Osveži se SQLite katalog; trening uzima aktivne ispravne slike iz foldera klasa, poštujući podelu celih izvora iz baze.
4. Slika se orijentiše, iseče na centralni kvadrat, smanji na 224 × 224 i normalizuje.
5. Osnovni MobileNet izračuna vizuelne karakteristike. Njegove težine se ne menjaju.
6. Novi klasifikacioni sloj uči razliku među klasama; trening slike dobijaju i augmentovane prikaze.
7. Izabere se epoha sa najmanjom validacionom greškom.
8. Model se dopuni vizuelnim predstavnicima trening slika za računanje sličnosti.
9. Izveze se `equipment.onnx` i proveri da izvoz reprodukuje PyTorch odluke.
10. Upisuju se konfiguracija, izveštaj i ostali pomoćni fajlovi.
11. Odabrani ONNX i JSON ubace se u aplikaciju i napravi se novi APK.

Model je klasifikator centralnog kadra: ne pronalazi i ne označava svaku spravu zasebnim okvirom. Ako u kadar staviš više sličnih sprava, ne postoji garancija da će odabrati onu na koju ti misliš.

## Put podataka tokom skeniranja uživo

Korisnik otvori **Scan equipment**. Aplikacija učita konfiguraciju i lokalni ONNX fajl, pa pokrene kameru. Jedna obrada kadra može da bude aktivna u datom trenutku; ne gomilaju se svi frejmovi videa.

Nominalno se novi kadar razmatra približno svakih 400 ms, dakle oko 2,5 puta u sekundi. Ako uređaj sporije obrađuje slike, stvarna učestalost je niža. To nije obećanje stalnog broja prepoznavanja u sekundi na svakom telefonu.

Kadar prolazi kroz konverziju boja, korekciju orijentacije senzora i izdvajanje centralnog kvadrata. Ulaz modela je RGB slika od 224 × 224 piksela, zapakovana u niz oblika `1 × 3 × 224 × 224`.

Normalizacija po RGB kanalima je:

```text
vrednost = (piksel / 255 - mean) / std
mean = [0.485, 0.456, 0.406]
std  = [0.229, 0.224, 0.225]
```

Obrada fotografije i kamere ima isti centralni kadar, raspored kanala i normalizaciju, ali njihovi postupci promene rezolucije nisu bajt-po-bajt identični Python obradi. Foto put koristi prosečno filtriranje, a kamera direktno uzorkuje kadar. Zato provera ONNX/PyTorch izvoza nije zamena za probu stvarnog mobilnog toka.

Model vraća po jedan skor klase i skor vizuelne sličnosti za svaku klasu. Dart kod na osnovu njih odlučuje da li kandidat može biti prihvaćen.

## Trenutna pravila prihvatanja

Izvor pravila je [`config.json`](../fitness_plus/assets/ml/equipment_scan/config.json).

| Pravilo | Trenutna vrednost | Značenje |
|---|---:|---|
| `min_probability` | 0,88 | Minimalan softmax skor prvoplasirane klase. |
| `min_margin` | 0,40 | Minimalna razlika skorova prve i druge klase. |
| `min_similarity` | 0,80 za svaku klasu | Minimalna sličnost sa bar jednim trening predstavnikom te klase. |
| `stable_frames` | 3 | Ista prihvatljiva klasa u tri uzastopne obrade. |
| `max_frame_gap_ms` | 1800 | Preveliki razmak prekida sakupljanje uzastopnih potvrda. |
| `scan_timeout_ms` | 12000 | Skeniranje prestaje ako nema stabilnog rezultata za 12 s. |

Slab ili drugačiji rezultat prekida prethodni niz potvrda. Kasni rezultat ne sme da pretvori već isteklo skeniranje u uspešno. Eksplicitna klasa `unknown`, kada bude dodata, nikad se ne prikazuje kao uspešno prepoznata sprava.

Za ručno odabranu ili uslikanu fotografiju koriste se isti pragovi modela, ali nema tri uzastopna kadra: fotografija se procenjuje jednom.

Greška kamere, učitavanja modela ili izvršavanja prikazuje se kao tehnički problem. Ona nije dokaz da sprava ne postoji u datasetu. Ako tokom skeniranja uopšte nema uspešno obrađenih kadrova, ekran prikazuje problem kamere.

## Zašto 50% nije dobar uslov

Softmax raspodeljuje skor među poznatim klasama tako da njihov zbir bude približno 1. Sa dve klase bar jedna ima skor najmanje 0,5, čak i kada fotografija uopšte ne prikazuje jednu od te dve sprave. Zato `>= 50%` praktično ne omogućava smisleno odbacivanje nepoznatih objekata.

Ni skor 0,99 nije dokaz 99% stvarne tačnosti. Dodatna sličnost i tri uzastopna prolaza pomažu kao početna pravila, ali ne čine model pouzdanim za sve nepoznate predmete. Ista pogrešna pretpostavka može biti stabilna kroz više kadrova.

Poruka **Equipment not recognized** znači da nije dobijeno dovoljno dobro poklapanje. Ona ne dokazuje da sprave nema u sistemu: i poznata sprava može biti odbijena zbog ugla, kadra, zaklona ili osvetljenja.

## Šta sada ograničava kvalitet

Postoje samo dva snimka po klasi i to iz jedne teretane. Model može zapamtiti boju, okvir, pod ili pozadinu vezanu za konkretne primerke. Prepoznavanje drugog proizvođača iste vrste sprave nije potvrđeno.

Još nema označenih stvarnih negativnih primera. Zato odbacivanje drugih sprava nije izmereno. Povremeno pogrešno prihvatanje ne može se isključiti samo podizanjem praga.

Validacija je korišćena za izbor najbolje epohe. Ne postoji zaseban nezavisni test skup koji nije uticao na izbor modela. Izveštaj treba čitati kao rezultat razvojnog eksperimenta sa konkretnim snimcima.

Funkcionalnost trenutno ne čita natpise na spravi, ne prepoznaje barkod, ne prikazuje vezane vežbe i ne uči automatski iz ispravki korisnika.

## Kako da tumačiš izveštaj pri sledećem treningu

| Polje | Značenje i ograničenje |
|---|---|
| `validation_accuracy` | Tačan prvi izbor klase pre pragova. |
| `accepted_correct` | Prihvaćeni rezultati čija klasa odgovara oznaci slike. |
| `accepted_wrong` | Prihvaćeni rezultati sa pogrešnom klasom; ovu meru posebno prati. |
| `rejected_known` | U novim izveštajima broji odbijene poznate klase; odbijanje eksplicitnih `unknown` primera posebno je u `rejected_unknown`. Stara verzija je brojala sva odbijanja. |
| `validation_predictions` | Rezultat po slici: stvarna oznaka, predikcija, skor, sličnost i prihvatanje. |
| `unknown_validation_available` | Da li u ovom datasetu postoji `unknown` klasa. |
| `onnx_max_absolute_error` | Razlika numeričkih rezultata Python ONNX i PyTorch modela. |

Kada dodaš `unknown`, zasebno pregledaj njegove rezultate i rezultate poznatih klasa u `validation_predictions`. Zbirna trenutna polja nisu potpuna analiza odbacivanja nepoznatih sprava.

Novi niz `limitations` opisuje opšta ograničenja validacije i porekla, umesto tvrdnje da svaki dataset ima samo dva snimka. Za stvarnu podelu gledaj `split_manifest.json`. Novi izveštaji beleže i broj raspoloživih test slika i da one nisu automatski ocenjene.

## Menjanje pragova

Pragovi za aplikaciju nalaze se u `assets/ml/equipment_scan/config.json`. Podrazumevane vrednosti koje trening upisuje definisane su u rečniku `thresholds` unutar `tool/ai_train_local.py`.

Ako promeniš samo JSON pa kasnije ponoviš trening na istu izlaznu putanju, trening će ponovo upisati vrednosti iz Python skripte. Za trajnu promenu podrazumevanog ponašanja treba održavati oba dela kroz postupak izvoza. Pragovi izmenjeni nakon treninga takođe znače da prethodno sačuvane brojke o prihvatanju više ne opisuju novu odluku.

Viši prag obično odbija više kadrova, uključujući i neke dobre. Niži može lakše prepoznati poznatu spravu, ali i prihvatiti pogrešnu. Podešavanje treba zasnivati na novim poznatim i nepoznatim primerima, a ne na želji da ekran češće prikazuje rezultat.

Ne menjaj `input_size`, normalizaciju, raspored kanala, izlaze niti redosled `labels` bez odgovarajuće izmene/izvoza modela. To nisu samo estetska podešavanja.

## Životni ciklus i privatnost podataka

Kada aplikacija ode u pozadinu, kamera se oslobađa. Pri ponovnom skeniranju prethodne potvrde se ne nastavljaju kao da su novi kadrovi. Inference pozivi se izvršavaju serijski, a resursi modela oslobađaju se pri zatvaranju skenera.

Model se pri učitavanju kopira iz trenutne aplikacije u njen privatni privremeni folder. Time se izbegava problem korišćenja starog modela iz keša posle ažuriranja aplikacije. Pri urednom zatvaranju briše se ta privremena kopija; Android/iOS mogu zadržati keš nakon neočekivanog prekida procesa.

Skener ne šalje kadrove na spoljne servere i ne dodaje ih u bazu trening slika. Postojeći drugi delovi fitnes aplikacije mogu koristiti mrežu za svoje funkcije; tvrdnja o offline radu odnosi se na prepoznavanje sprava.

## Šta bi bila naredna poboljšanja

Prvo prikupi nove uglove poznatih sprava i stvarne negativne primere. Zatim proširi broj različitih primeraka svake vrste, odvoji novu teretanu za proveru i podesi pragove na tim podacima. Tek ako se pokaže potreba, slede promene modela, detekcija više objekata, čitanje natpisa ili server za verzije modela.

Za veliki dataset treba ponovo razmotriti čuvanje svih trening predstavnika i učitavanje svih njihovih augmentovanih karakteristika u memoriju. Trenutno rešenje je namenjeno početnom skupu; rast broja slika i klasa može povećati model, RAM/GPU memoriju i vreme obrade.
