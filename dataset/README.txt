DATASET ZA PREPOZNAVANJE SPRAVA

AKTUELNO (20.09.2026): lokalni PyTorch trening, ONNX model i SQLite katalog.
Slike: raw/<klasa>/, snimci: raw_video/<teretana>/, baza: catalog.sqlite3.
Katalog sadrzi putanje i metapodatke, ne same fotografije. U aplikaciju idu
equipment.onnx i config.json, ne dataset niti SQLite baza.
Uputstva: ../AI_machine_recognise/README.md i 06_LOKALNA_BAZA.md.
Uvoz iz fitness_plus: .venv-ai\Scripts\python.exe tool/ai_catalog.py sync

ISPOD JE SACUVAN RANIJI PLAN (Colab/TFLite/12 klasa), NE AKTUELNI TOK.

(prati planAI.txt, sekcije 5 i 7)

Ovaj folder je NAMERNO van fitness_plus/ - ima par GB i nema sta da trazi
u Flutter projektu ni u APK-u. U aplikaciju na kraju ide samo ~3 MB .tflite.


KAKO SE ZOVU FAJLOVI
--------------------------------------------------------------------
  raw_video/gym01/lat_pulldown_s01.mp4
            ^^^^^ ^^^^^^^^^^^^ ^^^
            |     |            +-- sesija: s01, s02... (drugi dan/drugo svetlo)
            |     +--------------- KLASA = tacno ime iz labels.txt
            +--------------------- teretana: gym01, gym02, ...

  gymNN u imenu nije ukras. ai_split_dataset.py deli train/val/test PO
  TERETANI, jer je jedino posteno pitanje "radi li ovo u sali koju model
  nije video". Ako to pomesas, dobices 97% u Colabu i 60% u teretani.

  Jedna teretana = jedan gymNN folder. Ista sprava snimljena drugi put
  (drugo svetlo, drugi dan) = ista klasa, druga sesija: _s02.


KLASE v1 (12 komada, planAI.txt sekcija 4)
--------------------------------------------------------------------
  lat_pulldown          seated_cable_row      cable_crossover
  chest_press_machine   shoulder_press_machine  pec_deck
  leg_press             leg_extension         leg_curl
  smith_machine         treadmill             unknown

  unknown = podovi, zidovi, ogledala, bucice, sipke, stalci, svlacionica,
  ljudi bez sprave, nasumicne kucne stvari, i SVAKA sprava koja nije na
  spisku gore. Bez ove klase softmax mora svaki zid da proglasi spravom.


KAKO SE SNIMA (planAI.txt 5.3 i 5.4)
--------------------------------------------------------------------
  1080p, 30 fps, 40-45 sekundi po spravi. Ne 4K.
  Polako obidji spravu u punom krugu. Menjaj visinu telefona dok hodas:
  koleno -> grudi -> iznad glave -> opet dole.
  Poslednjih 10 sekundi pridji blizu: jastuce, rucke, plocica sa
  uputstvom, stack tegova, logo.
  Uhvati sve uglove: front, 45, profil, 135, pozadi. Profil je presudan
  za leg_extension vs leg_curl - to je jedini ugao gde se vidi da li je
  jastuce iznad ili ispod potkolenice.

  Pitaj osoblje za dozvolu. Izbegavaj tudja lica u kadru.


POKRETANJE
--------------------------------------------------------------------
  fitness_plus\tool\ai_prepare.bat

  ili rucno:
    python fitness_plus/tool/ai_extract_frames.py
    python fitness_plus/tool/ai_split_dataset.py --val gym05 --test gym06

  raw_video/ -> raw/ (frejmovi, 1024 px) -> work/ (train|val|test, 256 px)
  work/ zipujes i saljes na Google Drive -> Colab.
  raw/ i work/ se generisu, slobodno ih brisi i pravi ponovo.
  raw_video/ je jedino nezamenljivo - njega bekapuj.
