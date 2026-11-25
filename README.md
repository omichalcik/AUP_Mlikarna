# AUP_Mlikarna

Projekt do předmětu MPC-AUP (Automatizace procesů), VUT FEKT – **pasterizace mléka** (pasterizační jednotka: nádrž 2 m³, napouštěcí a vypouštěcí ventil, míchadlo, ohřev 15 MW, max. teplota 95 °C).
PLC program v TwinCAT 3 (Beckhoff), strukturovaný podle ISA-88 (moduly zařízení → fáze), komunikace se simulátorem přes Modbus TCP.

Autoři: Ondřej Michalčík, David Pazour

## Spuštění

1. Potřebné doplňky TwinCATu: **TF6250 Modbus TCP** (Engineering + Runtime) a **TF4100 Controller Toolbox** (knihovna `Tc2_ControllerToolbox`, PID). Licence: 7denní trial v *System → License*.
2. Spustit simulátor `maup.NET.exe` (Modbus TCP, `127.0.0.1:504`).
3. Activate Configuration (`<Local>`) → Login → Start.
4. Fáze ovládají zařízení jen v automatu – zařízení (ventily, Mixer, Heater) přepnout do **Aut**.

## Struktura programu

| Část            | Bloky                                                                                              |
| --------------- | -------------------------------------------------------------------------------------------------- |
| Tasky           | `PlcTask` 100 ms: `MB_Connector`, `Main_Batch`; `PlcTask500ms`: `MAIN` (sekundový impulz `secImp`) |
| Tank            | `Main_Batch` (LD) → `FB_Tank` (LD) s datovým blokem `Global.Tank1 : TTank`                         |
| Moduly zařízení | `Valve_2DO` (2×), `Engine_1DO` (mixér), `Heater` (PID z Tc2_ControllerToolbox)                     |
| Fáze            | `Filling`, `Draining`, `Heating`, `Mixing`                                                         |
| Ostatní         | `MB_Connector` (Modbus TCP), `RawToEng` / `EngToRaw` (převody analogových hodnot)                  |

DUT fází (`TPhase*`) se **nesmí měnit** (kompatibilita s MES SkuBatch). DUT zařízení měnit lze.

## TODO

Hodnocení projektu (22 b, min. 11 b) je uvedené u jednotlivých částí.

### Návrh a dokumentace technologie

- [x] Popis technologie (1 b) – `Michalcik_AUP.pdf`
- [x] Procesní schéma / P&ID (2 b) – `schemata/qeltech/PI&D_v03.qet`, `PI&D_v01-*.pdf`
- [x] Instrumentace (2 b) – tabulka v `Michalcik_AUP.pdf`, `tabulka_instrumentace.tex`
- [x] Elektrotechnické schéma (4 b) – `schemata/qeltech/elektroschema_v13.qet`
- [x] UML diagramy (2 b) – use-case (+ specifikace), sekvenční, stavový (fáze napouštění), diagram tříd – *ověřit stav, ve složce projektu nenalezeno*

### Program (5 b)

- [x] `MB_Connector` – Modbus TCP se simulátorem

- [x] `RawToEng` / `EngToRaw` – převody hladiny, teploty a výkonu ohřevu

- [x] `Engine_1DO` – mixér (Aut/Man, motohodiny, nerozběh, chyby, StateInt)

- [x] `Valve_2DO` – napouštěcí a vypouštěcí ventil (Aut/Man, nedoběh, chyby, StColor)

- [x] `Heater` – ohřev s PID regulátorem, ochrana proti přehřátí

- [x] `FB_Tank`, `Main_Batch` – zapojení modulů a fází, `GlobalError`

- [x] Fáze `Filling` – napouštění

- [x] Fáze `Draining` – vypouštění

- [x] Fáze `Heating` – ohřev, Done po 10 s v pásmu ±1 °C (otestováno)

- [x] Fáze `Mixing` – míchání po zadanou dobu, Pause zachová uběhlý čas (otestováno)

