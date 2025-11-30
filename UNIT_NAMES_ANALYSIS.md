# Unit Internal Names & Codenames Analysis
## Command & Conquer Remastered Collection

---

## Table of Contents
1. [Overview](#overview)
2. [Tiberian Dawn Units](#tiberian-dawn-units)
3. [Red Alert Units](#red-alert-units)
4. [Naming Conventions](#naming-conventions)
5. [Notable Codenames](#notable-codenames)
6. [Development Insights](#development-insights)

---

## Overview

The Command & Conquer source code reveals fascinating internal naming conventions for units. While players saw polished names like "Mammoth Tank" or "Commando," developers used terse enum identifiers and quirky codenames. This document catalogs all unit internal names from both Tiberian Dawn and Red Alert, revealing the personality and humor embedded in the codebase.

**Key Findings:**
- Many units have pop culture references (INFANTRY_RAMBO, UNIT_VICE for "Visceroid")
- Some retain development codenames never seen by players (UNIT_STANK for "Stealth tank (Romulan)")
- Naming reflects 1990s development culture and sci-fi influences
- Abbreviations prioritize brevity for programming convenience

---

## Tiberian Dawn Units

### Infantry Types (`InfantryType` enum)

| Internal Name | In-Game Name | Description | INI Name | Notes |
|--------------|--------------|-------------|----------|-------|
| `INFANTRY_E1` | Minigunner | Mini-gun armed soldier | "E1" | Generic soldier designation |
| `INFANTRY_E2` | Grenadier | Grenade thrower | "E2" | GDI infantry |
| `INFANTRY_E3` | Rocket Soldier | Rocket launcher | "E3" | Anti-armor infantry |
| `INFANTRY_E4` | Flamethrower | Flame thrower equipped | "E4" | Nod specialty unit |
| `INFANTRY_E5` | Chem Warrior | Chemical spray thrower | "E5" | Nod chemical weapons |
| `INFANTRY_E7` | Engineer | Building capture specialist | "E6" | **Note: E7 enum, E6 INI!** |
| `INFANTRY_RAMBO` | Commando | Elite commando unit | "RAMBO" | **Pop culture reference!** |
| `INFANTRY_C1` - `INFANTRY_C9` | Civilians | Various civilian types | "C1" - "C9" | Numbered civilians |
| `INFANTRY_C10` | Nikoomba | Special civilian | "C10" | Named character |
| `INFANTRY_MOEBIUS` | Dr. Moebius | Story character | "MOBL" | Key campaign figure |
| `INFANTRY_DELPHI` | Agent Delphi | Covert operative | "DELPHI" | Mission-specific |
| `INFANTRY_CHAN` | Dr. Chan | Scientist character | "CHAN" | Story NPC |

**Analysis:**
- "E" series = Enemy/Entity soldiers (E1-E7)
- "C" series = Civilians (C1-C10)
- Named characters use actual names (Moebius, Delphi, Chan)
- **RAMBO** is a direct Sylvester Stallone reference for the commando

### Vehicle/Unit Types (`UnitType` enum)

| Internal Name | In-Game Name | Description | INI Name | Commentary |
|--------------|--------------|-------------|----------|------------|
| `UNIT_HTANK` | Mammoth Tank | Heavy tank (Mammoth) | "HTNK" | "H" = Heavy |
| `UNIT_MTANK` | Medium Tank | Medium tank (M1) | "MTNK" | GDI main battle tank |
| `UNIT_LTANK` | Light Tank | Light tank ('Bradly') | "LTNK" | **Typo: Bradley** |
| `UNIT_STANK` | Stealth Tank | Stealth tank (**Romulan**) | "STNK" | **Star Trek reference!** |
| `UNIT_FTANK` | Flame Tank | Flame thrower tank | "FTNK" | Nod flame vehicle |
| `UNIT_VICE` | Visceroid | Mutant creature | "VICE" | Tiberium mutation |
| `UNIT_APC` | APC | Armored personnel carrier | "APC" | Standard abbreviation |
| `UNIT_MLRS` | MLRS | MLRS rocket launcher | "MLRS" | Real-world weapon system |
| `UNIT_JEEP` | Hummer | 4x4 jeep replacement | "JEEP" | Light recon vehicle |
| `UNIT_BUGGY` | Nod Buggy | Rat patrol dune buggy | "BGGY" | Fast attack vehicle |
| `UNIT_HARVESTER` | Harvester | Resource gathering vehicle | "HARV" | Economic unit |
| `UNIT_ARTY` | Artillery | Artillery unit | "ARTY" | Long-range support |
| `UNIT_MSAM` | Mobile SAM | Anti-aircraft vehicle | "MLRS" | **Same INI as MLRS!** |
| `UNIT_HOVER` | Hovercraft | Landing craft (LST) | "LST" | Landing Ship Tank |
| `UNIT_MHQ` | Mobile HQ | Mobile headquarters | "MHQ" | Radar vehicle |
| `UNIT_GUNBOAT` | Gunboat | Naval artillery | "BOAT" | Water-based unit |
| `UNIT_MCV` | MCV | Mobile construction vehicle | "MCV" | Base deployment |
| `UNIT_BIKE` | Recon Bike | Nod recon motor-bike | "BIKE" | Fast scout |
| `UNIT_TRIC` | Triceratops | Dinosaur unit | "TRIC" | Hidden/cheat unit |
| `UNIT_TREX` | T-Rex | Tyrannosaurus Rex | "TREX" | Hidden/cheat unit |
| `UNIT_RAPT` | Velociraptor | Raptor | "RAPT" | Hidden/cheat unit |
| `UNIT_STEG` | Stegosaurus | Stegosaurus | "STEG" | Hidden/cheat unit |

**Key Observations:**
- Tank naming: H=Heavy, M=Medium, L=Light, S=Stealth, F=Flame
- **"Romulan"** comment refers to Star Trek's cloaking technology
- **"Bradly"** is a misspelling of "Bradley" (M2 Bradley Fighting Vehicle)
- Dinosaur units exist for fun/testing (TRIC, TREX, RAPT, STEG)
- **VICE** = Visceroid, the Tiberium-mutated blob creature

### Aircraft Types (`AircraftType` enum)

| Internal Name | In-Game Name | Description | INI Name | Details |
|--------------|--------------|-------------|----------|---------|
| `AIRCRAFT_TRANSPORT` | Chinook | Transport helicopter | "TRAN" | Troop carrier |
| `AIRCRAFT_A10` | A-10 | Ground attack plane | "A10" | Real aircraft designation |
| `AIRCRAFT_HELICOPTER` | Apache | Attack helicopter | "HELI" | Nod gunship |
| `AIRCRAFT_CARGO` | C-17 | Cargo plane | "C17" | Reinforcement transport |
| `AIRCRAFT_ORCA` | Orca | Nod attack helicopter | "ORCA" | Unique fictional design |

**Notes:**
- Mix of real aircraft (A-10, Apache) and fictional (Orca)
- "ORCA" is a wholly original design name
- A-10 "Warthog" used for close air support
- C-17 used for paradrop reinforcements

---

## Red Alert Units

### Infantry Types (`InfantryType` enum)

| Internal Name | In-Game Name | Description | INI Name | Commentary |
|--------------|--------------|-------------|----------|------------|
| `INFANTRY_E1` | Rifle Infantry | Mini-gun armed | "E1" | Basic Soviet/Allied |
| `INFANTRY_E2` | Grenadier | Grenade thrower | "E2" | Anti-infantry |
| `INFANTRY_E3` | Rocket Soldier | Rocket launcher | "E3" | Anti-armor/air |
| `INFANTRY_E4` | Flamethrower | Flame thrower | "E4" | Soviet specialty |
| `INFANTRY_RENOVATOR` | Engineer | Building capture | "E6" | **"Renovator" codename!** |
| `INFANTRY_TANYA` | Tanya | Elite commando | "E7" | **Named after dev's daughter** |
| `INFANTRY_SPY` | Spy | Infiltration unit | "SPY" | Allied special ops |
| `INFANTRY_THIEF` | Thief | Steals money | "THF" | Soviet stealth unit |
| `INFANTRY_MEDIC` | Medic | Heals infantry | "MEDI" | Allied support |
| `INFANTRY_GENERAL` | Field Marshal | Mission character | "GNRL" | Campaign-specific |
| `INFANTRY_DOG` | Attack Dog | Soviet guard dog | "DOG" | Anti-spy unit |
| `INFANTRY_C1` - `INFANTRY_C10` | Civilians | Various civilians | "C1" - "C10" | Non-combatants |
| `INFANTRY_EINSTEIN` | Einstein | Scientist | "EINSTEIN" | Historical figure! |
| `INFANTRY_DELPHI` | Agent Delphi | Covert agent | "DELPHI" | Returning character |
| `INFANTRY_CHAN` | Dr. Chan | Scientist | "CHAN" | From Tiberian Dawn |
| `INFANTRY_SHOCK` | Shock Trooper | Tesla infantry | "SHOK" | Soviet advanced unit |
| `INFANTRY_MECHANIC` | Mechanic | Vehicle repair | "MECH" | Support unit |

**Fascinating Details:**
- **INFANTRY_RENOVATOR** = "Renovator" was the internal codename for Engineer!
- **INFANTRY_TANYA** = Named after Tanya Adams, allegedly inspired by a developer's daughter
- **INFANTRY_EINSTEIN** = Albert Einstein appears as a character!
- **INFANTRY_DOG** = First "animal" unit in C&C
- **INFANTRY_SHOCK** = Shock Trooper uses Tesla technology

### Vehicle/Unit Types (`UnitType` enum)

| Internal Name | In-Game Name | Description | INI Name | Analysis |
|--------------|--------------|-------------|----------|----------|
| `UNIT_HTANK` | Mammoth Tank | Heavy tank | "4TNK" | Returning from TD |
| `UNIT_MTANK` | Heavy Tank | Soviet heavy tank | "3TNK" | Soviet main tank |
| `UNIT_MTANK2` | Medium Tank | Allied medium tank | "2TNK" | Allied main tank |
| `UNIT_LTANK` | Light Tank | Light tank | "1TNK" | Fast, weak armor |
| `UNIT_APC` | APC | Armored personnel carrier | "APC" | Troop transport |
| `UNIT_MINELAYER` | Minelayer | Mine-laying vehicle | "MNLY" | Defensive unit |
| `UNIT_JEEP` | Ranger | 4x4 jeep | "JEEP" | Scout vehicle |
| `UNIT_HARVESTER` | Ore Truck | Resource gatherer | "HARV" | Economic unit |
| `UNIT_ARTY` | Artillery | Long-range artillery | "ARTY" | Soviet siege unit |
| `UNIT_MRJ` | Mobile Radar Jammer | Radar jammer | "MRJ" | Electronic warfare |
| `UNIT_MGG` | Mobile Gap Generator | Gap generator | "MGG" | Stealth field |
| `UNIT_MCV` | MCV | Mobile construction vehicle | "MCV" | Base deployment |
| `UNIT_V2_LAUNCHER` | V2 Rocket Launcher | V2 rocket launcher | "V2RL" | Soviet rocket artillery |
| `UNIT_TRUCK` | Supply Truck | Convoy truck | "TRUK" | Mission objective |
| `UNIT_ANT1` | Ant (Small) | Warrior ant | "ANT1" | Giant ant mutant |
| `UNIT_ANT2` | Ant (Medium) | Warrior ant | "ANT2" | Larger ant |
| `UNIT_ANT3` | Ant (Large) | Warrior ant | "ANT3" | Largest ant |
| `UNIT_CHRONOTANK` | Chrono Tank | Chrono-shifting tank | "CTNK" | **Time travel tank!** |
| `UNIT_TESLATANK` | Tesla Tank | Tesla-equipped tank | "TTNK" | Electricity weapon |
| `UNIT_MAD` | MAD Tank | M.A.D. Tank (Timequake) | "QTNK" | **Mutual Assured Destruction** |
| `UNIT_DEMOTRUCK` | Demo Truck | Demolition truck | "DTRK" | **"Jihad truck" comment!** |
| `UNIT_PHASE` | Phase Transport | Cloaking APC | "CTNK" | Mission-specific |

**Notable Findings:**
- Tank numbering: 1TNK (Light), 2TNK (Medium), 3TNK (Heavy), 4TNK (Mammoth)
- **UNIT_CHRONOTANK** = Time-traveling tank inspired by teleportation
- **UNIT_MAD** = "M.A.D." = Mutually Assured Destruction (Cold War reference)
- **UNIT_DEMOTRUCK** = Source comment calls it "Jihad truck" (suicide bomber)
- **UNIT_ANT1/2/3** = Giant ants from 1950s sci-fi movies (Them!)
- **V2_LAUNCHER** = References WWII German V-2 rocket

### Aircraft Types (`AircraftType` enum)

| Internal Name | In-Game Name | Description | INI Name | Commentary |
|--------------|--------------|-------------|----------|------------|
| `AIRCRAFT_TRANSPORT` | Chinook | Transport helicopter | "TRAN" | Paradrop unit |
| `AIRCRAFT_BADGER` | Badger | Soviet bomber | "BADR" | Real Soviet aircraft! |
| `AIRCRAFT_U2` | Spy Plane | Photo recon plane | "U2" | **U-2 Dragon Lady** |
| `AIRCRAFT_MIG` | MiG | Soviet fighter | "MIG" | Real Soviet jet |
| `AIRCRAFT_YAK` | Yak | Soviet attack plane | "YAK" | Yakovlev aircraft |
| `AIRCRAFT_LONGBOW` | Apache Longbow | Allied attack helicopter | "HELI" | AH-64 Apache |
| `AIRCRAFT_HIND` | Hind | Soviet attack helicopter | "HIND" | Mi-24 Hind |

**Historical References:**
- **BADGER** = Tupolev Tu-16 (NATO reporting name "Badger")
- **U2** = Lockheed U-2 spy plane (Francis Gary Powers incident)
- **MIG** = Mikoyan-Gurevich fighter jets
- **YAK** = Yakovlev design bureau
- **LONGBOW** = AH-64D Apache Longbow variant
- **HIND** = Mi-24 Hind assault helicopter

All aircraft use real Cold War-era designations!

### Vessel Types (`VesselType` enum)

| Internal Name | In-Game Name | Description | INI Name | Naval Classification |
|--------------|--------------|-------------|----------|---------------------|
| `VESSEL_SS` | Submarine | Attack submarine | "SS" | **SS = Submarine** |
| `VESSEL_DD` | Destroyer | Medium patrol craft | "DD" | **DD = Destroyer** |
| `VESSEL_CA` | Cruiser | Heavy patrol craft | "CA" | **CA = Heavy Cruiser** |
| `VESSEL_TRANSPORT` | Transport | Unit transporter | "LST" | Landing Ship Tank |
| `VESSEL_PT` | PT Boat | Light patrol craft | "PT" | **PT = Patrol Torpedo** |
| `VESSEL_MISSILESUB` | Missile Submarine | Missile-equipped sub | "MSUB" | SLBM carrier |
| `VESSEL_CARRIER` | Aircraft Carrier | Mobile airfield | "CARR" | Capital ship |

**Naval Code Analysis:**
- Uses **real US Navy hull classification symbols**!
- **SS** = Submarine (diesel-electric)
- **DD** = Destroyer (Fletcher-class style)
- **CA** = Heavy Cruiser (gun cruiser)
- **PT** = Patrol Torpedo boat (PT-109 reference)
- **LST** = Landing Ship, Tank (amphibious warfare)
- Shows developers' attention to military authenticity

---

## Naming Conventions

### Pattern Analysis

#### Alphabetic Prefixes
- **E-series** (E1-E7): Enemy/Entity - combat infantry
- **C-series** (C1-C10): Civilian - non-combatant humans
- **UNIT_xTANK**: Tank variants (H=Heavy, M=Medium, L=Light, S=Stealth, F=Flame)
- **INFANTRY_**: All human foot soldiers
- **AIRCRAFT_**: All flying units
- **VESSEL_**: All naval units (Red Alert only)

#### Abbreviation Strategy
- **3-4 character INI names**: HTNK, JEEP, HARV, MCV
- **Preserves readability**: ARTY (artillery), BIKE (recon bike)
- **Military standard codes**: APC, MLRS, V2RL
- **Sometimes identical to enum**: JEEP, BIKE, DOG

#### Pop Culture & Historical References
| Reference | Unit | Source |
|-----------|------|--------|
| Rambo | `INFANTRY_RAMBO` | Sylvester Stallone films |
| Romulan | `UNIT_STANK` | Star Trek cloaking |
| Einstein | `INFANTRY_EINSTEIN` | Historical figure |
| U-2 | `AIRCRAFT_U2` | Cold War spy plane |
| MiG, Yak, Hind | Aircraft | Soviet military |
| Badger | `AIRCRAFT_BADGER` | NATO reporting name |
| Them! | `UNIT_ANT1/2/3` | 1954 giant ant movie |
| Tanya | `INFANTRY_TANYA` | Team member's daughter (allegedly) |

---

## Notable Codenames

### Most Interesting Internal Names

#### 1. **INFANTRY_RENOVATOR** (Engineer)
Engineers capture and repair buildings, but internally they're called "Renovators"—a surprisingly wholesome name for a unit that steals enemy structures!

#### 2. **UNIT_STANK** → "Stealth tank (Romulan)"
The comment explicitly references Star Trek's Romulans, known for cloaking technology. Shows sci-fi influence on game design.

#### 3. **INFANTRY_RAMBO** (Commando)
Openly named after Rambo. The commando is a one-man army who infiltrates enemy bases with explosives—very on-brand for the character.

#### 4. **UNIT_DEMOTRUCK** → "Jihad truck"
The source code comment calls the Demolition Truck a "Jihad truck." This self-destructing unit is a suicide bomber vehicle. Reflects the darker Cold War themes.

#### 5. **UNIT_MAD** (M.A.D. Tank)
"Mutually Assured Destruction" - the ultimate Cold War doctrine. The MAD Tank creates a shockwave that destroys everything nearby, including itself.

#### 6. **UNIT_VICE** (Visceroid)
"Vice" as shorthand for "Visceroid"—the horrifying Tiberium-mutated blob creatures. Shows how developers abbreviated even grotesque concepts.

#### 7. **UNIT_LTANK** → "Light tank ('Bradly')"
Misspelling "Bradley" as "Bradly" suggests rapid development and less formal documentation. The M2 Bradley is a real US Army infantry fighting vehicle.

#### 8. **AIRCRAFT_BADGER**
Uses the NATO reporting name for the Soviet Tu-16 bomber. All Soviet aircraft use authentic designations (MiG, Yak, Hind, Badger), showing historical research.

#### 9. **INFANTRY_DOG**
The Attack Dog is the first non-human combatant unit in C&C. Simply called "DOG" internally—no fancy codename needed!

#### 10. **VESSEL_SS, DD, CA, PT**
Red Alert vessels use real US Navy hull classification symbols (SS=Submarine, DD=Destroyer, CA=Heavy Cruiser, PT=Patrol Torpedo). Demonstrates military authenticity.

---

## Development Insights

### What These Names Reveal

#### 1. **Pop Culture Immersion**
- **Rambo, Star Trek, 1950s sci-fi** influences are embedded directly in the code
- Developers were clearly fans of action movies and sci-fi television
- "Romulan" cloaking = Star Trek; Giant ants = "Them!" (1954)

#### 2. **Military Accuracy Mixed with Fantasy**
- Real designations: A-10, Apache, Bradley, MiG, U-2, PT boat
- Fictional elements: Mammoth Tank, Orca, Chronosphere technology
- Shows developers valued authenticity where appropriate

#### 3. **Cold War Themes**
- MAD (Mutually Assured Destruction)
- V-2 rockets (WWII/Cold War)
- Soviet vs. Allied naming conventions
- Nuclear weapons and advanced tech

#### 4. **Development Humor**
- "Jihad truck" for suicide bomber
- "Renovator" for building capturer
- Dinosaur units (TRIC, TREX, RAPT, STEG) for fun
- Tanya allegedly named after someone's daughter

#### 5. **Practical Programming**
- Short, memorable enum names (E1, E2, HTANK, LTANK)
- Consistent prefixes (INFANTRY_, UNIT_, AIRCRAFT_, VESSEL_)
- 3-4 character INI names for file I/O efficiency
- Type-safe enumerations prevent bugs

#### 6. **1990s Game Development Culture**
- Less corporate oversight = more personality
- Developer humor embedded in comments
- Pop culture references in production code
- Misspellings preserved in shipped product ("Bradly")

---

## Comparative Analysis

### Tiberian Dawn vs. Red Alert Naming

| Aspect | Tiberian Dawn | Red Alert |
|--------|--------------|-----------|
| **Tone** | Sci-fi futuristic | Historical Cold War |
| **Infantry** | E1-E7 designations | E1-E7 + named characters |
| **Vehicles** | Generic tank names | Numbered tanks (1TNK-4TNK) |
| **Aircraft** | Mix of real/fictional | All historically accurate |
| **Special Units** | Visceroid, dinosaurs | Giant ants, chrono tech |
| **References** | Rambo, Star Trek | Einstein, U-2, MAD doctrine |
| **Naval Units** | None (Gunboat only) | Full navy with hull codes |

**Key Difference:**
- **TD** = Science fiction with Tiberium mutation theme
- **RA** = Alternate history with real Cold War hardware

---

## Fun Facts

1. **INFANTRY_RAMBO** is the only unit with a movie character's name in the enum
2. **UNIT_STANK** includes a Star Trek reference in the code comments
3. **INFANTRY_EINSTEIN** brings Albert Einstein into the game as a playable character
4. **UNIT_DEMOTRUCK** is explicitly called a "Jihad truck" in source comments
5. **INFANTRY_RENOVATOR** (Engineer) has the friendliest-sounding internal name
6. **UNIT_MAD** references Cold War nuclear doctrine (Mutually Assured Destruction)
7. **Giant ants** (ANT1/2/3) reference the 1954 film "Them!"
8. **Vessel types** use authentic US Navy hull classification symbols
9. **All Soviet aircraft** use real NATO reporting names or Soviet designations
10. **UNIT_TRIC/TREX/RAPT/STEG** = Hidden dinosaur units (Jurassic Park influence?)

---

## Conclusion

The unit internal names in Command & Conquer reveal a development team deeply influenced by:
- **Action movies** (Rambo)
- **Sci-fi television** (Star Trek)
- **Military history** (Cold War, WWII)
- **1950s sci-fi** (giant ants, atomic age)
- **Alternate history** (What if Einstein changed the timeline?)

The code names balance **practical engineering** (short enums, clear prefixes) with **personality and humor** (Romulan cloaking, Jihad trucks, Renovators). This reflects the **1990s indie game development culture** where small teams had creative freedom to inject pop culture and inside jokes into their work.

These names are more than identifiers—they're **archaeological artifacts** of game development history, preserving the mindset, influences, and humor of the Westwood Studios team circa 1995-1996.

---

*This analysis is based on the released GPL v3 source code for Command & Conquer Remastered Collection (Tiberian Dawn and Red Alert).*
