# Command & Conquer Red Alert - Map Editor System Analysis

## Table of Contents
1. [Overview](#overview)
2. [MapEditClass Architecture](#mapeditclass-architecture)
3. [Conditional Compilation](#conditional-compilation)
4. [Object Placement System](#object-placement-system)
5. [Editor UI Components](#editor-ui-components)
6. [Scenario Persistence](#scenario-persistence)
7. [Waypoint System](#waypoint-system)
8. [House Management](#house-management)
9. [Editor Main Loop](#editor-main-loop)
10. [File Organization](#file-organization)

## Overview

The Red Alert map editor is a sophisticated in-game tool for creating and modifying scenarios. The editor is built directly into the game executable and is activated through conditional compilation with the `SCENARIO_EDITOR` flag. This integration allows developers and modders to use the same engine that powers the game for level creation.

**Key Features:**
- Full object placement with live preview
- Scenario save/load with INI persistence  
- Waypoint and trigger management
- Multi-house support with faction-specific objects
- Base building percentage control
- Team and mission editing
- Real-time validation of placement rules

**Implementation Stats:**
- **Primary Class:** `MapEditClass` (extends `MouseClass`)
- **Source Files:** 5 modules totaling ~8,500 lines
- **Object Types:** 9 categories (templates, overlays, smudges, terrain, units, infantry, vessels, buildings, aircraft)
- **Maximum Objects:** Defined by `MAX_EDIT_OBJECTS` constant

## MapEditClass Architecture

### Class Hierarchy

```
DisplayClass
    ↓
MouseClass  
    ↓
MapEditClass ← Main editor controller
```

The `MapEditClass` extends the mouse handling system to add editor-specific functionality while preserving all normal game map operations.

### Core Data Members

```cpp
class MapEditClass : public MouseClass
{
private:
    // Object library for placement
    ObjectTypeClass const * Objects[MAX_EDIT_OBJECTS];
    int ObjCount;                     // # of objects in Objects array
    
    // Placement state tracking
    int LastChoice;                   // Index of last selected object
    HousesType LastHouse;             // House of last placed object
    
    // Object manipulation
    ObjectClass * GrabbedObject;      // Currently grabbed object
    CELL GrabOffset;                  // Offset to grabbed obj's upper-left
    unsigned long LastClickTime;      // For double-click detection
    
    // Category management (9 types)
    int NumType[NUM_EDIT_CLASSES];    // Count per category:
    // 0 = Template, 1 = Overlay, 2 = Smudge, 3 = Terrain
    // 4 = Unit, 5 = Infantry, 6 = Vessels, 7 = Building, 8 = Aircraft
    
    int TypeOffset[NUM_EDIT_CLASSES]; // Starting index for each category
    
    // Base building state
    bool BaseBuilding;                // Building AI base?
    CELL CurrentCell;                 // Current cursor position
    TeamTypeClass * CurTeam;          // Current team being edited
    TriggerTypeClass * CurTrigger;    // Current trigger
    int Changed;                      // Scenario modified flag
    bool LMouseDown;                  // Left button state for painting
};
```

### Method Groups

**Initialization & Setup:**
```cpp
void One_Time(void);                  // One-time initialization of controls
void Init_IO(void);                   // Initialize button input list
void Clear_List(void);                // Clear object selection list
bool Add_To_List(ObjectTypeClass const *object);  // Add placeable object
```

**Main Control:**
```cpp
void AI(KeyNumType &input, int x, int y);  // Main editor logic
void Draw_It(bool forced = true);          // Render editor overlays
void Main_Menu(void);                      // Process main menu
bool Mouse_Moved(void);                    // Detect mouse motion
```

**Object Placement (mapedplc.cpp):**
```cpp
int Placement_Dialog(void);           // Object selection dialog
void Start_Placement(void);           // Begin placement mode
int Place_Object(void);               // Place current object
void Cancel_Placement(void);          // Exit placement mode
void Place_Next(void);                // Next object in category
void Place_Prev(void);                // Previous object
void Place_Next_Category(void);       // Cycle to next category
void Place_Prev_Category(void);       // Cycle to previous category
void Place_Home(void);                // Jump to first object
void Toggle_House(void);              // Change owner faction
```

**Object Selection (mapedsel.cpp):**
```cpp
int Select_Object(void);              // Select object at cursor
void Select_Next(void);               // Select next object
void Popup_Controls(void);            // Show/hide property editor
void Grab_Object(void);               // Grab for moving
int Move_Grabbed_Object(void);        // Move grabbed object
bool Change_House(HousesType newhouse);  // Change object owner
```

**Scenario Management (mapeddlg.cpp):**
```cpp
int New_Scenario(void);               // Create new scenario
int Load_Scenario(void);              // Load from disk
int Save_Scenario(void);              // Save to disk
int Pick_Scenario(char const * caption, ...);  // Scenario picker
int Size_Map(int x, int y, int w, int h);      // Resize map
int Scenario_Dialog(void);            // Edit scenario properties
void Handle_Triggers(void);           // Trigger editor
int Select_Trigger(void);             // Trigger selection dialog
```

**Team Management (mapedtm.cpp):**
```cpp
void Draw_Member(TechnoTypeClass const * ptr, int index, 
                 int quant, HousesType house);
void Handle_Teams(char const * caption);
int Select_Team(char const * caption);
int Team_Members(HousesType house);
```

## Conditional Compilation

### Editor Activation

The entire map editor is wrapped in conditional compilation directives to control its inclusion in different build configurations:

```cpp
#ifdef SCENARIO_EDITOR

/*
** All editor code only exists when SCENARIO_EDITOR is defined.
** This keeps the main game executable clean and reduces size.
*/

// Editor-specific data
MissionType MapEditClass::MapEditMissions[] = {
    MISSION_GUARD,
    MISSION_STICKY,
    MISSION_HARMLESS,
    MISSION_HARVEST,
    MISSION_GUARD_AREA,
    MISSION_RETURN,
    MISSION_AMBUSH,
    MISSION_HUNT,
    MISSION_SLEEP,
};

#endif  // SCENARIO_EDITOR
```

### Runtime Detection

Even with the code compiled in, the editor is only active when `Debug_Map` is true:

```cpp
void MapEditClass::AI(KeyNumType & input, int x, int y)
{
    // Trap F2 regardless of mode
    if ((input == KN_F2 && Session.Type == GAME_NORMAL) || 
        input == (KN_F2 | KN_CTRL_BIT)) {
        
        if (Debug_Map && Changed) {
            // Prompt for saving changes
            rc = WWMessageBox().Process("Save Changes?", TXT_YES, TXT_NO);
            if (rc == 0) {
                if (Save_Scenario() != 0) {
                    input = KN_NONE;
                } else {
                    Changed = 0;
                    Go_Editor(!Debug_Map);  // Toggle mode
                }
            }
        }
    }
    
    // For normal game mode, skip editor AI
    if (!Debug_Map) {
        MouseClass::AI(input, x, y);
        return;
    }
    
    // Editor-specific processing continues...
}
```

### Build Configuration

**Developer Build:**
```makefile
# Scenario editor enabled
CFLAGS += -DSCENARIO_EDITOR
```

**Release Build:**
```makefile
# No editor flag - code completely removed
# (No SCENARIO_EDITOR define)
```

This approach provides:
- **Zero overhead** in release builds (code doesn't exist)
- **Same executable** used for development and level creation
- **Instant toggling** with F2 key in debug builds
- **Clean separation** of editor and game logic

## Object Placement System

### Placement State Machine

The editor maintains distinct modes for object placement:

```
IDLE ──────────────> PLACEMENT ──────────> PAINTING
  ↑                      |                      |
  |                      ↓                      ↓
  └──────────────── CONFIRM/CANCEL ──── PLACE_OBJECT
```

**States:**
- **IDLE:** Normal map browsing, object selection allowed
- **PLACEMENT:** Object selected, cursor shows ghost preview
- **PAINTING:** Left mouse held, rapid placement on drag
- **CONFIRM:** Object placement validation

### Add_To_List Implementation

Building the object library at editor startup:

```cpp
bool MapEditClass::Add_To_List(ObjectTypeClass const * object)
{
    // Validate space available
    if (object && ObjCount < MAX_EDIT_OBJECTS) {
        Objects[ObjCount++] = object;
        
        // Update category counters for quick access
        switch (object->What_Am_I()) {
            case RTTI_TEMPLATETYPE:
                NumType[0]++;  // Templates (clear terrain)
                break;
                
            case RTTI_OVERLAYTYPE:
                NumType[1]++;  // Overlays (ore, walls, crates)
                break;
                
            case RTTI_SMUDGETYPE:
                NumType[2]++;  // Smudges (craters, scorch marks)
                break;
                
            case RTTI_TERRAINTYPE:
                NumType[3]++;  // Terrain (trees, rocks)
                break;
                
            case RTTI_UNITTYPE:
                NumType[4]++;  // Vehicles
                break;
                
            case RTTI_INFANTRYTYPE:
                NumType[5]++;  // Infantry
                break;
                
            case RTTI_VESSELTYPE:
                NumType[6]++;  // Naval units
                break;
                
            case RTTI_BUILDINGTYPE:
                NumType[7]++;  // Structures
                break;
                
            case RTTI_AIRCRAFTTYPE:
                NumType[8]++;  // Aircraft
                break;
        }
        return true;
    }
    return false;
}
```

### Start_Placement Flow

Entering placement mode with keyboard/menu:

```cpp
void MapEditClass::Start_Placement(void)
{
    // Show placement dialog to pick object
    if (Placement_Dialog()) {
        PendingObject = Objects[LastChoice];
        PendingHouse = LastHouse;
        
        // Set appropriate cursor shape for object
        Map.Set_Cursor_Shape(PendingObject->Occupy_List());
        Map.Set_Cursor_Pos(CurrentCell);
        
        // Enable rapid placement mode
        LMouseDown = false;
        
        // Visual feedback
        Flag_To_Redraw(true);
    }
}
```

### Place_Object Validation

The core placement logic with rule checking:

```cpp
int MapEditClass::Place_Object(void)
{
    if (!PendingObject) return -1;
    
    // Get target cell
    CELL cell = CurrentCell;
    if (cell == -1) return -1;
    
    // Validate placement legality
    ObjectClass * obj = NULL;
    
    switch (PendingObject->What_Am_I()) {
        case RTTI_BUILDINGTYPE: {
            BuildingTypeClass const * btype = 
                (BuildingTypeClass const *)PendingObject;
            
            // Check foundation footprint
            if (!Map.Legal_Placement(cell, btype, PendingHouse)) {
                return -1;  // Invalid placement
            }
            
            // Create building
            BuildingClass * building = new BuildingClass(
                btype->Type, PendingHouse);
            
            if (building && building->Unlimbo(Cell_Coord(cell))) {
                obj = building;
                Changed = 1;  // Mark scenario as modified
            } else {
                delete building;
                return -1;
            }
            break;
        }
        
        case RTTI_UNITTYPE: {
            UnitTypeClass const * utype = 
                (UnitTypeClass const *)PendingObject;
            
            // Check passability
            if (!Map[cell].Is_Clear_To_Move(SPEED_TRACK, false, false)) {
                return -1;
            }
            
            // Create unit
            UnitClass * unit = new UnitClass(utype->Type, PendingHouse);
            if (unit && unit->Unlimbo(Cell_Coord(cell), DIR_N)) {
                obj = unit;
                Changed = 1;
            } else {
                delete unit;
                return -1;
            }
            break;
        }
        
        case RTTI_INFANTRYTYPE: {
            InfantryTypeClass const * itype = 
                (InfantryTypeClass const *)PendingObject;
            
            // Infantry can share cells (5 per cell)
            InfantryClass * infantry = new InfantryClass(
                itype->Type, PendingHouse);
            
            if (infantry && infantry->Unlimbo(Cell_Coord(cell), DIR_N)) {
                obj = infantry;
                Changed = 1;
            } else {
                delete infantry;
                return -1;
            }
            break;
        }
        
        case RTTI_TERRAINTYPE: {
            TerrainTypeClass const * ttype = 
                (TerrainTypeClass const *)PendingObject;
            
            // Check cell empty
            if (Map[cell].Cell_Object()) {
                return -1;  // Occupied
            }
            
            // Create terrain object
            TerrainClass * terrain = new TerrainClass(
                ttype->Type, cell);
            
            if (terrain && terrain->Unlimbo(Cell_Coord(cell))) {
                obj = terrain;
                Changed = 1;
            } else {
                delete terrain;
                return -1;
            }
            break;
        }
        
        case RTTI_OVERLAYTYPE: {
            OverlayTypeClass const * otype = 
                (OverlayTypeClass const *)PendingObject;
            
            // Set overlay directly on cell
            Map[cell].Overlay = otype->Type;
            Map[cell].OverlayData = 0;
            Map[cell].Recalc_Attributes();
            Map[cell].Redraw_Objects();
            Changed = 1;
            return 0;
        }
        
        case RTTI_SMUDGETYPE: {
            SmudgeTypeClass const * stype = 
                (SmudgeTypeClass const *)PendingObject;
            
            // Create smudge
            SmudgeClass * smudge = new SmudgeClass(stype->Type);
            if (smudge && smudge->Unlimbo(Cell_Coord(cell))) {
                Changed = 1;
            } else {
                delete smudge;
                return -1;
            }
            break;
        }
    }
    
    // Success - object placed
    if (obj) {
        obj->Mark(MARK_DOWN);
        Map.Radar_Pixel(cell);
        return 0;
    }
    
    return -1;
}
```

### Painting Mode

Rapid placement while dragging mouse:

```cpp
void MapEditClass::AI(KeyNumType & input, int x, int y)
{
    // ... other AI code ...
    
    // Check for mouse motion while left button is down
    bool rc = Mouse_Moved();
    if (LMouseDown && rc) {
        
        // "Paint" mode: place current object, and restart placement
        if (PendingObject) {
            Flag_To_Redraw(true);
            if (Place_Object() == 0) {
                Changed = 1;
                Start_Placement();  // Keep painting!
            }
        } else {
            // Move the currently-grabbed object
            if (GrabbedObject) {
                GrabbedObject->Mark(MARK_CHANGE);
                if (Move_Grabbed_Object() == 0) {
                    Changed = 1;
                }
            }
        }
    }
    
    // ... more AI code ...
}
```

### Category Navigation

Cycling through object types:

```cpp
void MapEditClass::Place_Next_Category(void)
{
    // Find current category
    int current_category = -1;
    for (int i = 0; i < NUM_EDIT_CLASSES; i++) {
        if (LastChoice >= TypeOffset[i] && 
            LastChoice < TypeOffset[i] + NumType[i]) {
            current_category = i;
            break;
        }
    }
    
    // Move to next non-empty category
    int next_category = (current_category + 1) % NUM_EDIT_CLASSES;
    while (NumType[next_category] == 0) {
        next_category = (next_category + 1) % NUM_EDIT_CLASSES;
    }
    
    // Jump to first object in that category
    LastChoice = TypeOffset[next_category];
    PendingObject = Objects[LastChoice];
    
    // Update cursor
    Map.Set_Cursor_Shape(PendingObject->Occupy_List());
    Flag_To_Redraw(true);
}
```

## Editor UI Components

### Control Initialization

The editor creates specialized UI widgets at startup:

```cpp
void MapEditClass::One_Time(void)
{
    MouseClass::One_Time();
    
    // The map: a single large "button" for click detection
    MapArea = new ControlClass(MAP_AREA, 0, 8, 
                               640-8, 400-8,
                               GadgetClass::LEFTPRESS | 
                               GadgetClass::LEFTRELEASE, 
                               false);
    
    // House selection list
    HouseList = new ListClass(POPUP_HOUSELIST, 
                              POPUP_HOUSE_X, POPUP_HOUSE_Y, 
                              POPUP_HOUSE_W, POPUP_HOUSE_H,
                              TPF_EFNT|TPF_NOSHADOW,
                              MFCD::Retrieve("EBTN-UP.SHP"),
                              MFCD::Retrieve("EBTN-DN.SHP"));
    
    // Populate with all house types
    for (HousesType house = HOUSE_FIRST; 
         house < HOUSE_COUNT; house++) {
        HouseList->Add_Item(
            HouseTypeClass::As_Reference(house).IniName);
    }
    
    // Mission selection list
    MissionList = new ListClass(POPUP_MISSIONLIST,
                                POPUP_MISSION_X, POPUP_MISSION_Y, 
                                POPUP_MISSION_W, POPUP_MISSION_H,
                                TPF_EFNT|TPF_NOSHADOW,
                                MFCD::Retrieve("EBTN-UP.SHP"),
                                MFCD::Retrieve("EBTN-DN.SHP"));
    
    for (int i = 0; i < NUM_EDIT_MISSIONS; i++) {
        MissionList->Add_Item(
            MissionClass::Mission_Name(MapEditMissions[i]));
    }
    
    // Health/strength gauge
    HealthGauge = new TriColorGaugeClass(POPUP_HEALTHGAUGE,
                                         POPUP_HEALTH_X, 
                                         POPUP_HEALTH_Y, 
                                         POPUP_HEALTH_W, 
                                         POPUP_HEALTH_H);
    HealthGauge->Use_Thumb(true);
    HealthGauge->Set_Maximum(0x100);  // 256 = 100% health
    HealthGauge->Set_Red_Limit(0x3f - 1);      // < 25% = red
    HealthGauge->Set_Yellow_Limit(0x7f - 1);   // < 50% = yellow
    
    // Health percentage label
    HealthBuf[0] = 0;
    HealthText = new TextLabelClass(HealthBuf,
                                    POPUP_HEALTH_X + POPUP_HEALTH_W / 2,
                                    POPUP_HEALTH_Y + POPUP_HEALTH_H + 1,
                                    GadgetClass::Get_Color_Scheme(),
                                    TPF_CENTER | TPF_FULLSHADOW | TPF_EFNT);
    
    // Building attribute buttons
    Sellable = new TextButtonClass(POPUP_SELLABLE, 
                                   TXT_SELLABLE, 
                                   TPF_EBUTTON, 
                                   320-65, 200-25, 60);
    Rebuildable = new TextButtonClass(POPUP_REBUILDABLE, 
                                      TXT_REBUILD, 
                                      TPF_EBUTTON, 
                                      320-65, 200-15, 60);
    
    // Facing direction dial (8-way)
    FacingDial = new Dial8Class(POPUP_FACINGDIAL, 
                                POPUP_FACEBOX_X,
                                POPUP_FACEBOX_Y, 
                                POPUP_FACEBOX_W, 
                                POPUP_FACEBOX_H, 
                                (DirType)0);
    
    // Base building percentage slider
    BaseGauge = new GaugeClass(POPUP_BASEPERCENT, 
                               POPUP_BASE_X, POPUP_BASE_Y, 
                               POPUP_BASE_W, POPUP_BASE_H);
    BaseLabel = new TextLabelClass("Base:", 
                                   POPUP_BASE_X - 3, 
                                   POPUP_BASE_Y, 
                                   GadgetClass::Get_Color_Scheme(),
                                   TPF_RIGHT | TPF_NOSHADOW | TPF_EFNT);
    BaseGauge->Set_Maximum(100);
    BaseGauge->Set_Value(Scen.Percent);
}
```

### Popup Property Editor

Contextual editing for selected objects:

```cpp
void MapEditClass::Popup_Controls(void)
{
    // Get currently selected object
    if (CurrentObject.Count() == 0) return;
    
    ObjectClass * obj = CurrentObject[0];
    TechnoClass * techno = NULL;
    
    if (obj->Is_Techno()) {
        techno = (TechnoClass *)obj;
    }
    
    // Remove existing controls
    Remove_A_Button(*HouseList);
    Remove_A_Button(*MissionList);
    Remove_A_Button(*HealthGauge);
    Remove_A_Button(*HealthText);
    Remove_A_Button(*FacingDial);
    Remove_A_Button(*Sellable);
    Remove_A_Button(*Rebuildable);
    
    // Show appropriate controls based on object type
    if (techno) {
        // House selection
        Add_A_Button(*HouseList);
        HouseList->Set_Selected_Index((int)techno->Owner());
        
        // Mission selection (if capable)
        if (techno->Can_Have_Mission()) {
            Add_A_Button(*MissionList);
            MissionList->Set_Selected_Index(
                Mission_Index(techno->Mission));
        }
        
        // Health gauge
        Add_A_Button(*HealthGauge);
        Add_A_Button(*HealthText);
        HealthGauge->Set_Value(techno->Health_Ratio() * 256);
        sprintf(HealthBuf, "%d%%", techno->Health_Ratio() * 100);
        
        // Facing dial (if has facing)
        if (obj->What_Am_I() != RTTI_INFANTRY) {
            Add_A_Button(*FacingDial);
            FacingDial->Set_Direction(techno->PrimaryFacing.Current());
        }
    }
    
    // Building-specific controls
    if (obj->What_Am_I() == RTTI_BUILDING) {
        BuildingClass * building = (BuildingClass *)obj;
        
        Add_A_Button(*Sellable);
        Add_A_Button(*Rebuildable);
        
        Sellable->Turn_On(building->IsAllowedToSell);
        Rebuildable->Turn_On(building->IsToRebuild);
    }
    
    Flag_To_Redraw(true);
}
```

### Mouse Interaction

Detecting cursor movement for paint mode:

```cpp
bool MapEditClass::Mouse_Moved(void)
{
    static int old_x = -1;
    static int old_y = -1;
    
    int x = Get_Mouse_X();
    int y = Get_Mouse_Y();
    
    if (x != old_x || y != old_y) {
        old_x = x;
        old_y = y;
        return true;
    }
    return false;
}
```

## Scenario Persistence

### INI File Structure

Red Alert scenarios use INI format for all data:

```ini
[Basic]
Name=Soviet Mission 1
Intro=INTRO1
Brief=BRIEF1
Win=WIN1
Lose=LOSE1
Theme=Hell March
CarryOverMoney=0
ToCarryOver=no
ToInherit=no
TimerInherit=no
CivEvac=no
FadeInTimer=
FadeOutTimer=
FadeInBrief=yes
FadeOutBrief=yes
BridgeDestroyed=no

[Map]
Theater=TEMPERATE
X=10
Y=10
Width=50
Height=50

[Waypoints]
0=12345    ; Cell index for waypoint A
1=12346    ; Waypoint B
; ... up to WAYPT_COUNT
98=12400   ; Home cell
99=12401   ; Reinforcement point

[GOODGUY]  ; Allied house section
IQ=5
TechLevel=10
Credits=5000
Edge=North
Allies=GoodGuy,Neutral
MaxBuilding=50
MaxUnit=50
MaxInfantry=80

[BADGUY]   ; Soviet house section
IQ=5
TechLevel=10
Credits=10000
Edge=South
Allies=BadGuy
MaxBuilding=50
MaxUnit=50
MaxInfantry=80

[Units]
; Format: ID = Owner,Type,Health,Cell,Facing,Mission,Trigger
001=BadGuy,HIND,256,12345,0,Guard,None
002=GoodGuy,JEEP,256,12346,64,Hunt,None

[Infantry]
001=BadGuy,E1,256,12347,0,0,Guard,None  ; Last 0 is subcell
002=GoodGuy,E1,256,12348,0,1,Hunt,None

[Structures]
001=BadGuy,FACT,256,12349,0,Ready,None
002=GoodGuy,WEAP,256,12350,0,Ready,None

[Terrain]
; Cell = Type
12400=T01  ; Tree
12401=T05  ; Rock

[Smudge]
; Cell = Type,Unknown
12450=CR1,0  ; Crater

[Overlay]
; Cell = Type
12500=GOLD01  ; Gold ore
12501=GEM01   ; Gems

[Base]
; Prebuilt base for AI
; Node# = Type,Coordinate
000=FACT,12500
001=WEAP,12501
; ... continues

[Triggers]
; Trigger definitions
001=Trigger01,Event,Action,House

[TeamTypes]
; Team type definitions
001=Team01,House,Origin,Mission
```

### Read_INI Implementation

Loading scenario from INI file:

```cpp
void MapEditClass::Read_INI(CCINIClass & ini)
{
    // Call parent to load map basics
    MouseClass::Read_INI(ini);
    
    // Load editor-specific settings
    char const * section = "Basic";
    
    // Load scenario metadata
    ini.Get_String(section, "Name", "", 
                   Scen.Description, sizeof(Scen.Description));
    
    // Load waypoints
    section = "Waypoints";
    for (int i = 0; i < WAYPT_COUNT; i++) {
        char key[10];
        sprintf(key, "%d", i);
        
        int cell = ini.Get_Int(section, key, -1);
        if (cell != -1) {
            Scen.Waypoint[i] = cell;
            (*this)[cell].IsWaypoint = 1;
        }
    }
    
    // Load base building data
    section = "Base";
    for (int i = 0; ; i++) {
        char key[10];
        sprintf(key, "%03d", i);
        
        char buffer[128];
        if (ini.Get_String(section, key, NULL, buffer, sizeof(buffer))) {
            // Parse: Type,Coordinate
            char * type_name = strtok(buffer, ",");
            char * coord_str = strtok(NULL, ",");
            
            if (type_name && coord_str) {
                StructType type = BuildingTypeClass::From_Name(type_name);
                COORDINATE coord = atoi(coord_str);
                
                // Add to base node list
                BaseNodeClass * node = new BaseNodeClass;
                node->Type = type;
                node->Coord = coord;
                Base.Add(node);
            }
        } else {
            break;  // No more base nodes
        }
    }
    
    // Load triggers
    section = "Triggers";
    // ... trigger loading code ...
    
    // Load team types
    section = "TeamTypes";
    // ... team type loading code ...
}
```

### Write_INI Implementation

Saving scenario to INI file:

```cpp
void MapEditClass::Write_INI(CCINIClass & ini)
{
    // Call parent to save map basics
    MouseClass::Write_INI(ini);
    
    // Save scenario metadata
    char const * section = "Basic";
    ini.Put_String(section, "Name", Scen.Description);
    ini.Put_String(section, "Intro", Scen.IntroMovie ? 
                   VQType_Name(Scen.IntroMovie) : "");
    ini.Put_String(section, "Brief", Scen.BriefMovie ? 
                   VQType_Name(Scen.BriefMovie) : "");
    ini.Put_String(section, "Win", Scen.WinMovie ? 
                   VQType_Name(Scen.WinMovie) : "");
    ini.Put_String(section, "Lose", Scen.LoseMovie ? 
                   VQType_Name(Scen.LoseMovie) : "");
    ini.Put_String(section, "Theme", Theme_Name(Scen.TransitTheme));
    ini.Put_Bool(section, "ToCarryOver", Scen.IsToCarryOver);
    ini.Put_Bool(section, "ToInherit", Scen.IsToInherit);
    
    // Save waypoints
    section = "Waypoints";
    for (int i = 0; i < WAYPT_COUNT; i++) {
        if (Scen.Waypoint[i] != -1) {
            char key[10];
            sprintf(key, "%d", i);
            ini.Put_Int(section, key, Scen.Waypoint[i]);
        }
    }
    
    // Save base building data
    section = "Base";
    int count = 0;
    for (int i = 0; i < Base.Count(); i++) {
        BaseNodeClass * node = Base.Ptr(i);
        if (node) {
            char key[10];
            sprintf(key, "%03d", count++);
            
            char buffer[128];
            sprintf(buffer, "%s,%d", 
                    BuildingTypeClass::As_Reference(node->Type).IniName,
                    node->Coord);
            
            ini.Put_String(section, key, buffer);
        }
    }
    
    // Save all units
    section = "Units";
    count = 0;
    for (int i = 0; i < Units.Count(); i++) {
        UnitClass * unit = Units.Ptr(i);
        if (unit && !unit->IsInLimbo) {
            char key[10];
            sprintf(key, "%03d", count++);
            
            char buffer[256];
            sprintf(buffer, "%s,%s,%d,%d,%d,%s,%s",
                    HouseTypeClass::As_Reference(
                        unit->Owner()).IniName,
                    UnitTypeClass::As_Reference(
                        unit->Class->Type).IniName,
                    unit->Strength,
                    Coord_Cell(unit->Coord),
                    (int)unit->PrimaryFacing.Current(),
                    MissionClass::Mission_Name(unit->Mission),
                    unit->Trigger ? unit->Trigger->Get_Name() : "None");
            
            ini.Put_String(section, key, buffer);
        }
    }
    
    // Save all infantry
    section = "Infantry";
    count = 0;
    for (int i = 0; i < Infantry.Count(); i++) {
        InfantryClass * infantry = Infantry.Ptr(i);
        if (infantry && !infantry->IsInLimbo) {
            char key[10];
            sprintf(key, "%03d", count++);
            
            char buffer[256];
            sprintf(buffer, "%s,%s,%d,%d,%d,%d,%s,%s",
                    HouseTypeClass::As_Reference(
                        infantry->Owner()).IniName,
                    InfantryTypeClass::As_Reference(
                        infantry->Class->Type).IniName,
                    infantry->Strength,
                    Coord_Cell(infantry->Coord),
                    (int)infantry->PrimaryFacing.Current(),
                    infantry->SubCell,  // 0-4 for cell position
                    MissionClass::Mission_Name(infantry->Mission),
                    infantry->Trigger ? 
                        infantry->Trigger->Get_Name() : "None");
            
            ini.Put_String(section, key, buffer);
        }
    }
    
    // Save buildings
    section = "Structures";
    count = 0;
    for (int i = 0; i < Buildings.Count(); i++) {
        BuildingClass * building = Buildings.Ptr(i);
        if (building && !building->IsInLimbo) {
            char key[10];
            sprintf(key, "%03d", count++);
            
            char buffer[256];
            sprintf(buffer, "%s,%s,%d,%d,%d,%s,%s",
                    HouseTypeClass::As_Reference(
                        building->Owner()).IniName,
                    BuildingTypeClass::As_Reference(
                        building->Class->Type).IniName,
                    building->Strength,
                    Coord_Cell(building->Coord),
                    (int)building->PrimaryFacing.Current(),
                    MissionClass::Mission_Name(building->Mission),
                    building->Trigger ? 
                        building->Trigger->Get_Name() : "None");
            
            ini.Put_String(section, key, buffer);
        }
    }
    
    // Save triggers and team types
    // ... similar pattern for triggers, teams ...
}
```

### Save/Load Dialog Flow

User interface for scenario file operations:

```cpp
int MapEditClass::Load_Scenario(void)
{
    int scen_num = Scen.Scenario;
    ScenarioPlayerType player = Scen.ScenPlayer;
    ScenarioDirType dir = Scen.ScenDir;
    ScenarioVarType var = Scen.ScenVar;
    
    // Show file picker
    if (Pick_Scenario("Load Scenario", scen_num, 
                      player, dir, var) == 0) {
        
        // Build filename
        Set_Scenario_Name(scen_num, player, dir, var);
        
        // Open INI file
        CCINIClass ini;
        if (ini.Load(Scen.ScenarioName)) {
            // Clear existing scenario
            Clear_Scenario();
            
            // Load new data
            Read_INI(ini);
            
            // Reset state
            Changed = 0;
            return 0;  // Success
        }
    }
    
    return -1;  // Cancelled or failed
}

int MapEditClass::Save_Scenario(void)
{
    // Prompt for scenario number if new
    int scen_num = Scen.Scenario;
    ScenarioPlayerType player = Scen.ScenPlayer;
    ScenarioDirType dir = Scen.ScenDir;
    ScenarioVarType var = Scen.ScenVar;
    
    if (Pick_Scenario("Save Scenario", scen_num, 
                      player, dir, var) == 0) {
        
        // Build filename
        Set_Scenario_Name(scen_num, player, dir, var);
        
        // Create INI file
        CCINIClass ini;
        
        // Write all data
        Write_INI(ini);
        
        // Save to disk
        if (ini.Save(Scen.ScenarioName)) {
            Changed = 0;
            return 0;  // Success
        }
    }
    
    return -1;  // Cancelled or failed
}
```

## Waypoint System

### Waypoint Management

Waypoints mark special locations on the map:

```cpp
void MapEditClass::Update_Waypoint(int waypt_idx)
{
    if (waypt_idx < 0 || waypt_idx >= WAYPT_COUNT) return;
    if (CurrentCell == -1) return;
    
    // Check if waypoint already exists on another cell
    CELL old_cell = Scen.Waypoint[waypt_idx];
    
    if (old_cell != -1) {
        // Check if old cell has other waypoints
        bool has_other_waypoints = false;
        for (int i = 0; i < WAYPT_COUNT; i++) {
            if (i != waypt_idx && Scen.Waypoint[i] == old_cell) {
                has_other_waypoints = true;
                break;
            }
        }
        
        // Clear waypoint flag if no others
        if (!has_other_waypoints) {
            (*this)[old_cell].IsWaypoint = 0;
            Flag_Cell(old_cell);
        }
    }
    
    // Set new waypoint
    Scen.Waypoint[waypt_idx] = CurrentCell;
    (*this)[CurrentCell].IsWaypoint = 1;
    Flag_Cell(CurrentCell);
    Changed = 1;
}

bool MapEditClass::Get_Waypoint_Name(char wayptname[])
{
    // Show input dialog for two-letter waypoint name
    wayptname[0] = 0;
    
    // Simple text input (implementation varies)
    WWKeyboardClass::Get()->Get_String(wayptname, 2);
    
    return (strlen(wayptname) > 0);
}
```

### Special Waypoints

**WAYPT_HOME (98):** Player starting position
**WAYPT_REINF (99):** Reinforcement arrival point
**0-25:** Letter waypoints (A-Z) for triggers
**26+:** Extended waypoints (AA, AB, etc.)

## House Management

### House Verification

Ensuring objects can be owned by specific houses:

```cpp
bool MapEditClass::Verify_House(HousesType house, 
                                 ObjectTypeClass const * objtype)
{
    if (!objtype) return false;
    
    // Get ownership bitmap
    int ownable = objtype->Get_Ownable();
    
    // Check if this house bit is set
    return ((1 << house) & ownable) != 0;
}

HousesType MapEditClass::Cycle_House(HousesType curhouse, 
                                      ObjectTypeClass const * objtype)
{
    if (!objtype) return curhouse;
    
    // Try each house in order
    HousesType house = curhouse;
    for (int i = 0; i < HOUSE_COUNT; i++) {
        house = (HousesType)((house + 1) % HOUSE_COUNT);
        
        if (Verify_House(house, objtype)) {
            return house;
        }
    }
    
    // No valid house found
    return curhouse;
}
```

### Toggle House

Cycling ownership for placed objects:

```cpp
void MapEditClass::Toggle_House(void)
{
    if (PendingObject) {
        // Cycle to next valid house
        LastHouse = Cycle_House(LastHouse, PendingObject);
        PendingHouse = LastHouse;
        
        // Update display
        Flag_To_Redraw(true);
    }
}
```

## Editor Main Loop

### AI Processing

The core editor update loop:

```cpp
void MapEditClass::AI(KeyNumType & input, int x, int y)
{
    // Frame counter
    ::Frame++;
    
    // Mouse cursor management over map
    if (Get_Mouse_X() > TacPixelX && 
        Get_Mouse_X() < TacPixelX + Lepton_To_Pixel(TacLeptonWidth) &&
        Get_Mouse_Y() > TacPixelY && 
        Get_Mouse_Y() < TacPixelY + Lepton_To_Pixel(TacLeptonHeight)) {
        
        // Restore cursor shape when over map
        if (CurTrigger) {
            Override_Mouse_Shape(MOUSE_CAN_MOVE);
        } else {
            Override_Mouse_Shape(MOUSE_NORMAL);
        }
    }
    
    // Track cursor cell
    if (Get_Mouse_X() >= TacPixelX && 
        Get_Mouse_X() <= TacPixelX + Lepton_To_Pixel(TacLeptonWidth) &&
        Get_Mouse_Y() >= TacPixelY && 
        Get_Mouse_Y() <= TacPixelY + Lepton_To_Pixel(TacLeptonHeight)) {
        
        CELL cell = Click_Cell_Calc(Get_Mouse_X(), Get_Mouse_Y());
        if (cell != -1) {
            Set_Cursor_Pos(cell);
            if (PendingObject) {
                Flag_To_Redraw(true);
            }
        }
    }
    
    // Paint mode while dragging
    bool rc = Mouse_Moved();
    if (LMouseDown && rc) {
        if (PendingObject) {
            Flag_To_Redraw(true);
            if (Place_Object() == 0) {
                Changed = 1;
                Start_Placement();  // Continue painting
            }
        } else if (GrabbedObject) {
            GrabbedObject->Mark(MARK_CHANGE);
            if (Move_Grabbed_Object() == 0) {
                Changed = 1;
            }
        }
    }
    
    // Process keyboard shortcuts
    switch (input) {
        case KN_RMOUSE:
            // Right-click menu
            if (PendingObject) {
                if (BaseBuilding) {
                    Cancel_Base_Building();
                } else {
                    Cancel_Placement();
                }
            }
            if (CurTrigger) {
                Stop_Trigger_Placement();
            }
            if (CurrentObject.Count()) {
                CurrentObject[0]->Unselect();
                Popup_Controls();
            }
            Main_Menu();
            input = KN_NONE;
            break;
            
        case KN_INSERT:
            // Begin placement mode
            if (!PendingObject) {
                if (CurrentObject.Count()) {
                    CurrentObject[0]->Unselect();
                    Popup_Controls();
                }
                Start_Placement();
            }
            input = KN_NONE;
            break;
            
        case KN_ESC:
            // Exit placement or editor
            if (PendingObject) {
                if (BaseBuilding) {
                    Cancel_Base_Building();
                } else {
                    Cancel_Placement();
                }
            } else if (CurTrigger) {
                Stop_Trigger_Placement();
            } else {
                // Prompt to exit
                // ... exit confirmation dialog ...
            }
            input = KN_NONE;
            break;
            
        case KN_LEFT:
            if (PendingObject) Place_Prev();
            input = KN_NONE;
            break;
            
        case KN_RIGHT:
            if (PendingObject) Place_Next();
            input = KN_NONE;
            break;
            
        case KN_PGUP:
            if (PendingObject) Place_Prev_Category();
            input = KN_NONE;
            break;
            
        case KN_PGDN:
            if (PendingObject) Place_Next_Category();
            input = KN_NONE;
            break;
            
        case KN_HOME:
            if (PendingObject) {
                Place_Home();
            } else {
                // Jump to home waypoint
                ScenarioInit++;
                Set_Tactical_Position(Scen.Waypoint[WAYPT_HOME]);
                ScenarioInit--;
                HidPage.Clear();
                Flag_To_Redraw(true);
                Render();
            }
            input = KN_NONE;
            break;
            
        case ((int)KN_HOME | (int)KN_SHIFT_BIT):
            // Set new home cell
            if (CurrentCell != 0) {
                CELL cell = Scen.Waypoint[WAYPT_HOME];
                if (cell != -1) {
                    // Remove old home waypoint if unique
                    bool found = false;
                    for (int i = 0; i < WAYPT_COUNT; i++) {
                        if (i != WAYPT_HOME && 
                            Scen.Waypoint[i] == cell) {
                            found = true;
                        }
                    }
                    if (!found) {
                        (*this)[cell].IsWaypoint = 0;
                        Flag_Cell(cell);
                    }
                }
                
                // Set new home
                Scen.Waypoint[WAYPT_HOME] = CurrentCell;
                (*this)[CurrentCell].IsWaypoint = 1;
                Flag_Cell(CurrentCell);
                Changed = 1;
            }
            input = KN_NONE;
            break;
            
        case KN_H:
        case ((int)KN_H | (int)KN_SHIFT_BIT):
            if (PendingObject) Toggle_House();
            input = KN_NONE;
            break;
            
        // ALT+Letter: Set waypoint
        case ((int)KN_A | (int)KN_ALT_BIT):
        case ((int)KN_B | (int)KN_ALT_BIT):
        // ... through Z ...
        case ((int)KN_Z | (int)KN_ALT_BIT):
            if (CurrentCell != 0) {
                int waypt_idx = (input & ~KN_ALT_BIT) - KN_A;
                Update_Waypoint(waypt_idx);
            }
            input = KN_NONE;
            break;
            
        case ((int)KN_SPACE | (int)KN_ALT_BIT):
            // Remove waypoint from current cell
            if (CurrentCell != 0) {
                for (int i = 0; i < WAYPT_HOME; i++) {
                    if (Scen.Waypoint[i] == CurrentCell) {
                        Scen.Waypoint[i] = -1;
                    }
                }
                
                // Check if cell still has special waypoints
                if (Scen.Waypoint[WAYPT_HOME] != CurrentCell &&
                    Scen.Waypoint[WAYPT_REINF] != CurrentCell) {
                    (*this)[CurrentCell].IsWaypoint = 0;
                }
                Changed = 1;
                Flag_Cell(CurrentCell);
            }
            input = KN_NONE;
            break;
    }
    
    // Call parent AI for scrolling, etc.
    MouseClass::AI(input, x, y);
}
```

## File Organization

### Module Breakdown

**mapedit.cpp (2,171 lines)**
- MapEditClass constructor
- One_Time() and Init_IO() initialization
- AI() main loop with keyboard handling
- Add_To_List(), Clear_List() object management
- Mouse_Moved() detection
- Verify_House(), Cycle_House() ownership checks
- Main_Menu(), AI_Menu() menu systems

**mapeddlg.cpp (~1,500 lines)**
- New_Scenario() - Create new scenario
- Load_Scenario(), Save_Scenario() - File I/O
- Pick_Scenario() - File picker dialog
- Size_Map() - Map dimension editor
- Scenario_Dialog() - Scenario properties
- Handle_Triggers(), Select_Trigger() - Trigger editor

**mapedplc.cpp (~1,200 lines)**
- Placement_Dialog() - Object selection UI
- Start_Placement() - Begin placement mode
- Place_Object() - Core placement logic
- Cancel_Placement() - Exit placement
- Place_Next(), Place_Prev() - Object cycling
- Place_Next_Category(), Place_Prev_Category() - Category cycling
- Place_Home() - Jump to first object
- Toggle_House() - Change ownership
- Start_Trigger_Placement(), Stop_Trigger_Placement()
- Start_Base_Building(), Cancel_Base_Building()
- Build_Base_To() - Base percentage control

**mapedsel.cpp (~1,400 lines)**
- Select_Object() - Click to select
- Select_Next() - Keyboard selection
- Popup_Controls() - Show/hide property editor
- Grab_Object() - Begin drag
- Move_Grabbed_Object() - Drag logic
- Change_House() - Change selected object's owner

**mapedtm.cpp (~2,300 lines)**
- Draw_Member() - Team member rendering
- Handle_Teams() - Team editor dialog
- Select_Team() - Team selection
- Team_Members() - Count team units

### Header Definitions

**MAPEDIT.H (342 lines)**
```cpp
#ifndef MAPEDIT_H
#define MAPEDIT_H

#include "function.h"

// Maximum objects the editor can handle
enum MapEdit1Enum {
    MAX_EDIT_OBJECTS = TEMPLATE_COUNT + OVERLAY_COUNT + 
                       SMUDGE_COUNT + TERRAIN_COUNT + UNIT_COUNT +
                       INFANTRY_COUNT + VESSEL_COUNT + STRUCT_COUNT,
    
    MAX_TEAM_CLASSES = UNIT_COUNT + INFANTRY_COUNT + AIRCRAFT_COUNT,
    
    NUM_EDIT_CLASSES = 9,  // Different object categories
    
    MAX_MAIN_MENU_NUM = 8,
    MAX_MAIN_MENU_LEN = 20,
    
    MAX_AI_MENU_NUM = 6,
    MAX_AI_MENU_LEN = 20,
    
    // UI layout constants
    POPUP_HOUSE_X = 10,
    POPUP_HOUSE_Y = 100,
    POPUP_HOUSE_W = 60,
    POPUP_HOUSE_H = 90,
    
    POPUP_MISSION_W = 80,
    POPUP_MISSION_H = 40,
    POPUP_MISSION_X = 70,
    POPUP_MISSION_Y = 150,
    
    POPUP_FACEBOX_W = 30,
    POPUP_FACEBOX_H = 30,
    POPUP_FACEBOX_X = 160,
    POPUP_FACEBOX_Y = 160,
    
    POPUP_HEALTH_W = 50,
    POPUP_HEALTH_H = 10,
    POPUP_HEALTH_X = 200,
    POPUP_HEALTH_Y = 170,
    
    POPUP_BASE_W = 50,
    POPUP_BASE_H = 8,
    POPUP_BASE_X = 250,
    POPUP_BASE_Y = 0
};

// Button IDs for UI controls
enum MapEditButtonIDEnum {
    POPUP_SPAIN = 500,
    POPUP_FIRST = POPUP_SPAIN,
    POPUP_GREECE,
    POPUP_USSR,
    POPUP_ENGLAND,
    POPUP_ITALY,
    POPUP_GERMANY,
    POPUP_FRANCE,
    POPUP_TURKEY,
    POPUP_HOUSELIST,
    POPUP_SELLABLE,
    POPUP_REBUILDABLE,
    POPUP_MISSIONLIST,
    POPUP_HEALTHGAUGE,
    POPUP_FACINGDIAL,
    POPUP_BASEPERCENT,
    MAP_AREA,
    BUTTON_FLAG = 0x8000
};

#endif  // MAPEDIT_H
```

## Conclusion

The Red Alert scenario editor represents a sophisticated tool integrated directly into the game engine through conditional compilation. Its comprehensive feature set includes:

**Technical Achievements:**
- Seamless editor/game mode switching without restart
- Zero performance overhead in release builds
- Complete scenario authoring within the game engine
- Real-time validation and preview

**Design Patterns:**
- Conditional compilation for feature gating
- Object-oriented placement system with type safety
- INI-based persistence for human readability
- Modular code organization across 5 specialized files

**User Experience:**
- Intuitive keyboard shortcuts for rapid placement
- Paint mode for efficient bulk operations
- Context-sensitive property editor
- Visual feedback with cursor shapes and overlays

The editor's architecture demonstrates how game development tools can be integrated into the runtime environment while maintaining clean separation through compile-time flags. This approach enabled Westwood to create scenarios using the same code that would execute them, ensuring perfect fidelity between design and gameplay.

The system's influence can be seen in modern level editors like Unreal Engine's Blueprint editor and Unity's scene view, which similarly integrate authoring tools directly into the runtime environment for real-time feedback and iteration.
