# Command & Conquer Remastered Collection: Cheat Codes Analysis

## Table of Contents
1. [Overview](#overview)
2. [Cheat System Architecture](#cheat-system-architecture)
3. [Enabling Cheats](#enabling-cheats)
4. [Keyboard Cheat Codes](#keyboard-cheat-codes)
5. [Command-Line Parameters](#command-line-parameters)
6. [Special Options Dialog](#special-options-dialog)
7. [Debug Flags](#debug-flags)
8. [Implementation Details](#implementation-details)

---

## Overview

The Command & Conquer Remastered Collection (both Tiberian Dawn and Red Alert) contains an extensive cheat code system inherited from the original games. These cheats were used during development for debugging, playtesting, and quality assurance. The cheat system is conditionally compiled using the `#ifdef CHEAT_KEYS` preprocessor directive and is controlled by various debug flags.

The cheat system provides functionality for:
- **Instant victory/defeat** for testing mission flows
- **Free money** for testing unit/building balance
- **Map revelation** for level design verification
- **Instant building** for rapid prototyping
- **Unit spawning** for testing combat scenarios
- **Special game options** for tweaking game behavior

---

## Cheat System Architecture

### Compilation Control

The entire cheat system is wrapped in conditional compilation:

```cpp
#ifdef CHEAT_KEYS
    // Cheat code implementation
#endif
```

This means the cheat features can be completely removed from release builds by not defining the `CHEAT_KEYS` macro.

### Key System Components

1. **`Debug_Flag`** - Master cheat enable flag
2. **`Debug_Playtest`** - Playtest mode for limited cheats
3. **`SpecialClass`** - Game modification options
4. **`Debug_*` variables** - Individual cheat toggles
5. **Command-line parameters** - CRC-hashed developer codes

### Core Files

- **`DEBUG.CPP`** / **`DEBUG.H`** - Main cheat code handlers
- **`CONQUER.CPP`** - Keyboard processing and cheat activation
- **`SPECIAL.CPP`** / **`SPECIAL.H`** - Special options system
- **`DEFINES.H`** - Cheat system configuration
- **`EXTERNS.H`** - Debug flag declarations
- **`INIT.CPP`** - Command-line parameter processing

---

## Enabling Cheats

### Primary Methods

#### 1. Debug Flag Activation
The `Debug_Flag` variable must be set to `true` to enable most cheats. This can be done via:
- Command-line parameters (developer-specific)
- Code modification (for modders)
- Debugger attachment

#### 2. Playtest Mode
`Debug_Playtest` enables a limited subset of cheats, typically for QA testing:
- Win/lose cheats
- Map reveal (F4)
- Limited debug features

#### 3. Conditional Compilation
The `CHEAT_KEYS` preprocessor symbol must be defined during compilation for cheats to exist in the binary.

---

## Keyboard Cheat Codes

### Universal Cheats (Both Games)

#### Instant Victory
- **Key Combination:** `Alt+W`
- **Effect:** Immediately wins the current mission
- **Code Location:** `CONQUER.CPP`, `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case (int)KN_W|(int)KN_ALT_BIT:
      PlayerPtr->Flag_To_Win();
      break;
  ```

#### Instant Defeat
- **Key Combination:** `Alt+L`
- **Effect:** Immediately loses the current mission
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case (int)KN_L|(int)KN_ALT_BIT:
      PlayerPtr->Flag_To_Lose();
      break;
  ```

#### Free Money
- **Key Combination:** `Shift+M`, `Alt+M`, or `Ctrl+M`
- **Effect:** Adds 10,000 credits to player's account (Tiberian Dawn)
- **Effect (Red Alert):** Adds 10,000 credits to ALL players
- **Code Location:** `CONQUER.CPP`
- **Tiberian Dawn Implementation:**
  ```cpp
  case (int)KN_M|(int)KN_SHIFT_BIT:
  case (int)KN_M|(int)KN_ALT_BIT:
  case (int)KN_M|(int)KN_CTRL_BIT:
      PlayerPtr->Credits += 10000;
      break;
  ```
- **Red Alert Implementation:**
  ```cpp
  case int(int(KN_M) | int(KN_SHIFT_BIT)):
  case int(int(KN_M) | int(KN_ALT_BIT)):
  case int(int(KN_M) | int(KN_CTRL_BIT)):
      for (h = HOUSE_FIRST; h < HOUSE_COUNT; h++) {
          Houses.Ptr(h)->Refund_Money(10000);
      }
      break;
  ```

#### Special Options Dialog
- **Key:** `/` (forward slash)
- **Requirement:** `Debug_Flag` must be true
- **Effect:** Opens special game options dialog
- **Code Location:** `CONQUER.CPP`
- **Implementation:**
  ```cpp
  if (Debug_Flag && input == KN_SLASH) {
      if (GameToPlay != GAME_NORMAL) {
          SpecialDialog = SDLG_SPECIAL;
      } else {
          Special_Dialog();
      }
  }
  ```

#### Toggle Pathfinding Debug
- **Key:** `F`
- **Effect:** Toggles pathfinding visualization
- **Code Location:** `DEBUG.CPP`
- **Variable:** `Debug_Find_Path`
- **Implementation:**
  ```cpp
  case KN_F:
      Debug_Find_Path ^= 1;
      break;
  ```

#### Delete Selected Unit
- **Key:** `Delete`
- **Effect:** Removes the currently selected object from the game
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case KN_DELETE:
      if (CurrentObject.Count()) {
          Map.Recalc();
          delete CurrentObject[0];
      }
      break;
  ```

#### Damage Selected Unit
- **Key:** `Shift+Delete`
- **Effect:** Deals 50 damage to currently selected object
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case (int)KN_DELETE|(int)KN_SHIFT_BIT:
      if (CurrentObject.Count()) {
          Map.Recalc();
          int damage = 50;
          CurrentObject[0]->Take_Damage(damage, 0, WARHEAD_SA);
      }
      break;
  ```

#### Clone Selected Unit
- **Key:** `Insert`
- **Effect:** Creates a copy of the selected object under cursor
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case KN_INSERT:
      if (CurrentObject.Count()) {
          Map.PendingObject = &CurrentObject[0]->Class_Of();
          Map.PendingHouse = CurrentObject[0]->Owner();
          Map.PendingObjectPtr = Map.PendingObject->Create_One_Of(...);
          Map.Set_Cursor_Shape(Map.PendingObject->Occupy_List());
      }
      break;
  ```

#### Toggle Mono Debug Screen
- **Key:** `M`
- **Effect:** Enables/disables monochrome debug monitor
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case KN_M:
      if (Debug_Flag) {
          if (MonoClass::Is_Enabled()) {
              MonoClass::Disable();
          } else {
              MonoClass::Enable();
          }
      }
      break;
  ```

#### Enable Full Cheat Mode
- **Key:** `C`
- **Effect:** Enables comprehensive cheat mode and adds 3 nuke pieces (TD)
- **Code Location:** `DEBUG.CPP`
- **Variable:** `Debug_Cheat`
- **Tiberian Dawn Implementation:**
  ```cpp
  case KN_C:
      Debug_Cheat = (Debug_Cheat == false);
      PlayerPtr->IsRecalcNeeded = true;
      PlayerPtr->Add_Nuke_Piece();
      PlayerPtr->Add_Nuke_Piece();
      PlayerPtr->Add_Nuke_Piece();
      // Updates buildable options
      if (!ScenarioInit) {
          Map.Recalc();
          for (int index = 0; index < Buildings.Count(); index++) {
              Buildings.Ptr(index)->Update_Buildables();
          }
      }
      break;
  ```

#### Zoom Map View
- **Key Combination:** `Alt+Z`
- **Effect:** Toggles between normal and full map view
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case (int)KN_Z|(int)KN_ALT_BIT:
      if (map_x == -1) {
          map_x = Map.MapCellX;
          map_y = Map.MapCellY;
          map_width = Map.MapCellWidth;
          map_height = Map.MapCellHeight;
          Map.MapCellX = 1;
          Map.MapCellY = 1;
          Map.MapCellWidth = 62;  // TD
          Map.MapCellHeight = 62; // TD
          // Red Alert uses MAP_CELL_W-2 and MAP_CELL_H-2
      } else {
          // Restore original view
      }
      break;
  ```

### Tiberian Dawn Specific Cheats

#### Spawn Orca
- **Key:** `O`
- **Effect:** Creates an Orca helicopter at mouse cursor
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case KN_O:
      AircraftClass * air = new AircraftClass(AIRCRAFT_ORCA, PlayerPtr->Class->House);
      if (air) {
          air->Altitude = 0;
          air->Unlimbo(Map.Pixel_To_Coord(Get_Mouse_X(), Get_Mouse_Y()), DIR_N);
      }
      break;
  ```

#### Spawn Apache
- **Key:** `B`
- **Effect:** Creates an Apache helicopter at mouse cursor
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case KN_B:
      AircraftClass * air = new AircraftClass(AIRCRAFT_HELICOPTER, PlayerPtr->Class->House);
      if (air) {
          air->Altitude = 0;
          air->Unlimbo(Map.Pixel_To_Coord(Get_Mouse_X(), Get_Mouse_Y()), DIR_N);
      }
      break;
  ```

#### Spawn Transport Helicopter
- **Key:** `T`
- **Effect:** Creates a transport helicopter at mouse cursor
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case KN_T:
      AircraftClass * air = new AircraftClass(AIRCRAFT_TRANSPORT, PlayerPtr->Class->House);
      if (air) {
          air->Altitude = 0;
          air->Unlimbo(Map.Pixel_To_Coord(Get_Mouse_X(), Get_Mouse_Y()), DIR_N);
      }
      break;
  ```

#### Create Explosion
- **Key:** `` ` `` (grave/tilde)
- **Effect:** Creates large explosion at mouse cursor
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case KN_GRAVE:
      new AnimClass(ANIM_ART_EXP1, Map.Pixel_To_Coord(Get_Mouse_X(), Get_Mouse_Y()));
      Explosion_Damage(Map.Pixel_To_Coord(Get_Mouse_X(), Get_Mouse_Y()), 250, NULL, WARHEAD_HE);
      break;
  ```

#### Trigger GDI Ending
- **Key:** `Z`
- **Effect:** Plays GDI ending sequence
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case KN_Z:
      GDI_Ending();
      break;
  ```

#### Make Selected Unit Cloakable
- **Key:** `R`
- **Effect:** Grants cloaking ability to selected unit
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case KN_R:
      if (CurrentObject.Count()) {
          ((TechnoClass *)CurrentObject[0])->IsCloakable = true;
      }
      break;
  ```

#### Delete Team
- **Key:** `D`
- **Effect:** Deletes the first team in the list
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case KN_D:
      if (Teams.Ptr(0)) {
          delete Teams.Ptr(0);
      }
      break;
  ```

#### Toggle Free Scrolling
- **Key:** `F` (in CONQUER.CPP keyboard processing)
- **Effect:** Allows camera to scroll freely beyond normal bounds
- **Code Location:** `CONQUER.CPP`
- **Implementation:**
  ```cpp
  case VK_F:
      Options.IsFreeScroll = (Options.IsFreeScroll == false);
      break;
  ```

#### Stop Recording/Save Screenshot
- **Key:** `Ctrl+K`
- **Effect:** Initiates screen recording mode
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case (int)KN_K|(int)KN_CTRL_BIT:
      ScreenRecording = true;
      break;
  ```

#### Save Screenshot (PCX)
- **Key:** `K`
- **Effect:** Saves current screen as PCX file (scrsht00.pcx - scrsht99.pcx)
- **Code Location:** `DEBUG.CPP`
- **Note:** Commented out in remastered version (`#if (0)`)

#### Pause Game
- **Key:** `P`
- **Effect:** Pauses game until any key is pressed
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case KN_P:
      Keyboard::Clear();
      while (!Keyboard::Check()) {
          Self_Regulate();
          Sound_Callback();
      }
      Keyboard::Clear();
      break;
  ```

#### Net Debug Toggle
- **Key:** `L`
- **Effect:** Toggles network debugging display
- **Code Location:** `DEBUG.CPP`
- **Implementation:**
  ```cpp
  case KN_L:
      extern int NetMonoMode, NewMonoMode;
      if (NetMonoMode)
          NetMonoMode = 0;
      else
          NetMonoMode = 1;
      NewMonoMode = 1;
      break;
  ```

### Red Alert Specific Cheats

#### Spawn Hind
- **Key:** `O`
- **Effect:** Creates a Hind attack helicopter at mouse cursor
- **Code Location:** `DEBUG.CPP` (Red Alert)
- **Implementation:**
  ```cpp
  case KN_O:
      AircraftClass * air = new AircraftClass(AIRCRAFT_HIND, PlayerPtr->Class->House);
      if (air) {
          air->Height = 0;
          air->Unlimbo(Map.Pixel_To_Coord(Get_Mouse_X(), Get_Mouse_Y()), DIR_N);
      }
      break;
  ```

#### Spawn Longbow
- **Key:** `B`
- **Effect:** Creates a Longbow attack helicopter at mouse cursor
- **Code Location:** `DEBUG.CPP` (Red Alert)
- **Implementation:**
  ```cpp
  case KN_B:
      AircraftClass * air = new AircraftClass(AIRCRAFT_LONGBOW, PlayerPtr->Class->House);
      if (air) {
          air->Height = 0;
          air->Unlimbo(Map.Pixel_To_Coord(Get_Mouse_X(), Get_Mouse_Y()), DIR_N);
      }
      break;
  ```

#### Random Explosion
- **Key:** `` ` `` (grave/tilde)
- **Effect:** Creates explosion with random warhead at mouse cursor
- **Code Location:** `DEBUG.CPP` (Red Alert)
- **Implementation:**
  ```cpp
  case KN_GRAVE:
      WarheadType warhead = Random_Pick(WARHEAD_HE, WARHEAD_FIRE);
      COORDINATE coord = Map.Pixel_To_Coord(Get_Mouse_X(), Get_Mouse_Y());
      int damage = 1000;
      new AnimClass(Combat_Anim(damage, warhead, Map[coord].Land_Type()), coord);
      Explosion_Damage(coord, damage, NULL, warhead);
      break;
  ```

#### Trigger Chronal Vortex
- **Key:** `Backspace`
- **Effect:** Activates/deactivates Chronal Vortex at mouse cursor
- **Code Location:** `DEBUG.CPP` (Red Alert)
- **Implementation:**
  ```cpp
  case KN_BACKSPACE:
      if (ChronalVortex.Is_Active()) {
          ChronalVortex.Disappear();
      } else {
          int xxxx = Get_Mouse_X() + Map.TacPixelX;
          int yyyy = Get_Mouse_Y() + Map.TacPixelY;
          CELL cell = Map.DisplayClass::Click_Cell_Calc(xxxx, yyyy);
          ChronalVortex.Appear(Cell_Coord(cell));
      }
      break;
  ```

#### Grant All Superweapons
- **Key:** `P`
- **Effect:** Enables and fully charges all superweapons
- **Code Location:** `DEBUG.CPP` (Red Alert)
- **Implementation:**
  ```cpp
  case KN_P:
      for (SpecialWeaponType spc = SPC_FIRST; spc < SPC_COUNT; spc++) {
          PlayerPtr->SuperWeapon[spc].Enable(true, true);
          PlayerPtr->SuperWeapon[spc].Forced_Charge(true);
          Map.Add(RTTI_SPECIAL, spc);
          Map.Column[1].Flag_To_Redraw();
      }
      break;
  ```

#### Flash Power/Money Indicators
- **Key:** `I`
- **Effect:** Causes power and money indicators to flash
- **Code Location:** `DEBUG.CPP` (Red Alert)
- **Implementation:**
  ```cpp
  case KN_I:
      Map.Flash_Power();
      Map.Flash_Money();
      break;
  ```

#### Toggle Unit Range Display
- **Key:** `F7`
- **Effect:** Shows sight (white) and weapon (red) ranges for selected unit
- **Code Location:** `DEBUG.CPP` (Red Alert)
- **Implementation:**
  ```cpp
  case KN_F7:
      if (CurrentObject.Count() && CurrentObject[0]->Is_Techno()) {
          TechnoTypeClass const & ttype = (TechnoTypeClass const &)CurrentObject[0]->Class_Of();
          int sight = ((int)ttype.SightRange) << 8;
          int weapon = 0;
          if (ttype.PrimaryWeapon != NULL) weapon = ttype.PrimaryWeapon->Range;
          // Draws circles around unit
      }
      break;
  ```

#### Toggle Icon Debug
- **Key:** `V` or `F3`
- **Effect:** Toggles debug icon visualization
- **Code Location:** `DEBUG.CPP` (Red Alert)
- **Variable:** `Debug_Icon`
- **Implementation:**
  ```cpp
  case KN_V:
  case KN_F3:
      Debug_Icon = (Debug_Icon == false);
      Map.Flag_To_Redraw(true);
      break;
  ```

#### Toggle Map Reveal
- **Key:** `F4` (with Debug_Playtest or Debug_Flag)
- **Effect:** Reveals/unreveals entire map
- **Code Location:** `CONQUER.CPP` (Red Alert)
- **Variable:** `Debug_Unshroud`
- **Implementation:**
  ```cpp
  if ((Debug_Flag || Debug_Playtest) && plain == KN_F4) {
      if (Session.Type == GAME_NORMAL) {
          Debug_Unshroud = (Debug_Unshroud == false);
          Map.Flag_To_Redraw(true);
      }
  }
  ```

#### Cycle Debug Pages
- **Key:** `[` (left bracket) or `F11` - Previous page
- **Key:** `]` (right bracket) or `F12` - Next page
- **Effect:** Cycles through mono debug display pages
- **Code Location:** `DEBUG.CPP` (Red Alert)
- **Variable:** `MonoPage`
- **Implementation:**
  ```cpp
  case KN_LBRACKET:
  case KN_F11:
      if (MonoPage == DMONO_FIRST) {
          MonoPage = DMonoType(DMONO_COUNT-1);
      } else {
          MonoPage = DMonoType(MonoPage - 1);
      }
      DebugTimer = 0;
      break;
  ```

#### Start Motion Capture
- **Key:** `J` (Windows only)
- **Effect:** Enables motion capture recording
- **Code Location:** `DEBUG.CPP` (Red Alert)
- **Variable:** `Debug_MotionCapture`
- **Implementation:**
  ```cpp
  #ifdef WIN32
  case KN_J:
      Debug_MotionCapture = true;
      break;
  #endif
  ```

#### Force Chronal Vortex Debug
- **Key:** `Ctrl+F4`
- **Effect:** Toggles unshroud/reveal in multiplayer
- **Code Location:** `DEBUG.CPP` (Red Alert)
- **Implementation:**
  ```cpp
  case ((int)KN_F4 | (int)KN_CTRL_BIT):
      Debug_Unshroud = (Debug_Unshroud == false);
      Map.Flag_To_Redraw(true);
      break;
  ```

---

## Command-Line Parameters

### Developer-Specific Cheat Parameters

The game uses CRC hash values to verify developer-specific command-line parameters. These are defined in `DEFINES.H`:

```cpp
#define PARM_CHEATDAVID    0xBE79088C  // David Dettmer
#define PARM_CHEATERIK     0x9F38A19D  // Erik Yeo
#define PARM_CHEATJOE      0xABDD0362  // Joe Bostic
#define PARM_CHEATPHIL     0x8CEE96D0  // Phil Gorrow
#define PARM_CHEATBILL     0x8A8AEB37  // Bill Randolph
#define PARM_CHEATADAM     0xADD92C9F  // Adam Isgreen
#define PARM_CHEATMIKE     0x999E3F52  // Mike Legg
#define PARM_CHEAT_STEVET  0xE6FF47E8  // Steve Tall

#define PARM_EDITORBILL    0x3A01ECF5  // Map editor for Bill
#define PARM_EDITORERIK    0xD940EEB8  // Map editor for Erik
```

### Parameter Processing

These parameters are processed in `INIT.CPP`:

```cpp
switch (Obfuscate(string)) {  // Obfuscate() calculates CRC hash
    case PARM_CHEATERIK:
        Debug_Playtest = true;
        Debug_Flag = true;
        break;
    
    case PARM_CHEATDAVID:
        Debug_Playtest = true;
        Debug_Flag = true;
        break;
    
    // ... similar for other developers
    
    case PARM_EDITORBILL:
        Debug_Map = true;
        Debug_Unshroud = true;
        Debug_Flag = true;
        break;
}
```

### Standard Parameters

#### Easy Mode
- **Parameter:** `-EASY`
- **Hash:** `PARM_EASY`
- **Effect:** Sets game to easy difficulty
- **Implementation:**
  ```cpp
  case PARM_EASY:
      Special.IsEasy = true;
      Special.IsDifficult = false;
      break;
  ```

#### Hard Mode
- **Parameter:** `-HARD`
- **Hash:** `PARM_HARD`
- **Effect:** Sets game to hard difficulty
- **Implementation:**
  ```cpp
  case PARM_HARD:
      Special.IsEasy = false;
      Special.IsDifficult = true;
      break;
  ```

#### Playtest Mode
- **Parameter:** `-PLAYTEST` (only with `VIRGIN_CHEAT_KEYS` defined)
- **Hash:** `PARM_PLAYTEST`
- **Effect:** Enables playtest mode
- **Implementation:**
  ```cpp
  #ifdef VIRGIN_CHEAT_KEYS
  case PARM_PLAYTEST:
      Debug_Playtest = true;
      break;
  #endif
  ```

#### Special Mode (Jurassic)
- **Parameter:** `-SPECIAL`
- **Hash:** `PARM_SPECIAL`
- **Effect:** Enables Jurassic mode and "thingies"
- **Implementation:**
  ```cpp
  case PARM_SPECIAL:
      Special.IsJurassic = true;
      AreThingiesEnabled = true;
      break;
  ```

#### Install Mode
- **Parameter:** `-INSTALL`
- **Hash:** `PARM_INSTALL`
- **Effect:** Runs game from installer
- **Implementation:**
  ```cpp
  case PARM_INSTALL:
  #ifndef DEMO
      Special.IsFromInstall = true;
  #endif
      break;
  ```

#### Check Map Mode
- **Parameter:** `-CHECKMAP`
- **Effect:** Enables map validation mode
- **Implementation:**
  ```cpp
  if (stricmp(string, "-CHECKMAP") == 0) {
      Debug_Check_Map = true;
  }
  ```

#### Version Override
- **Parameter:** `-O` or `-0`
- **Effect:** Enables compatibility with v1.07
- **Implementation:**
  ```cpp
  if (stricmp(string, "-O") == 0 || stricmp(string, "-0") == 0) {
      IsV107 = true;
  }
  ```

#### CD Path Override
- **Parameter:** `-CD<path>`
- **Effect:** Overrides default file search path
- **Implementation:**
  ```cpp
  if (strstr(string, "-CD")) {
      CCFileClass::Set_Search_Drives(&string[3]);
  }
  ```

---

## Special Options Dialog

The Special Options Dialog (`Special_Dialog()` function in `SPECIAL.CPP`) provides a user interface for toggling various game behavior options. Access it by pressing `/` when `Debug_Flag` is enabled.

### Tiberian Dawn Options

1. **TXT_SEPARATE_HELIPAD**
   - Variable: `Special.IsSeparate`
   - Effect: Separates helipad from airfield building

2. **TXT_VISIBLE_TARGET**
   - Variable: `Special.IsVisibleTarget`
   - Effect: Makes targeting visible/explicit

3. **TXT_TREE_TARGET**
   - Variable: `Special.IsTreeTarget`
   - Effect: Allows targeting of trees

4. **TXT_MCV_DEPLOY**
   - Variable: `Special.IsMCVDeploy`
   - Effect: Controls MCV deployment behavior

5. **TXT_SMART_DEFENCE**
   - Variable: `Special.IsSmartDefense`
   - Effect: Enables intelligent defensive AI

6. **TXT_THREE_POINT**
   - Variable: `Special.IsThreePoint`
   - Effect: Uses three-point turn pathfinding

7. **TXT_TIBERIUM_FAST**
   - Variable: `Special.IsTFast`
   - Effect: Speeds up Tiberium growth/spread

8. **TXT_ROAD_PIECES**
   - Variable: `Special.IsRoad`
   - Effect: Displays road pieces

9. **TXT_SCATTER**
   - Variable: `Special.IsScatter`
   - Effect: Units scatter when attacked

10. **TXT_SHOW_NAMES**
    - Variable: `Special.IsNamed`
    - Effect: Displays unit/building names

### Red Alert Options

1. **IsSpeedBuild**
   - Effect: Instant building construction

2. **IsInert**
   - Effect: Makes units non-functional/inert

3. **IsTGrowth**
   - Default: `true`
   - Effect: Enables ore growth

4. **IsMCVDeploy**
   - Effect: Controls MCV deployment behavior

5. Additional options similar to Tiberian Dawn

### Dialog Implementation

```cpp
void Special_Dialog(void)
{
    SpecialClass oldspecial = Special;
    // Creates dialog with checkboxes for each option
    
    // Initialize checkbox states from Special flags
    for (index = 0; index < option_count; index++) {
        switch (_options[index].Description) {
            case TXT_TREE_TARGET:
                value = Special.IsTreeTarget;
                break;
            // ... other options
        }
        _options[index].Setting = value;
    }
    
    // Display dialog and process input
    while (process) {
        // Input handling
        // Toggle checkboxes when clicked
    }
    
    // Apply changes via network event
    OutList.Add(EventClass(oldspecial));
}
```

---

## Debug Flags

### Global Debug Variables

Declared in `EXTERNS.H`:

```cpp
extern bool Debug_Quiet;          // Suppress debug output
extern bool Debug_Cheat;          // Master cheat flag
extern bool Debug_Remap;          // Debug color remapping
extern bool Debug_Flag;           // Master debug enable
extern bool Debug_Lose;           // Force lose condition
extern bool Debug_Map;            // Map editor mode
extern bool Debug_Win;            // Force win condition
extern bool Debug_Icon;           // Debug icon rendering
extern bool Debug_Passable;       // Show passability
extern bool Debug_Unshroud;       // Reveal entire map
extern bool Debug_Threat;         // Display threat values
extern bool Debug_Find_Path;      // Visualize pathfinding
extern bool Debug_Check_Map;      // Validate map integrity
extern bool Debug_Playtest;       // Playtest mode

extern bool Debug_Heap_Dump;      // Dump heap information
extern bool Debug_Smart_Print;    // Enhanced debug printing
extern bool Debug_Trap_Check_Heap;// Heap validation traps
extern bool Debug_Instant_Build;  // Instant construction
extern bool Debug_Force_Crash;    // Force crash for testing
```

### Flag Behavior

#### Debug_Cheat
When enabled (`C` key):
- Unlocks all prerequisites for building
- Adds Ion Cannon/Nuclear Strike pieces
- Forces recalculation of buildable options
- Updates all buildings' construction lists

```cpp
Debug_Cheat = (Debug_Cheat == false);
PlayerPtr->IsRecalcNeeded = true;
PlayerPtr->Add_Nuke_Piece();  // Add 3 pieces in TD
```

#### Debug_Unshroud
When enabled:
- Removes shroud from entire map
- Shows all units and terrain
- Useful for level design and testing
- Can be toggled with F4 in Red Alert playtest mode

```cpp
if (Debug_Unshroud) {
    // Map rendering skips shroud checks
}
```

#### Debug_Instant_Build
When enabled (`Alt+B` in Tiberian Dawn):
- Buildings construct instantly
- No build time required
- Useful for rapid base construction testing

```cpp
case (int)KN_B|(int)KN_ALT_BIT:
    Debug_Instant_Build ^= 1;
    break;
```

#### Debug_Find_Path
When enabled (`F` key):
- Visualizes pathfinding calculations
- Shows path nodes and search areas
- Helps debug unit navigation issues

#### Debug_Icon
When enabled (`V` or `F3` in Red Alert):
- Shows debug rendering information
- Displays bounding boxes
- Visualizes collision data

---

## Implementation Details

### Input Processing Flow

1. **Main Game Loop** (`Main_Loop()` in `CONQUER.CPP`)
   - Captures keyboard input via `input = Get_Input()`
   - Passes to `Keyboard_Process(input)`

2. **Keyboard Processing** (`Keyboard_Process()` in `CONQUER.CPP`)
   - Filters input through message system
   - Strips modifier bits to get plain key
   - Checks for cheat key combinations
   - Calls `Debug_Key(input)` if `Debug_Flag` is set

3. **Debug Key Handler** (`Debug_Key()` in `DEBUG.CPP`)
   - Processes individual cheat codes
   - Only active when `Debug_Flag == true`
   - Contains game-specific debug functions

### Key Input Structure

```cpp
// Key modifiers defined in KEYBOARD.H
#define KN_SHIFT_BIT    0x2000
#define KN_CTRL_BIT     0x4000
#define KN_ALT_BIT      0x8000
#define KN_RLSE_BIT     0x8000  // Key release
#define KN_BUTTON       0x8000  // Mouse button

// Example key combination check
if (input == (KN_W | KN_ALT_BIT)) {
    // Alt+W pressed
}
```

### Conditional Compilation Hierarchy

```
#define CHEAT_KEYS                    // Master cheat enable
    ↓
    #define VIRGIN_CHEAT_KEYS         // Publisher-specific cheats
        ↓
        #define PARM_CHEATERIK        // Developer-specific parameters
        #define PARM_CHEATDAVID
        // etc...
```

### CRC Hash System

The command-line parameter system uses CRC hashing to obfuscate cheat codes:

```cpp
unsigned long Obfuscate(char const * string)
{
    // Calculate CRC32 hash of input string
    // Prevents casual discovery of cheat parameters
    return Calculate_CRC(string, strlen(string));
}

// Usage in parameter processing
switch (Obfuscate(argv[i])) {
    case PARM_CHEATERIK:  // 0x9F38A19D
        // Enable cheats
        break;
}
```

This means the actual command-line parameter is hashed before comparison, making it difficult to discover the original strings without source code access.

### Network Synchronization

In multiplayer games, cheat activation must be synchronized:

```cpp
// Special options are sent via network events
OutList.Add(EventClass(oldspecial));

// This ensures all players receive the same game state changes
```

### Cheat Detection

The game doesn't actively prevent or detect cheats, as they were development tools. However, in multiplayer:
- Certain cheats are disabled (`Session.Type != GAME_NORMAL` checks)
- Some cheats work only in single-player
- Network events ensure all players see the same state

---

## Security Considerations

### Why Cheats Weren't Removed

1. **Historical Preservation**: The remastered collection aims to preserve the original games
2. **Modding Support**: Enables modders and map creators to test content
3. **Development Legacy**: Shows how QA and development teams worked
4. **Single-Player Focus**: Most cheats only work in single-player mode

### Multiplayer Protection

```cpp
// Example of multiplayer cheat restriction
if (Session.Type == GAME_NORMAL) {
    // Cheat allowed in single-player
    Debug_Unshroud = !Debug_Unshroud;
} else {
    // Restricted in multiplayer
}
```

### Enabling Cheats Safely

For modders wanting to enable cheats:

1. **Set `CHEAT_KEYS` during compilation**
   ```cpp
   #define CHEAT_KEYS
   ```

2. **Enable `Debug_Flag` at runtime**
   ```cpp
   Debug_Flag = true;
   ```

3. **Use command-line parameters** (requires knowing original strings or modifying code)

---

## Conclusion

The Command & Conquer Remastered Collection's cheat system represents a comprehensive suite of development and debugging tools used during the original games' creation. The system demonstrates:

- **Modular Design**: Clean separation between cheat code, game logic, and compilation flags
- **Developer Workflow**: Shows how Westwood Studios tested and balanced gameplay
- **Network Awareness**: Proper synchronization in multiplayer contexts
- **Security Through Obscurity**: CRC hashing of command-line parameters
- **Maintainability**: Well-organized code structure for easy modification

These cheats provide invaluable insight into game development practices of the 1990s and continue to serve modern modders and content creators working with the Remastered Collection.

---

## Appendix: Quick Reference

### Most Useful Cheats

| Cheat | Key | Effect | Games |
|-------|-----|--------|-------|
| Win Mission | `Alt+W` | Instant victory | Both |
| Lose Mission | `Alt+L` | Instant defeat | Both |
| Free Money | `Shift/Alt/Ctrl+M` | +10,000 credits | Both |
| Reveal Map | `F4` | Show entire map | RA (Playtest) |
| Full Cheats | `C` | Enable all cheats | Both |
| Special Options | `/` | Open options dialog | Both |
| Delete Unit | `Delete` | Remove selected object | Both |
| Clone Unit | `Insert` | Copy selected object | Both |
| Spawn Orca | `O` | Create Orca helicopter | TD |
| Spawn Hind | `O` | Create Hind helicopter | RA |
| Explosion | `` ` `` | Create explosion | Both |
| Superweapons | `P` | Grant all superweapons | RA |

### Flag Summary

| Flag | Default | Effect |
|------|---------|--------|
| `Debug_Flag` | `false` | Master debug enable |
| `Debug_Cheat` | `false` | Unlock all building prerequisites |
| `Debug_Unshroud` | `false` | Reveal entire map |
| `Debug_Playtest` | `false` | Limited cheat access |
| `Debug_Instant_Build` | `false` | Zero-time construction |
| `Debug_Find_Path` | `false` | Visualize pathfinding |
| `Debug_Map` | `false` | Map editor mode |

---

*Document Version: 1.0*  
*Last Updated: 2024*  
*Based on: C&C Remastered Collection Source Code (EA/Petroglyph)*
