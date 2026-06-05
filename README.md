# ChromaPlex v2.0: Specification & Architecture Documentation
### *Massivt Parallel Spatial & Angulær Optisk Computerarkitektur og Instruktionssæt*

Velkommen til det officielle ChromaPlex-repository. Dette dokument indeholder den fulde tekniske specifikation af **ChromaPlex-arkitekturen** og dens tilhørende **Spatial Tensor Assembly-sprog (v2.0)**. 

ChromaPlex er designet til at bryde med de sekventielle flaskehalse i traditionel silicium-baseret elektronik (såsom NVMe SSD'er) ved at udnytte lysets fundamentale egenskaber til ultrahurtig, parallel 3D-datahåndtering i glaskrystaller (*Fused Silica*).

---

## 1. Introduktion & Paradigmeskift

Traditionelle lagringsmedier flytter elektroner sekventielt gennem transistorer i silicium ved hastigheder omkring ~10^5 m/s, begrænset af varmeudvikling, støj og elektronmobilitet. Dette sætter en effektiv ingeniørmæssig barriere for moderne NVMe SSD'er på omkring 7–14 GB/s.

**ChromaPlex redefinerer dette fundamentalt ved at udnytte:**
1. **Lyshastighed i glas:** Fotoner bevæger sig med ~2 × 10^8 m/s (3000 gange hurtigere end elektroner).
2. **Spatiell Parallelisme (2D-planer):** Hele 2D-lag (millioner af voxels) belyses og læses simultant af en højhastighedsdetektor i stedet for bit-for-bit aflæsning.
3. **Angulær og Spektrel Multipleksing:** Ved at ændre indfaldsvinklen på laserstrålerne via en Spatial Light Modulator (SLM) og udnytte polarisering samt farvebølgelængder (RGB), kan ét enkelt fysisk punkt (en voxel) indeholde flere uafhængige datakanaler samtidig uden krydstale.

---

## 2. Det Fysiske Hardware-lag

For at kunne eksekvere ChromaPlex-instruktioner kræves et hardware-setup bestående af fem kernekomponenter integreret via en FPGA-baseret kontrolenhed:

| Komponent | Funktion | Typisk Specifikation (Kommerciel baseline) |
| :--- | :--- | :--- |
| **Lagringsmedie** | Fused Silica (Kvartsglas) krystal | 1 cm³ krystal, 500 nm voxel-afstand, 9.0 eV båndgap |
| **Skrivelaser** | Femtosekund-laser | 10W effekt, 1 MHz repetitionsrate, skadetærskel ~2 J/cm² |
| **Stråle-splitter** | Spatial Light Modulator (SLM) | 4K+ opløsning flydende krystalmatrix, 60–180 Hz opdatering |
| **Læsedetektor** | Højhastigheds CMOS-sensor | 20–100 Megapixel, 1.000–10.000 fps med farve-/polarisationsfilter |
| **Positionering** | Piezo-elektrisk stage | Sub-nanometer mekanisk præcision med termisk stabilisering |

### Voxel-Datastrukturen (96 bits pr. Voxel)
Hvert optisk punkt i krystallen kaldes en **voxel**. ChromaPlex opnår ekstrem datatæthed ved at multiplekse 96 bits ind i en enkelt voxel via følgende dimensioner:
* **3 Farvekanaler (RGB)** × **2 Polarisationsretninger (0° og 90°)** = 6 uafhængige kanaler.
* Hver kanal koder både **Intensitet (8 bit)** og **Faseforskydning (8 bit)** via interferometri = 16 bit pr. kanal.
* **Total kapacitet pr. voxel:** 6 × 16 bits = 96 bits.

---

## 3. Registerarkitektur (Hardwareniveau)

ChromaPlex-kontrolleren (FPGA) opererer med tre specialiserede registertyper, der håndterer de multidimensionelle datatensorer:

1. **`SR` (Spatial Registers - 64-bit):**
   Indeholder geometriske og rumlige koordinatsæt. Bruges til at adressere specifikke X, Y, Z koordinater, plan-afgrænsninger samt vinkel-vektorer (Theta, Phi) for den angulære multipleksing.
   
2. **`VR` (Voxel Registers - Ultra-brede):**
   Ultrabrede interne hardware-buffere (op til 2 gigabit brede). Disse registre føder data direkte til SLM-skriveinterfacerne og modtager rå streams fra CMOS-læsedetektoren i realtid uden CPU-overhead.

3. **`OPU` (Optical Processing Unit):**
   Den dedikerede hardware-coprocessor på FPGA'en, der varetager lynhurtig asynkron de-multipleksing, fase-afkodning og hardware-baseret fejlkorrektion (ECC).

---

## 4. ChromaPlex ISA v2.0 (Instruktionssæt)

ChromaPlex-samlesproget erstatter skalære operationer med spatiale tensor-primitiver.

### `LOAD.PLANE`
* **Syntaks:** `LOAD.PLANE VR_dest, SR_coord, MASK`
* **Beskrivelse:** Trigger CMOS-detektoren til asynkront at indlæse et komplet 2D-dybdelag fra krystallen. `SR_coord` definerer Z-planet samt X/Y grænserne. `MASK` kan filtrere specifikke polarisationsvinkler eller farvekanaler.
* **Gennemstrømning:** Ved 20 MP sensor og 1000 fps indlæses 240 MB pr. operation, hvilket giver en teoretisk læsehastighed på **240 GB/s**.

### `DEMUX.VOXELS`
* **Syntaks:** `DEMUX.VOXELS VR_dest_bits, VR_source_raw`
* **Beskrivelse:** Finder sted i OPU'en. Behandler de rå optiske interferensmønstre og intensitetsmatrixer fra `VR_source_raw`, kører hardware-fejlkorrektion (ECC), og omdanner dem til ukomprimerede, binære data (96 bits pr. voxel) i `VR_dest_bits`.

### `PREP.PHASE`
* **Syntaks:** `PREP.PHASE VR_slm_dest, DATA_PTR`
* **Beskrivelse:** Tager en binær datablok fra systemhukommelsen (`DATA_PTR`) og beregner det holografiske fasediagram og interferensmønster, som SLM'en skal vise for at splitte laserstrålen optimalt.

### `STORE.ANGLES`
* **Syntaks:** `STORE.ANGLES SR_coord, VR_slm_source`
* **Beskrivelse:** Det primære parallelle skriveprimitiv. `VR_slm_source` sender det beregnede fasediagram til SLM'ens flydende krystaller. Når de har stabiliseret sig, affyres én enkelt femtosekund-puls fra laseren. Lyset brydes i op til 1.000.000 delstråler fra unikke vinkler (Theta, Phi) via SLM'en, og skriver op til 1.000.000 voxels simultant i krystallens brydningsindeks.
* **Gennemstrømning:** Ved en 1 MHz laser og 1 million parallelle stråler skrives 10^12 voxels/s, hvilket svarer til en skrivehastighed på **12 TB/s**.

### `SYNC.OPTICS`
* **Syntaks:** `SYNC.OPTICS`
* **Beskrivelse:** En hardware-barriere. Pauser pipelinen indtil SLM'ens krystaller er fuldt omarrangerede (60–180 Hz fysisk grænse), eller indtil CMOS-sensorens readout-buffer er tømt. Dette forhindrer korrupt data og termisk overbelastning af krystallen fra de 10W lasereffekt.

---

## 5. Eksempelprogram: Massiv Parallel Skrivning & Læsning

Følgende program demonstrerer, hvordan ChromaPlex-assembler bruges til at initialisere et skrivemønster på 1 million voxels via vinkelmultipleksing, efterfulgt af en lynhurtig planlæsning.

```assembly
; =====================================================================
; CHROMAPLEX ASSEMBLY V2.0 SPARK EXEMPEL
; Formål: Parallel skrivning af 1M voxels og efterfølgende kontrol-læsning
; =====================================================================

.data
    DATA_CHUNK_ADDR   equ 0x00FF0000   ; Startadresse for rå data (96 Megabits)

.code
_start:
    ; --- DEL 1: INITIALISERING AF REGISTRE ---
    SET.SR    SR1, Z=42, X=(0,999), Y=(0,999)    ; Definer målområde i krystal (Z-plan 42)
    SET.SR    SR2, THETA_OFFSET=0, PHI_OFFSET=0  ; Initialiser vinkelmatrix-offset

    ; --- DEL 2: HOLOGRAFISK FORBEREDELSE & PARALLEL SKRIVNING ---
    PREP.PHASE  VR2, DATA_CHUNK_ADDR            ; Beregn 2D fasediagram for de 96 Mb data
    
    SYNC.OPTICS                                 ; Vent på at hardwaren er klar og stabil
    STORE.ANGLES SR1, VR2                        ; Affyr laserpuls: Skriv 1.000.000 voxels simultant!

    ; --- DEL 3: ASYNKRON LÆSNING & DE-MULTIPLEKSING ---
    SYNC.OPTICS                                 ; Sikrer at krystallen har fundet termisk ro
    LOAD.PLANE   VR0, SR1, MASK_ALL             ; Aktiver CMOS: Læs hele Z-plan 42 med det samme (240 MB)
    
    ; OPU de-multiplekser data i baggrunden, mens CPU/FPGA kan håndtere næste instruktion
    DEMUX.VOXELS VR1, VR0                       ; Kør ECC og split kanalerne (RGB + Polarisation + Fase)
    
    ; VR1 indeholder nu de perfekt rekonstruerede 96 Megabits i binær form
    HALT
```

---

## 6. Bidrag og Videre Udvikling (Freedom to Operate)

ChromaPlex er et open-source deep tech initiativ, der bygger broen mellem anvendt fotonik og computerarkitektur. Hvis du ønsker at arbejde videre med projektet:

1. **Systemintegration:** Bidrag til FPGA-pipelinen for `DEMUX.VOXELS`-instruktionen, som i øjeblikket er flaskehalsen for realtidshåndtering af 100+ Gbps streams.
2. **Kompiler-udvikling:** Vi mangler algoritmer til automatisk at pakke vilkårlige arrays og datastrukturer optimalt ind i de 2D-fasediagrammer, som SLM'en kræver.
3. **Patent-landskab:** Før kommerciel udnyttelse skal der foretages en grundig *Freedom to Operate*-undersøgelse, da aktører som Microsoft (Project Silica), Hitachi og University of Southampton ejer grundlæggende patenter inden for 5D/optisk datalagring.

---
*ChromaPlex er udviklet af fritænkere til fremtidens computere. Lad os gøre lyset eksekverbart.*
