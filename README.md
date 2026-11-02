# RC car – autko zdalnie sterowane na ESP32

Autko RC z napędem na tylną oś, sterowane mikrokontrolerem **ESP32**. Wszystkie części mechaniczne zaprojektowałem samodzielnie w **Fusion 360** i wydrukowałem na **Bambu Lab P1S**.
To mój projekt rekrutacyjny do koła naukowego **SKAR**.

> Inspiracja: [3D Printed RC car – TommyB (Printables)](https://www.printables.com/model/415497). Projekt jest wzorowany na tym autku, ale każdą część zaprojektowałem od nowa pod własne wymiary i podzespoły.

---

## Spis treści
- [Parametry](#parametry)
- [Podzespoły (BOM)](#podzespoły-bom)
- [Mechanika](#mechanika)
- [Elektronika](#elektronika)
- [Oprogramowanie](#oprogramowanie)
- [Druk 3D](#druk-3d)
- [Struktura repozytorium](#struktura-repozytorium)
- [Problemy i rozwiązania](#problemy-i-rozwiązania)
- [Status projektu](#status-projektu)
- [Plany rozwoju](#plany-rozwoju)

---

## Parametry

| Parametr | Wartość |
|---|---|
| Rozstaw osi | 175 mm |
| Rozstaw kół (przód i tył) | 136 mm |
| Szerokość podwozia | 96 mm |
| Koła | Ø62 × 26 mm (felgi 1/10, offset 2,5 mm) |
| Napęd | tylna oś, silnik DC przez przekładnię zębatą 24:20 |
| Maks. kąt skrętu | 30° (koło wewnętrzne) / 26,7° (koło zewnętrzne) |
| Zasilanie | 2S Li-ion (2 × 18650), 7,4 V nominalnie |
| Prędkość (obliczona) | ok. 11,8 km/h bez obciążenia, realnie ok. 8–9 km/h |
| Sterowanie | pad PS5 DualSense przez Bluetooth |

---

## Podzespoły (BOM)

### Elektronika
| Element | Model | Ilość | Uwagi |
|---|---|---|---|
| Mikrokontroler | ESP32 DevKit (WROOM) | 1 | wymagany Bluetooth Classic (pad PS5) |
| Sterownik silnika | L9110 (moduł 2-kanałowy) | 1 | kanały połączone równolegle |
| Silnik napędowy | GA25-370, 1360 obr/min @ 12 V | 1 | |
| Serwo skrętu | MG996R | 1 | |
| Przetwornica | Mini560 (step-down 5 V) | 1 | zasila ESP32 i serwo |
| Ogniwa | 18650 Li-ion | 2 | w koszyku 2 × 18650 |
| BMS | 2S | 1 | ochrona ogniw |
| Włącznik | – | 1 | na plusie baterii |
| Kondensator elektrolityczny | 1000 µF / 16 V | 1 | przy złączu serwa |
| Kondensator elektrolityczny | 100–470 µF / 16 V | 1 | przy zasilaniu L9110 |
| Kondensator ceramiczny | 100 nF („104”) | 1 | na stykach silnika |
| Płytka uniwersalna | 10 × 7 cm | 1 | rozdzielnia zasilania + gniazdo ESP32 |
| Goldpiny męskie i żeńskie, przewody 20–22| – | – | |

### Mechanika
| Element | Ilość | Uwagi |
|---|---|---|
| Łożyska 4 × 9 × 4 mm | 2+ | tylna oś |
| Wałek Ø4 mm (h6/g6), długość 150 mm | 1 | tylna oś |
| Felgi 1/10 + opony z wkładkami piankowymi | 4 | |
| Śruby M3 + nakrętki samokontrujące (nyloc) + podkładki | kpl. | przeguby układu kierowniczego, mocowania |
| Rzep / pasek z rzepem | 1 | mocowanie baterii |

---

## Mechanika

### Napęd
- Silnik GA25-370 napędza tylną oś przez **przekładnię zębatą 24:20** (24 zęby na silniku, 20 na osi), czyli przełożenie **1,2** „na szybkość”.
- Zębatka silnika jest **wciskana** na wałek z płaskiem (otwór D, po korektach **4,3 mm** w modelu).
- Koła są zamocowane na **adapterach hex 12 mm**, a oś pracuje na łożyskach 4 × 9 × 4 mm w obudowach przykręcanych do podwozia.

**Obliczenie prędkości**

```
obroty silnika przy 7,4 V:  1360 · 7,4 / 12 ≈ 840 obr/min
obroty kół:                 840 · 24 / 20  = 1008 obr/min = 16,8 obr/s
obwód koła:                 π · 0,062 m    = 0,195 m
prędkość:                   16,8 · 0,195   = 3,27 m/s ≈ 11,8 km/h (bez obciążenia)
```

Realnie, z obciążeniem, tarciem i spadkiem napięcia na sterowniku, wychodzi ok. **8–9 km/h**.

### Układ kierowniczy
- Zwrotnice obracają się na sworzniach (śruby M3 z nakrętkami samokontrującymi).
- Zwrotnice są połączone **drążkiem kierowniczym**, a serwo MG996R porusza drążek przez ramię ustawione na osi symetrii autka.
- Geometria jest zaprojektowana według zasady **Ackermanna**: przy skręcie koło wewnętrzne skręca bardziej niż zewnętrzne, więc oba toczą się po okręgach o wspólnym środku.

**Weryfikacja w Fusion 360 (Drive Joints):**

| | Koło wewnętrzne | Koło zewnętrzne |
|---|---|---|
| Zmierzone | 30° | 26,7° |
| Idealny Ackermann | 30° | 24,3° |

```
cot δo = cot δi + t / L = cot 30° + 84 / 175 ≈ 2,212  →  δo ≈ 24,3°
(t – rozstaw sworzni ≈ 84 mm, L – rozstaw osi 175 mm)

procent Ackermanna = (30 − 26,7) / (30 − 24,3) ≈ 58%
```

Wyniki są lustrzane dla skrętu w lewo i w prawo. 50–80% Ackermanna to typowa wartość w modelach RC.

---

## Elektronika

### Schemat zasilania

```
Ogniwa 2S ── BMS ── włącznik ──┬── szyna 7,4 V ──┬── L9110 (VCC) ── silnik DC
                               │                 │
                               │                 └── Mini560 (IN) ── szyna 5 V ──┬── serwo MG996R
                               │                                                 └── ESP32 (5V/VIN)
                               │
                               └── wspólna masa (GND) dla wszystkich modułów
```

### Połączenia ESP32

| ESP32 | Do | Opis |
|---|---|---|
| GPIO25 | L9110: A-IA + B-IA | PWM – jazda do przodu |
| GPIO26 | L9110: A-IB + B-IB | PWM – jazda do tyłu |
| GPIO18 | serwo (sygnał) | sterowanie skrętem |
| GPIO34 | dzielnik napięcia baterii (planowany) | pomiar napięcia |
| 5V / VIN | szyna 5 V | zasilanie z Mini560 |
| GND | szyna GND | wspólna masa |

### Najważniejsze decyzje
- **L9110 z kanałami równolegle:** jeden kanał daje ok. 0,8 A, a dwa razem ok. 1,6 A. Wejścia i wyjścia kanałów są połączone parami w tej samej kolejności.
- **Wspólna masa** baterii, przetwornicy, sterownika, serwa i ESP32. Bez niej sygnały sterujące nie działają poprawnie.
- **Kondensator 1000 µF przy serwie:** przy szarpnięciu serwo pobiera duży prąd. Kondensator zapobiega spadkom napięcia i resetom ESP32.
- **Kondensator 100 nF na silniku:** tłumi zakłócenia od szczotek, które mogą psuć łączność Bluetooth.
- **ESP32 w gniazdach goldpin:** można go wyjąć do programowania albo wymienić.
- **Dzielnik napięcia (planowany):** 100 kΩ / 47 kΩ. Przy 8,4 V na wejściu daje ok. 2,7 V na GPIO34, czyli bezpiecznie poniżej 3,3 V.

---

## Oprogramowanie

> W trakcie – kod trafi do folderu `firmware/`.

Plan:
- **PlatformIO** + framework Arduino
- **Bluepad32** do obsługi pada PS5 DualSense przez Bluetooth Classic
- sterowanie:
  - lewa gałka (oś X) → kąt serwa
  - R2 → gaz (PWM na GPIO25)
  - L2 → wsteczny (PWM na GPIO26)
- **failsafe:** po rozłączeniu pada silnik od razu się zatrzymuje

### Parowanie pada
Przytrzymaj **PS + Create**, aż pasek świetlny zacznie szybko migać. ESP32 połączy się automatycznie.

---

## Druk 3D

**Drukarka:** Bambu Lab P1S | **Slicer:** OrcaSlicer 2.4.2

| Część | Warstwa | Ściany | Wypełnienie | Supporty | Uwagi |
|---|---|---|---|---|---|
| Zębatki | 0,16 mm | 5 | 50–60% gyroid | nie | kompensacja „stopy słonia” 0,15, bez brimu |
| Zwrotnice | 0,2 mm | 4–5 | 50% gyroid | jeśli nawis | ramię płasko na stole, otwór sworznia pionowo |
| Drążek kierowniczy | 0,2 mm | 4–5 | 50% gyroid | tree (auto), tylko od stołu | brim zewnętrzny |
| Adapter hex | 0,2 mm | 4–5 | 50% | nie | włączone wykrywanie cienkich ścian (Arachne) |
| Ramię serwa | 0,2 mm | 4–5 | 50% | nie | płasko na stole |
| Podwozie | 0,2 mm | 3–4 | 15–20% | nie | brim zewnętrzny 5 mm |

### Pasowania (druk – część)
| Połączenie | Otwór w modelu |
|---|---|
| Śruba M3 przelotowo | Ø3,4 mm |
| Wkręt M3 w plastik | Ø2,8 mm |
| Zębatka na wałku silnika (wcisk) | D 4,3 mm |
| Adapter hex do felgi | 12,0–12,05 mm |
| Gniazdo łożyska 4 × 9 × 4 | Ø9,2 mm z kołnierzem Ø7 mm |

---

## Struktura repozytorium

```
rc-car/
├── README.md
├── cad/
│   ├── stl/        ← pliki do druku
│   └── 3fd/       ← gotowy model do Fusion360
├── firmware/       ← kod ESP32 (PlatformIO)  [w trakcie]
├── electronics/    ← schemat połączeń          [w trakcie]
└── docs/           ← zdjęcia, rendery
```

### Pliki STL
| Plik | Opis |
|---|---|
| `podwozie.stl` | podwozie |
| `zwrotnicaL.stl`, `zwrotnicaP.stl` | zwrotnice lewa i prawa |
| `drazek_acermann.stl` | drążek kierowniczy |
| `zebatka1.stl`, `zebatka2.stl` | zębatki przekładni napędowej |
| `adapter.stl` | adapter hex 12 mm koła |
| `holderLOZYSKA.stl` | obudowa łożyska tylnej osi |
| `zacisk.stl` | zacisk |
| `kolo.stl` | koło (model poglądowy) |
| `walek.stl` | wałek tylnej osi (model poglądowy, w autku stalowy wałek Ø4 mm) |
| `rc car.stl` | całe złożenie (podgląd) |

---

## Status projektu

- [x] Projekt tylnej osi, napędu i obudów łożysk
- [x] Projekt układu kierowniczego z weryfikacją geometrii Ackermanna
- [x] Druk zębatek i dopasowanie pasowań
- [ ] Mocowanie serwa i płytki elektroniki w podwoziu
- [ ] Druk i montaż układu kierowniczego
- [ ] Lutowanie płytki zasilania
- [ ] Oprogramowanie (pad PS5, failsafe)
- [ ] Pierwsza jazda i pomiar prędkośc

---

## Autor
**kminek22** – student Automatyki i Robotyki, Politechnika Gdańska