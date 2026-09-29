# Light Orb Shader: projekt

Data: 2026-09-29
Status: zatwierdzony w rozmowie, czeka na przegląd specu

## Cel

Wierne odtworzenie 1:1 efektu z `animation_02.mp4` jako demo w przeglądarce, zbudowane na
technice z [light-ribbon-shader](https://github.com/marcinkwiatkowski605/light-ribbon-shader).
Świecąca kula z zamkniętą wstęgą światła w kształcie siodła w środku. Brak interakcji poza
panelem z suwakami.

Poza zakresem: reakcja na mikrofon, kursor czy dotyk, eksport wideo, build step, frameworki.

## Referencja (zmierzone z wideo)

| Cecha | Wartość |
| --- | --- |
| Rozdzielczość / FPS / długość | 1440×1080, 30 fps, 15 s |
| Pętla | idealna, okres **5,0 s** (różnica klatek 0 s vs 5 s: 0,16/255), 3 powtórzenia |
| Pozycja orbu | dokładnie środek kadru (719, 540) |
| Średnica orbu z poświatą (piksele > 12/255) | ~480 px = **~44% wysokości kadru** |
| Tło | czysto czarne `#000000` |
| Średni błękit / najjaśniejszy | `#4c5e8f` / `#bcc3dd` |
| Średni bursztyn / najjaśniejszy | `#b27239` / `#f5d7b0` |
| Ziarno, aberracja chromatyczna | brak |

Opis jakościowy:
- Kula to miękka, prawie wolumetryczna poświata: jasny brzeg (Fresnel) i ciemniejszy środek.
  Dolna połowa wygląda jak misa podświetlona od środka, górna czasza jest ciemniejsza.
- Błękit i bursztyn układają się na kuli jako dwa ukośne płaty, które w trakcie pętli
  zamieniają się półkulami.
- Wstęga to zamknięta pętla w kształcie siodła (wygięty pierścień), wpisana w kulę. Ma
  bursztynowy półprzezroczysty korpus i kremową, najjaśniejszą krawędź tam, gdzie pas
  odwraca się bokiem do kamery. Grzbiet siodła wędruje dookoła (precesja).
- Wszystko jest mocno rozmyte i ma bloom.

## Podejście

Hybryda w Three.js (wariant A), czyli ten sam szkielet co w light-ribbon-shader: geometria
liczona w vertex shaderze, addytywne światło (fold + rim), łańcuch post-processingu, lil-gui.
Odrzucone: pełny raymarching (inna architektura, ciężki dla GPU, trudny do strojenia) oraz
kompozycja 2D (brak prawdziwej precesji 3D).

## Pliki (nowe repo `light-orb-shader`)

- `index.html`: cały efekt w jednym pliku, bez builda. Three.js r170 i lil-gui 0.19.2 z jsDelivr
  przez importmap, jak w oryginale.
- `README.md`, `LICENSE` (MIT, © 2026 Marcin Kwiatkowski), `THIRD_PARTY_NOTICES.md`,
  `preview.jpg`, `.gitignore`, `.gitattributes`.
- `docs/superpowers/specs/`: ten dokument.

Narzędzia weryfikacji (serwer do zapisu klatek, skrypt porównujący) żyją w scratchpadzie
sesji, a nie w repo.

## Scena

Trzy warstwy w grupie `orbGroup`, renderowane addytywnie (`AdditiveBlending`), bez depth testu
i depth write, `DoubleSide`:

1. **Powłoka kuli**: `SphereGeometry(R, 96, 64)`.
2. **Wstęga siodłowa**: `PlaneGeometry(1, 1, 256, 24)`, której wierzchołki vertex shader
   przebudowuje z `uv`.
3. **Halo wstęgi**: ta sama geometria i shader z definem `SOFT_CORE`, szerszy pas, miękki
   gaussian w poprzek szerokości.

### Geometria wstęgi

Dla `θ = 2π·u` i `v ∈ [−1, 1]`:

```
centre(θ) = R_r · ( cos θ,  h · sin(2θ + φ),  sin θ )
p(θ, v)   = centre(θ) + v · w · B(θ)
```

- `R_r`: promień pętli (ułamek promienia kuli, domyślnie ~0,78·R),
- `h`: wysokość siodła,
- `φ`: faza precesji,
- `B(θ)`: kierunek w poprzek pasa, czyli głównie oś Y, obrócona wokół stycznej o kąt skrętu
  `twist · sin(θ + φ)`, co daje zmienną szerokość pasa widzianą z kamery,
- normalne z różnic skończonych, jak w oryginale, z zabezpieczeniem przed zerową długością.

Pętla jest zamknięta (`u = 0` i `u = 1` dają ten sam punkt), więc nie ma envelope zwężającego
końce. Szerokość `w` jest stała.

### Materiał powłoki

- `fres = pow(1 − |n·v|, fresnelPower)`,
- przednia ścianka: `fres · shellIntensity`; tylna ścianka (`gl_FrontFacing == false`):
  `fres · interiorIntensity` (wnętrze misy),
- gradient pionowy: mnożnik `mix(topBrightness, 1, smoothstep(+0,6, −0,6, n.y))`, czyli
  ciemniejsza góra i jaśniejszy dół,
- kolor: `k = dot(n, lobeDir(t))`, `col = mix(cool, warm, smoothstep(−s, s, k))`. `lobeDir`
  jest pochylony o `lobeTilt` względem osi Y i obraca się wokół niej.

### Materiał wstęgi

Fragment shader z oryginału w nowej palecie:
- korpus: `bodyColor` (bursztyn) × `bodyIntensity`, jaśniejszy, gdy pas patrzy na kamerę,
- fold: `pow(1 − facing, foldPower) · foldIntensity` w kolorze kremowo-białym,
- rimy na obu krawędziach: kremowy (`rimColorA`) na zewnątrz, bursztyn (`rimColorB`) do środka,
- `SOFT_CORE`: `exp(−v²·k)` w kolorze bursztynu, gain `haloIntensity`.

## Ruch

Wszystko jest funkcją `phase = 2π · (t / loopSeconds)`, a domyślnie `loopSeconds = 5`.

| Ruch | Wzór | Parametr |
| --- | --- | --- |
| precesja siodła | `φ = precessCycles · phase` | `precessCycles` (int, dom. 1) |
| obrót płatów kolorów | `lobeAngle = −lobeCycles · phase` | `lobeCycles` (int, dom. 1) |
| kołysanie pętli wokół X | `tilt + wobble · sin(wobbleCycles · phase)` | `wobbleCycles` (int, dom. 1) |

Wartości liczbowe oznaczone w tym dokumencie jako „dom.” lub „~” to wartości startowe. Ostateczne
domyślne wartości ustalam przy strojeniu (sekcja Weryfikacja).

Suwaki „cykli” są całkowite (krok 1, zakres −3…3), więc pętla jest bezszwowa przy każdym
ustawieniu. Kierunki i fazy początkowe dobieram przy strojeniu do klatki 0 wideo.

## Kamera i kadr

`PerspectiveCamera(32°)`. Odległość liczona co klatkę tak, żeby średnica kuli z poświatą
zajmowała `orbSize` (dom. 0,44) **krótszego** boku viewportu. Na telefonie w pionie orb też
się mieści. Orb jest wyśrodkowany.

## Post-processing

`RenderPass → UnrealBloomPass → blur H → blur V → OutputPass (ACES, sRGB) → finish`

- Render targety `HalfFloatType`, bez depth buffera, jak w oryginale.
- **Blur względny**: promień podawany jako ułamek wysokości drawing buffera (dom. ~0,02) i
  przeliczany na piksele przy resize. Gdy promień w pikselach przekracza 24, blur H+V
  przechodzi dwa razy, żeby 13 próbek nie dawało pasków.
- Finish: aberracja chromatyczna i ziarno (dom. 0), tło `#000`, winieta wyłączona (dom. 0).
- Clear color czarny, dodatki w przestrzeni wyświetlania, jak w oryginale.

## Panel (lil-gui)

- **Shape**: sphere radius, loop radius, saddle height, band width, twist, tilt, wobble,
  loop length (s), precession cycles, wobble cycles, orb size.
- **Sphere**: fresnel power, shell brightness, interior brightness, top brightness, cool
  colour, warm colour, lobe softness, lobe tilt, lobe cycles.
- **Ribbon**: body colour, body brightness, fold sharpness, fold brightness, rim width, rim
  brightness, rim colours (outer/inner), halo brightness, halo width.
- **Post-processing**: bloom strength/radius/threshold, blur, exposure, grain, chromatic
  aberration, vignette.
- **Pause**.

Na ekranach < 700 px panel startuje zwinięty.

Nazwy w panelu są po angielsku, spójnie z oryginałem.

## Uchwyt debugowy

```js
window.__orb = { params, setTime(t), renderFrame(), renderer, composer }
```

`setTime(t)` ustawia czas symulacji na dokładne `t` (sekundy), a `renderFrame()` synchronicznie
wykonuje jeden `composer.render()` z aktualnymi parametrami, niezależnie od pauzy. Canvas
tworzony jest z `preserveDrawingBuffer: true` tylko wtedy, gdy URL ma `?capture`, żeby
przechwytywanie działało bez kosztu w normalnym trybie. `?capture` dodatkowo ukrywa panel
i zatrzymuje pętlę `requestAnimationFrame` (klatki powstają wyłącznie przez `renderFrame()`).

## Weryfikacja

1. Lokalny serwer w scratchpadzie (Python): serwuje repo i przyjmuje `POST /save?name=…` z PNG.
2. W przeglądarce aplikacji viewport **1440×1080**, strona `index.html?capture`, panel ukryty.
3. Dla `t ∈ {0, 1, 2, 3, 4, 5}`: `setTime → renderFrame → canvas.toBlob → POST`.
4. Skrypt numpy porównuje render z klatkami wideo w tych samych chwilach:

| Metryka | Kryterium |
| --- | --- |
| średnica orbu (bbox pikseli > 12/255) | 480 px ± 5% |
| średni kolor pikseli „niebieskich” (B > R + 30, suma > 150) | każdy kanał ±15/255 od `#4c5e8f` |
| średni kolor pikseli „bursztynowych” (R > B + 60, suma > 200) | każdy kanał ±15/255 od `#b27239` |
| pętla: render `t=0` vs `t=5` | średnia różnica < 0,5/255 |
| rogi kadru | `#000000` |

5. Arkusz porównawczy 6 par (wideo | render) do oceny wzrokowej przez użytkownika. To
   ostateczne kryterium.
6. Strojenie domyślnych parametrów: najwyżej 3–4 rundy, potem przegląd z użytkownikiem.
7. Dodatkowo: brak błędów w konsoli, płynność na desktopie, a viewport 375×812 mieści orb i ma
   zwinięty panel.

## Git i publikacja

- Lokalne repo `~/Desktop/light-orb-shader`, gałąź `main`, commity na kolejnych etapach.
- Utworzenie repo na GitHubie, push i włączenie GitHub Pages **tylko po wyraźnej zgodzie
  użytkownika**, o którą pytam osobno na końcu.

## Rewizja 2: model dwóch czasz (2026-09-29)

Zastępuje sekcje „Scena”, „Geometria wstęgi”, „Materiał powłoki” i „Ruch” powyżej. Pipeline,
post-processing, kamera, panel (poza nazwami grup), uchwyt debugowy i weryfikacja zostają bez
zmian.

**Obserwacja (użytkownik, potwierdzona na klatkach z podbitą gammą):** orb to dwa
półprzezroczyste elementy, każdy w dwóch kolorach, które lekko na siebie zachodzą i razem tworzą
kulę „przeciętą na pół”. To, co wyglądało jak osobna wstęga, to świecący brzeg górnej czaszy
widziany pod ostrym kątem. Ciemna „dziura” u góry to środek czaszy zwrócony do kamery.

Dlaczego poprzedni model nie mógł tego osiągnąć (z raportu strojenia): symetryczna pętla nie
przechyla się na boki, kolor kuli nie był powiązany ze wstęgą, a szerokość wstęgi była stała.

### Linia cięcia

Dla punktu na jednostkowej sferze o azymucie `θ = atan(z, x)`:

```
cutY(θ) = cutTilt · sin(θ − ψ) + saddleHeight · sin(2θ + φ) + cutOffset
```

Pierwsza harmoniczna przechyla płaszczyznę podziału (ukos lewa-wyżej / prawa-niżej), a druga
tworzy siodło. `ψ = cutPhase + cutCycles · phase` obraca kierunek przechyłu, a
`φ = precessPhase + precessCycles · phase` przesuwa falę siodła. Oba cykle są całkowite, więc
pętla zostaje bez szwu. Ten sam kod GLSL (`cutGlsl`) trafia do shadera czasz i do shadera wargi,
więc obie warstwy zawsze się zgadzają.

### Warstwy (addytywne, bez depth testu, DoubleSide)

1. **Górna czasza** i **dolna czasza**: ta sama `SphereGeometry(1, 128, 96)`, shader z
   `uSide = +1 / −1`.
   - Maska: `smoothstep(−edgeSoftness, edgeSoftness, uSide · (n.y − cutY(θ)) + overlap)`, czyli
     każda czasza sięga `overlap` za linię cięcia.
   - Jasność: Fresnel jak dotąd (przód `shellIntensity`, tył `interiorIntensity` z połową
     wykładnika), pomnożony przez `smoothstep(0, limbFade, facing)`, co wygasza sam brzeg sylwetki.
   - Kolor: `mix(cool, warm, smoothstep(−s, s, uSide · sin(θ − ψ + colorPhase)))`. Dwa kolory po
     obwodzie, obracające się razem z cięciem. Dolna czasza ma je w odwrotnym układzie.
   - Poświata krawędzi: `warm · exp(−((n.y − cutY)/lipWidth)²) · lipGlow`.
2. **Warga**: dotychczasowa wstęga (ten sam fragment shader: korpus, fold, rim, `backDim`)
   prowadzona dokładnie po linii cięcia na powierzchni kuli. Pas biegnie wzdłuż południka,
   `curl` wychyla go do wnętrza kuli. Halo (`SOFT_CORE`) zostaje.

Grupa `orbGroup` jest pochylona o `viewTilt` (lekki widok z góry). Osobna `ribbonGroup` znika.

### Parametry

- **Shape**: sphere radius, view tilt, cut tilt, saddle height, cut offset, overlap, edge
  softness, loop length, cut cycles/phase, precession cycles/phase, orb size.
- **Shells**: fresnel power, shell/interior brightness, limb fade, cool/warm colour, colour
  softness, colour phase, lip glow, lip width.
- **Lip**: band width, curl, body colour/brightness, fold sharpness/brightness, rim width,
  rim brightness, rim colours, back side brightness, halo brightness/width.

Znikają: `loopRadius`, `twist`, `tilt`, `wobble`, `wobbleCycles`, `topBrightness`,
`lobeSoftness`, `lobeTilt`, `lobeCycles`, `lobePhase`.
