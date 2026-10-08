# Šta je urađeno i koje tehnologije se koriste

[Početna strana](README.md) · [Sledeće: podaci i folderi](02_PODACI_I_FOLDERI.md)

## Rezultat

U postojeću Flutter aplikaciju dodat je ekran koji prepoznaje spravu sa kamere, nove fotografije ili slike iz galerije. Model je istreniran na tvojim snimcima i upakovan u aplikaciju. Nije postavljen zaseban AI server.

Obrađene su ove dve klase:

| ID klase | Naziv koji aplikacija prikazuje | Ukupno slika | Trening | Validacija |
|---|---|---:|---:|---:|
| `converging_chest_press` | Converging chest press machine | 29 | 13 | 16 |
| `plate_loaded_seated_row` | Plate-loaded seated row machine | 68 | 33 | 35 |
| Ukupno | | 97 | 46 | 51 |

Svaka klasa ima dva video-snimka. Prvi snimak je korišćen za trening, drugi za validaciju. Originalni desktop snimci su sačuvani.

## Šta je zatečeno, a šta je dovršeno

Već su postojali opšti plan `planAI.txt`, skripte za izdvajanje/podelu podataka i Colab notebook. Četiri videa su već bila organizovana, a 97 kadrova izdvojeno. Nije postojao završen model uključen u aplikaciju.

Dovršeno je:

1. Potvrđena je RTX 3060 sa 12 GB memorije i funkcionalan PyTorch/CUDA.
2. Napravljeno je projektno Python okruženje `.venv-ai` i instalirane dodatne biblioteke.
3. Napisan je lokalni trening koji odvaja čitave izvorne snimke za validaciju.
4. Istreniran je klasifikator za dve sprave i izvezen ONNX model.
5. Izvoz je upoređen sa PyTorch rezultatom na svih 51 validacionih slika.
6. Dodati su Flutter skener, obrada slike, izvršavanje modela i pravila prihvatanja.
7. Dodati su kamera/fotografija/galerija, čekanje stabilnog rezultata, timeout i obrada grešaka.
8. Napravljen je Android debug APK i otvoren skener na emulatoru.
9. Dodati su CLI za fotografije, uputstvo i testovi koji mogu da se pokrenu kasnije.
10. Naknadno je napravljena lokalna SQLite baza sa 97 slika, proverom duplikata, isključivanjem primera, podelom izvora i backup komandama. Trening i video-ekstrakcija povezani su sa katalogom; detalji su u [dokumentu 06](06_LOKALNA_BAZA.md).

Plan koji se oslanjao na TensorFlow/Colab nije korišćen za ovaj završeni model. Nije bilo potrebe da se instalira TensorFlow niti da se prijavljuješ na Colab.

## Biblioteke i alati

Verzije ispod opisuju korišćeno okruženje, a nisu zahtev da ubuduće instaliraš najnovije verzije svega.

| Alat | Korišćena verzija | Uloga |
|---|---|---|
| Python | 3.10.6 | Pokretanje pripreme podataka i treninga. |
| SQLite preko `sqlite3` | 3.37.2 | Lokalni katalog fotografija; već dostupan kroz Python, bez servera. |
| PyTorch | 2.5.1+cu121 | Model i trening pomoću postojeće CUDA instalacije. |
| torchvision | 0.20.1+cu121 | MobileNetV3-Small i transformacije slika. |
| NVIDIA GPU | RTX 3060, 12 GB | Lokalna obrada slika za trening. |
| OpenCV | 4.9.0.80 paket | Čitanje videa, procena zamućenosti i sličnosti kadrova. |
| NumPy | 1.26.4 | Obrada numeričkih nizova. |
| Pillow | 11.3.0 u lokalnom okruženju | Učitavanje slika, orijentacija, crop i resize. |
| ONNX | 1.17.0 | Format izvoza i provera strukture modela. |
| ONNX Runtime za Python | 1.20.1 | Lokalna provera izvezenog modela i CLI prepoznavanje. |
| Flutter / Dart | 3.47.0 / 3.13.0 | Postojeća mobilna aplikacija. |
| `camera` | 0.12.1 | Pregled kamere i tok kadrova. |
| `flutter_onnxruntime` | 1.8.5 | Pozivanje nativnog ONNX Runtime-a iz Flutter-a. |
| ONNX Runtime na Androidu | 1.23.0 | Izvršavanje modela na telefonu. |
| `image_picker` | 1.2.3 | Slikanje i izbor fotografije; već je bio u projektu. |
| `image` | 4.9.1 | Obrada odabrane fotografije u Dart-u. |

Python zahtevi su u [`ai_local_requirements.txt`](../fitness_plus/tool/ai_local_requirements.txt), a Flutter zavisnosti u [`pubspec.yaml`](../fitness_plus/pubspec.yaml) i `pubspec.lock`.