- [x] `Filling` / `Draining` vypínaly moc brzo – offsety v podmínce Done zmenšeny z `- 0.31` / `+ 0.17` na `± 0.05` m (2026-11-25)

- [ ] `Valve_2DO`: `State.Err := iError` – teď se chyba jen nastaví a nikdy nevynuluje *(David)*

- [ ] `Valve_2DO`: samostatný `Time2Close` pro `CloseFailTimer` (přidat do `TValve`) *(David)*

- [x] `Heater`: vypínací mez přehřátí pod 95 °C (95 °C = zničení technologie), např. vypnout 90 °C / znovu povolit pod 85 °C

- ~~Batch – stavový automat procedury v PLC (kap. 10.4)~~ – nebude se dělat (volitelné)

### Vizualizace (2 b)

- [x] Obrazovky `Prehled`, `Zarizeni`, `Faze`, `Diagnostika`
- [x] Ovládání a zobrazení fází Heating a Mixing

### Optimalizace regulačního děje (2 b)

- [x] `Heater`: PID se při vypnutém ohřevu resetuje (`pidMode` = `eCTRL_MODE_RESET` při `EnableOut = FALSE`), integrační složka se nenatahuje na 100 %
- [x] Naměřená přechodová charakteristika (Scope projekt `TwinCAT_Scope/`, data `TwinCAT_Scope/sys_data.csv`): Heater v Man, skok výkonu 0 → 20 % v čase 29,4 s, teplota 25,3 → ustáleno ~54,6 °C, záznam 600 s, Ts = 0,1 s. Sloupce CSV: čas; `HeatingOut` [%]; čas; `WaterTemp` [°C]. Orientačně K ≈ 1,5 °C/%, časová konstanta ~90 s
- [x] Identifikace soustavy v MATLABu (System Identification / `tfest`) – přenos + „fit“ do dokumentace
- [x] Návrh PI regulátoru (Simulink PID Tuner, tvar Ideal, saturace 0–100 %, anti-windup Clamping)
- [x] Konstanty do `Heater` (`fKp`, `tTn`, `tTv` = 0 pro PI) – teď jsou ze šablony (Kp = 10, Tn = 2 s, Tv = 0,5 s) a teplota kmitá
- [x] Ověření v PLC – skok žádané teploty ve fázi Heating, záznam ve Scope, porovnání se simulací
- [ ] (volitelně) parametry PID do `THeater` (`PIDParams : ST_CTRL_PID_PARAMS`) → ladění online bez kompilace

### MES – SkuBatch (rozhraní je požadovaný výstup programu)

- [ ] `Global.TcGVL`: u `Tank1` atributy `OPC.UA.DA` a `OPC.UA.DA.StructuredType`, TMC, OPC UA server (TF6100)
- [ ] Ověřit řízení fází v Aut ze SkuBatch (receptura: napustit → míchat + ohřívat → vypustit)

### Vlastní inovace (2 b)

- [ ] Např. blokace: netopit / nemíchat při prázdné nádrži, neplnit a nevypouštět zároveň, reakce fází na `GlobalError`

### Dokumentace a odevzdání

- [ ] Popis PLC programu – jednotlivé bloky + celková logika
- [ ] Popis postupu návrhu regulátoru
- [ ] Manuál k vizualizaci pro operátora
- [ ] Zip s kompletním programem
- [ ] Prezentace výsledků (13. cvičení)

### Úklid

- [ ] Smazat nepoužívané `Engineering/` (`Engine`, `MixingPhase`, `Tank`, `TankLD` – nejsou v projektu) a `Test.TcPOU`
- [ ] Sjednotit verzi TwinCATu (Ondra 3.1.4026, projekt byl v 4024) – TwinCAT 4026 přepisuje `MPCAUP_Tanky.tsproj`, `PLC_Tank.plcproj` a verzi v hlavičce každého uloženého souboru (`ProductVersion`)
- [ ] (volitelně) offsety v Done `Filling` / `Draining` jako pojmenované konstanty / parametry