`.venv-ai` je napravljen sa `--system-site-packages`: koristi već instalirane PyTorch/torchvision biblioteke. Zato to okruženje nije potpuno samostalno kada ga prenosiš na drugi računar. Globalni PyTorch nije zamenjen.

## Šta je tačno „moj istrenirani model“

Osnova je **MobileNetV3-Small**, sa prethodno naučenim težinama `IMAGENET1K_V1`. To je postojeći model za vizuelne karakteristike, koji je dostupan preko [TorchVision dokumentacije](https://docs.pytorch.org/vision/stable/models/generated/torchvision.models.mobilenet_v3_small.html).

U našem kodu osnovni deo modela je zamrznut. On iz slike izvlači vektor od 576 brojeva. Na tvojim primerima treniran je novi završni sloj koji te karakteristike povezuje sa nazivima sprava. Ovo je transfer learning: koristi se ranije stečeno vizuelno znanje, a novi zadatak se uči na tvom datasetu.

Pored toga, model sadrži normalizovane vektore trening slika. Za novu sliku izračunava najveću kosinusnu sličnost sa primerima svake klase. Taj dodatni skor služi kao uslov prihvatanja. U modelu nisu upakovane originalne JPG fotografije.

Trenutni podrazumevani trening koristi:

- 120 epoha za novi klasifikacioni sloj;
- osam nasumično izmenjenih pogleda svake trening slike, uz originalni centralni crop;
- promene osvetljenja/kontrasta, mali nagib, crop i horizontalno okretanje;
- AdamW optimizator i težine klasa radi ublažavanja neravnoteže broja slika;
- izbor epohe sa najmanjom validacionom greškom; u ovom pokretanju to je epoha 28.

Augmentacija povećava raznovrsnost prikaza, ali ne pravi nove nezavisne sprave ili nove teretane.

## Rezultati postojeće verzije

Izvor: [`evaluation.json`](../fitness_plus/artifacts/equipment_scan/evaluation.json).

| Mera | Rezultat |
|---|---:|
| Tačan prvi izbor klase, pre pragova | 51 / 51 |
| Prihvaćeni tačni rezultati sa pragovima | 34 / 51 |
| Prihvaćeni pogrešni rezultati | 0 / 51 |
| Odbijeni kadrovi poznatih sprava | 17 / 51 |
| Nezavisni primeri drugih sprava u validaciji | Nema ih |
| Veličina ONNX modela | 3.834.334 bajta, oko 3,8 MB |
| Najveća razlika ONNX/PyTorch skorova | Oko 0,00000143 |

Ovo nisu rezultati sa potpuno novog terenskog skupa. Drugi snimci su iz iste teretane, a validacija je korišćena i za izbor epohe. Zato rezultat 51/51 ne znači da će aplikacija pravilno prepoznati svaku fotografiju niti da pouzdano odbacuje druge sprave.

## Promene u aplikaciji

Glavna implementacija je u [`lib/features/equipment_scan`](../fitness_plus/lib/features/equipment_scan/):

| Fajl | Zadatak |
|---|---|
| `domain/equipment_scan.dart` | Konfiguracija, pragovi i uzastopne potvrde. |
| `data/equipment_preprocessing.dart` | Priprema fotografije i kadrova kamere. |
| `data/equipment_recognizer.dart` | Učitavanje modela i pozivanje ONNX Runtime-a. |
| `presentation/equipment_scan_screen.dart` | Ekran, kamera, galerija, rezultat i greške. |

Ulaz u skener dodat je u `lib/screens/home_screen.dart`. Tekstovi su u `lib/l10n/app_en.arb`. Model i konfiguracija registrovani su kao Flutter assets. Dodate su dozvole kamere i Android pravila za ONNX JNI klase pri optimizaciji aplikacije.

Android minimum ostao je API 24. iOS minimum u projektu podignut je sa 13 na **16**, koliko zahteva izabrani [Flutter ONNX paket](https://pub.dev/packages/flutter_onnxruntime). iOS build nije potvrđen na Macu.

## Šta je provereno, a šta ostaje

Potvrđeni su CUDA rad, lokalni trening, ONNX izvoz, Android debug build, instalacija na postojećem emulatoru i otvaranje skenera sa učitanim modelom i kamerom bez prijavljenih grešaka.

Detaljni Flutter i integracioni testovi pripremljeni su, ali nisu kompletno pokretani, prema dogovoru da se prvo završi funkcionalnost. Nije obavljena proba kamerom na fizičkom telefonu ispred stvarnih sprava niti nezavisno merenje prepoznavanja drugih sprava.

Nisu implementirani automatsko prikupljanje korisničkih slika, cloud trening, automatska isporuka novih modela, OCR natpisa, povezivanje rezultata sa vežbama iz kataloga ili detekcija više sprava sa okvirima. Trenutni rezultat je naziv prepoznate klase.
