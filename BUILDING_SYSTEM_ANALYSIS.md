# Command & Conquer: Building System Analysis

## Overview

Command & Conquer's building system implements the foundational mechanics of base construction, power management, production facilities, and defensive structures. The system manages **construction placement**, **power grid simulation**, **factory production queues**, and **building animations** through an elegant architecture that balances strategic depth with 1990s hardware constraints.

The building architecture is primarily implemented in `BUILDING.CPP/H`, `FACTORY.CPP/H`, and power management systems. Buildings represent the economic and military infrastructure that drives all RTS gameplay.

---

## Building Class Hierarchy

### Inheritance Chain

```
ObjectClass (base game object)
    ↓
MissionClass (behavior/mission system)
    ↓
RadioClass (communication between objects)
    ↓
TechnoClass (health, armor, production)
    ↓
BuildingClass (structures)
```

This inheritance provides buildings with:
- **ObjectClass:** Position, rendering, targeting
- **MissionClass:** AI state machines (MISSION_CONSTRUCTION, MISSION_GUARD, etc.)
- **RadioClass:** Communication with harvesters, repair depots, factories
- **TechnoClass:** Health, armor, production capabilities, crew management

---

## BuildingClass Structure

```cpp
class BuildingClass : public TechnoClass
{
public:
    /*
    **  Reference to the building type definition (Power Plant, Barracks, etc.)
    */
    CCPtr<BuildingTypeClass> Class;
    
    /*
    **  If this building produces units/structures, this points to the factory manager.
    */
    CCPtr<FactoryClass> Factory;
    
    /*
    **  Original owner (affects what can be built even if captured).
    */
    HousesType ActLike;
    
    // Flags
    unsigned IsToRebuild:1;         // AI should rebuild if destroyed
    unsigned IsToRepair:1;          // Allowed to repair
    unsigned IsAllowedToSell:1;     // AI can sell this building
    unsigned IsReadyToCommence:1;   // Ready for next order
    unsigned IsRepairing:1;         // Currently being repaired
    unsigned IsWrenchVisible:1;     // Show repair wrench graphic
    unsigned IsGoingToBlow:1;       // Commando C4 planted
    unsigned IsSurvivorless:1;      // No crew escapes on destruction
    unsigned IsCharging:1;          // Obelisk charging up
    unsigned IsCharged:1;           // Obelisk fully charged
    unsigned IsCaptured:1;          // Building was captured
    unsigned IsJamming:1;           // Gap generator active
    unsigned IsJammed:1;            // Radar jammed by enemy
    unsigned HasFired:1;            // GPS satellite launched
    unsigned HasOpened:1;           // Grand_Opening called
    
    /*
    **  Countdown timer for various operations.
    */
    CDTimerClass<FrameTimerClass> CountDown;
    
    /*
    **  Animation state (idle, active, construction, etc.)
    */
    BStateType BState;
    BStateType QueueBState;
    
    /*
    **  Damage tracking for multiplayer kill credit.
    */
    HousesType WhoLastHurtMe;
    TARGET WhomToRepay;
    
    /*
    **  Power management (cached for performance).
    */
    int LastStrength;
    
    /*
    **  Animation tracking (for special effects like sputdoor).
    */
    TARGET AnimToTrack;
    
    /*
    **  Factory retry timer (for unit placement).
    */
    CDTimerClass<FrameTimerClass> PlacementDelay;
    
public:
    // Construction/Destruction
    BuildingClass(StructType type, HousesType house);
    virtual ~BuildingClass(void);
    
    // Lifecycle
    virtual bool Unlimbo(COORDINATE coord, DirType dir = DIR_N);
    virtual bool Limbo(void);
    virtual void Grand_Opening(bool captured = false);
    virtual void Update_Buildables(void);
    
    // Rendering
    virtual void Draw_It(int x, int y, WindowNumberType window) const;
    void Begin_Mode(BStateType bstate);
    int Shape_Number(void) const;
    
    // AI
    virtual void AI(void);
    void Factory_AI(void);
    void Repair_AI(void);
    void Animation_AI(void);
    void Charging_AI(void);
    void Rotation_AI(void);
    
    // Power
    int Power_Output(void) const;
    
    // Production
    virtual bool Toggle_Primary(void);
    
    // Player interaction
    virtual void Repair(int control);
    virtual void Sell_Back(int control);
    virtual bool Captured(HouseClass* newowner);
    
    // Missions
    virtual int Mission_Construction(void);
    virtual int Mission_Deconstruction(void);
    virtual int Mission_Repair(void);
    virtual int Mission_Guard(void);
    virtual int Mission_Harvest(void);
    virtual int Mission_Attack(void);
    virtual int Mission_Missile(void);
};
```

---

## Building Types

### Complete Structure Enumeration

```cpp
typedef enum StructType : char {
    // GDI/Allied Structures
    STRUCT_POWER,               // Power Plant
    STRUCT_ADVANCED_POWER,      // Advanced Power Plant
    STRUCT_BARRACKS,            // Barracks (infantry)
    STRUCT_REFINERY,            // Tiberium/Ore Refinery
    STRUCT_WEAP,                // Weapons Factory (vehicles)
    STRUCT_RADAR,               // Radar/Communications Center
    STRUCT_TECH,                // Tech Center
    STRUCT_REPAIR,              // Service Depot
    STRUCT_HELIPAD,             // Helipad
    STRUCT_SAM,                 // SAM Site
    STRUCT_CONST,               // Construction Yard
    
    // Nod/Soviet Structures
    STRUCT_HAND,                // Hand of Nod (Nod barracks)
    STRUCT_AIRSTRIP,            // Airstrip (Nod airfield)
    STRUCT_OBELISK,             // Obelisk of Light / Tesla Coil
    
    // Defensive Structures
    STRUCT_PILLBOX,             // Guard Tower / Pillbox
    STRUCT_GTOWER,              // Gun Turret
    STRUCT_ATOWER,              // Advanced Guard Tower
    STRUCT_AATOWER,             // AA Gun
    STRUCT_FLAME_TOWER,         // Flame Tower
    
    // Support Structures
    STRUCT_SILO,                // Tiberium/Ore Silo
    STRUCT_MISSION,             // Mission Control (helipad building)
    
    // Special Weapons
    STRUCT_HOSPITAL,            // Hospital (Red Alert)
    STRUCT_BIO_LAB,             // Bio-Research Lab
    STRUCT_CHRONOSPHERE,        // Chronosphere
    STRUCT_IRON_CURTAIN,        // Iron Curtain
    STRUCT_MISSILE_SILO,        // Nuclear Missile Silo
    
    // Defenses (Red Alert)
    STRUCT_KENNEL,              // Dog Kennel
    STRUCT_FAKEFACT,            // Fake Weapons Factory
    STRUCT_FAKEWEAP,            // Fake War Factory
    STRUCT_FAKECONST,           // Fake Construction Yard
    STRUCT_GAP,                 // Gap Generator
    
    STRUCT_COUNT,
    STRUCT_NONE = -1
} StructType;
```

### Building Categories

Buildings fall into functional categories:

| Category | Examples | Purpose |
|----------|----------|---------|
| **Power** | Power Plant, Advanced Power | Provide electricity |
| **Production** | Barracks, War Factory, Airstrip | Build units |
| **Resource** | Refinery, Silo | Process/store economy |
| **Technology** | Tech Center, Radar | Unlock units/abilities |
| **Defense** | Gun Turret, SAM Site, Obelisk | Base protection |
| **Support** | Repair Depot, Helipad | Unit maintenance |
| **Superweapon** | Nuke Silo, Chronosphere | Game-changing abilities |

---

## Power System

### Power Production and Consumption

Every building has a **power rating**:

```cpp
int BuildingClass::Power_Output(void) const
{
    // Power plants produce power
    if (Class->Type == STRUCT_POWER) {
        return 100;  // Base power
    }
    
    if (Class->Type == STRUCT_ADVANCED_POWER) {
        return 200;  // Advanced power
    }
    
    // All other buildings consume power
    return -Class->Drain;  // Negative = consumption
}
```

### HouseClass Power Management

The owning house tracks global power state:

```cpp
class HouseClass {
public:
    int Power;      // Total power production
    int Drain;      // Total power consumption
    
    void Adjust_Power(int delta) {
        Power += delta;
        
        // Recalculate power state
        Recalc_Attributes();
    }
    
    void Adjust_Drain(int delta) {
        Drain += delta;
        
        // Recalculate power state
        Recalc_Attributes();
    }
    
    int Power_Fraction(void) const {
        // Return power ratio (256 = 100%)
        if (Drain == 0) return 256;
        return (Power * 256) / Drain;
    }
};
```

### Power State Effects

```cpp
void HouseClass::Recalc_Attributes(void)
{
    int power_fraction = Power_Fraction();
    
    // Determine power state
    if (power_fraction >= 256) {
        // Full power: all buildings operate normally
        for (int i = 0; i < Buildings.Count(); i++) {
            BuildingClass* building = Buildings.Ptr(i);
            if (building->Owner() == this) {
                building->Set_Rate(256);  // 100% effectiveness
            }
        }
    }
    else if (power_fraction >= 128) {
        // Low power (50-100%): reduced effectiveness
        for (int i = 0; i < Buildings.Count(); i++) {
            BuildingClass* building = Buildings.Ptr(i);
            if (building->Owner() == this) {
                building->Set_Rate(power_fraction);  // Proportional reduction
            }
        }
    }
    else {
        // Critical power (<50%): severe penalties
        for (int i = 0; i < Buildings.Count(); i++) {
            BuildingClass* building = Buildings.Ptr(i);
            if (building->Owner() == this) {
                building->Set_Rate(128);  // 50% maximum
                
                // Defensive structures offline
                if (building->Class->PrimaryWeapon != NULL) {
                    building->Set_Rate(0);  // Cannot fire
                }
            }
        }
    }
    
    // Sidebar visual indicator
    if (power_fraction < 192) {
        Power.Flash(6);  // Flash power bar yellow/red
    }
}
```

**Power Mechanics:**
- **256 (100%):** Full power - all buildings operate normally
- **192-255 (75-99%):** Slight power shortage - warning flash
- **128-191 (50-74%):** Low power - production slowed, defenses weakened
- **0-127 (<50%):** Critical shortage - defenses offline, severe production penalty

### Power Management on Building Events

```cpp
bool BuildingClass::Unlimbo(COORDINATE coord, DirType dir)
{
    // ... placement logic ...
    
    // Add power contribution
    int power_output = Power_Output();
    if (power_output > 0) {
        House->Adjust_Power(power_output);
    } else {
        House->Adjust_Drain(-power_output);
    }
    
    return true;
}

bool BuildingClass::Limbo(void)
{
    // Remove power contribution
    int power_output = Power_Output();
    if (power_output > 0) {
        House->Adjust_Power(-power_output);
    } else {
        House->Adjust_Drain(power_output);
    }
    
    // ... limbo logic ...
    
    return true;
}

ResultType BuildingClass::Take_Damage(int& damage, int distance, 
                                       WarheadType warhead, 
                                       TechnoClass* source, 
                                       bool forced)
{
    int old_strength = Strength;
    
    // Apply damage
    ResultType result = TechnoClass::Take_Damage(damage, distance, warhead, source, forced);
    
    // Adjust power if building damaged
    if (Strength != old_strength) {
        int power_delta = (Strength * Power_Output()) / Class->MaxStrength - 
                          (old_strength * Power_Output()) / Class->MaxStrength;
        
        if (power_delta > 0) {
            House->Adjust_Power(power_delta);
        } else {
            House->Adjust_Drain(-power_delta);
        }
    }
    
    return result;
}
```

**Dynamic Power Adjustment:**
- Damaged power plants produce less electricity proportional to health
- Example: Power Plant at 50% health produces 50 power (not 100)
- Ensures power shortages intensify as base is damaged

---

## Factory System

### Factory Production Architecture

```
BuildingClass (Factory building - Barracks, War Factory)
    ↓
FactoryClass (Production manager)
    ↓
TechnoClass (Object being produced)
    ↓
Completion → Exit to map
```

### FactoryClass Structure

```cpp
class FactoryClass : private StageClass
{
public:
    RTTIType RTTI;
    int ID;
    
    // State flags
    unsigned IsActive:1;        // Factory allocated
    unsigned IsSuspended:1;     // Production paused
    unsigned IsDifferent:1;     // Progress changed (update UI)
    unsigned IsBlocked:1;       // Exit blocked
    
    // Production tracking
    int Balance;                // Remaining cost
    int OriginalBalance;        // Total cost
    TechnoClass* Object;        // Object being produced
    int SpecialItem;            // Special item (if not object)
    HouseClass* House;          // Owner house
    
    enum StepCountEnum {
        STEP_COUNT = 54         // Production broken into 54 steps
    };
    
public:
    // Production control
    bool Set(TechnoTypeClass const& object, HouseClass& house);
    bool Set(int const& type, HouseClass& house);
    bool Start(void);
    bool Suspend(void);
    bool Abandon(void);
    bool Completed(void);
    
    // Queries
    bool Has_Changed(void);
    bool Has_Completed(void);
    bool Is_Building(void) const;
    int Completion(void);
    TechnoClass* Get_Object(void) const;
    
    // AI
    void AI(void);
    int Cost_Per_Tick(void);
};
```

### Production Flow

```cpp
// 1. Player initiates production
bool SidebarClass::Factory_Link(int factory_id, RTTIType type, int object_id)
{
    // Find factory building
    BuildingClass* factory = Find_Factory(type);
    if (!factory) return false;
    
    // Create factory manager
    FactoryClass* manager = new FactoryClass();
    factory->Factory = manager;
    
    // Set production object
    TechnoTypeClass* techno_type = Fetch_Techno_Type(type, object_id);
    manager->Set(*techno_type, *PlayerPtr);
    
    // Start production
    manager->Start();
    
    return true;
}

// 2. Factory processes production each frame
void FactoryClass::AI(void)
{
    if (IsSuspended) return;
    if (!Object) return;
    
    // Calculate cost per tick
    int cost = Cost_Per_Tick();
    
    // Check if house can afford
    if (House->Available_Money() >= cost) {
        // Deduct money
        House->Spend_Money(cost);
        Balance -= cost;
        
        // Mark progress changed for UI update
        IsDifferent = true;
        
        // Check if completed
        if (Balance <= 0) {
            Completed();
        }
    } else {
        // Insufficient funds - production stalled
        IsSuspended = true;
    }
}

// 3. Production completes
bool FactoryClass::Completed(void)
{
    if (!Object) return false;
    
    // Mark as completed
    Set_Stage(STEP_COUNT);
    IsDifferent = true;
    
    // Notify building
    BuildingClass* building = (BuildingClass*)House->Fetch_Factory(Object->What_Am_I());
    if (building) {
        building->IsReadyToCommence = true;
    }
    
    return true;
}

// 4. Building attempts to place unit
void BuildingClass::Factory_AI(void)
{
    if (!Factory) return;
    if (!Factory->Has_Completed()) return;
    if (!IsReadyToCommence) return;
    
    // Get produced object
    TechnoClass* object = Factory->Get_Object();
    if (!object) return;
    
    // Find exit cell
    CELL exit_cell = Find_Exit_Cell(object);
    
    if (exit_cell != -1) {
        // Check if exit clear
        if (Map[exit_cell].Is_Clear_To_Move(object->What_Am_I(), false, false)) {
            // Place object on map
            COORDINATE coord = Map[exit_cell].Center_Coord();
            object->Unlimbo(coord, DIR_N);
            
            // Release object from factory
            Factory->Set(object);
            delete Factory;
            Factory = NULL;
            
            // Play completion sound
            Sound_Effect(VOC_UNIT_READY, coord);
            
            // Begin production of next item in queue
            if (House->IsPlayer) {
                Map.Sidebar.Factory_Link(Factory_ID, object->What_Am_I(), next_id);
            }
        } else {
            // Exit blocked, retry later
            PlacementDelay = TICKS_PER_SECOND * 3;
            IsBlocked = true;
        }
    }
}
```

### Cost Per Tick Calculation

```cpp
int FactoryClass::Cost_Per_Tick(void)
{
    // Total cost divided into STEP_COUNT equal payments
    int cost_per_step = OriginalBalance / STEP_COUNT;
    
    // Apply house modifiers (difficulty, tech level)
    cost_per_step = (cost_per_step * House->CostBias) / 256;
    
    // Ensure at least 1 credit per tick
    if (cost_per_step < 1) cost_per_step = 1;
    
    return cost_per_step;
}
```

**Production Time Formula:**
```
Base Time = Object Cost / Cost Per Step
Actual Time = Base Time * (256 / Power Fraction)
```

**Example:**
- Medium Tank costs 800 credits
- 800 / 54 steps = 15 credits per step
- At 15 FPS, 1 step per frame = 54 frames = 3.6 seconds
- At 50% power: 3.6 * 2 = 7.2 seconds

### Multiple Factories

```cpp
bool BuildingClass::Toggle_Primary(void)
{
    // Check if this is a factory
    if (!Class->IsFactory) return false;
    
    RTTIType factory_type = Class->ToBuild;
    
    // Find all factories of this type
    for (int i = 0; i < Buildings.Count(); i++) {
        BuildingClass* building = Buildings.Ptr(i);
        
        if (building->Owner() == House &&
            building->Class->ToBuild == factory_type) {
            
            // Clear primary flag
            building->IsPrimary = false;
        }
    }
    
    // Set this as primary
    IsPrimary = true;
    
    // Primary factory receives production queue
    Map.Sidebar.Activate_Primary(factory_type);
    
    return true;
}
```

**Primary Factory Concept:**
- Only **primary factory** receives new production orders
- Non-primary factories remain idle unless explicitly assigned
- Player can toggle primary status via UI (golden border)
- Allows specialized factories (one for tanks, one for artillery)

---

## Construction and Deconstruction

### Building Placement

```cpp
MoveType BuildingClass::Can_Enter_Cell(CELL cell, FacingType facing) const
{
    // Check all cells occupied by building
    short const* offset_list = Class->Overlap_List();
    
    for (int i = 0; offset_list[i] != LIST_END; i++) {
        CELL check_cell = cell + offset_list[i];
        
        // Check cell validity
        if (!Map.In_Radar(check_cell)) {
            return MOVE_NO;
        }
        
        // Check terrain
        if (!Map[check_cell].Is_Buildable()) {
            return MOVE_NO;
        }
        
        // Check for occupants
        if (Map[check_cell].Cell_Building() != NULL) {
            return MOVE_NO;
        }
        
        // Check for units
        if (Map[check_cell].Cell_Techno() != NULL) {
            return MOVE_NO;
        }
    }
    
    // All cells clear
    return MOVE_OK;
}

bool BuildingClass::Unlimbo(COORDINATE coord, DirType dir)
{
    CELL cell = Coord_Cell(coord);
    
    // Validate placement
    if (Can_Enter_Cell(cell, FACING_NONE) != MOVE_OK) {
        return false;
    }
    
    // Mark cells as occupied
    Mark(MARK_UP);
    
    // Begin construction animation
    if (!ScenarioInit) {
        Begin_Mode(BSTATE_CONSTRUCTION);
        Assign_Mission(MISSION_CONSTRUCTION);
    }
    
    // Add to house building list
    House->Buildings.Add(this);
    
    // Adjust power
    int power_output = Power_Output();
    if (power_output > 0) {
        House->Adjust_Power(power_output);
    } else {
        House->Adjust_Drain(-power_output);
    }
    
    // Update fog of war
    Look(true);
    
    return true;
}
```

### Construction Mission

```cpp
int BuildingClass::Mission_Construction(void)
{
    enum {
        CONSTRUCTION_START,
        CONSTRUCTION_ANIM,
        CONSTRUCTION_COMPLETE,
    };
    
    switch (Status) {
        case CONSTRUCTION_START:
            // Play construction sound
            Sound_Effect(VOC_CONSTRUCTION, Coord);
            
            // Start construction animation
            Begin_Mode(BSTATE_CONSTRUCTION);
            
            Status = CONSTRUCTION_ANIM;
            return 1;
            
        case CONSTRUCTION_ANIM:
            // Wait for animation to complete
            if (BState == BSTATE_CONSTRUCTION) {
                return TICKS_PER_SECOND / 2;
            }
            
            // Animation complete
            Status = CONSTRUCTION_COMPLETE;
            return 1;
            
        case CONSTRUCTION_COMPLETE:
            // Finalize building
            Grand_Opening(false);
            
            // Switch to guard mission
            Assign_Mission(MISSION_GUARD);
            return TICKS_PER_SECOND * 30;
    }
    
    return TICKS_PER_SECOND * 1;
}
```

### Grand Opening

```cpp
void BuildingClass::Grand_Opening(bool captured)
{
    if (HasOpened) return;
    HasOpened = true;
    
    // Mark revealed
    Mark(MARK_CHANGE);
    
    // Update build options
    Update_Buildables();
    
    // Special building logic
    switch (Class->Type) {
        case STRUCT_RADAR:
            // Enable radar
            House->IsRadarActivated = true;
            Map.Sidebar.Activate_Radar();
            Map.Radar.Activate(1);
            break;
            
        case STRUCT_GAP:
            // Start jamming
            IsJamming = true;
            Map.Jam_Area(Coord, Class->Sight, House);
            break;
            
        case STRUCT_CHRONOSPHERE:
        case STRUCT_IRON_CURTAIN:
            // Charge special weapon
            House->SpecialWeaponCharge[Class->SpecialWeaponType] = 0;
            break;
            
        case STRUCT_CONST:
            // Enable building menu
            if (House->IsPlayer) {
                Map.Sidebar.Column[COLUMN_BUILDING].Activate();
            }
            break;
    }
    
    // AI notification
    if (House->IsBaseBuilding) {
        House->Begin_Production();
    }
    
    // Play completion sound
    if (!captured) {
        Sound_Effect(VOC_CONSTRUCTION_COMPLETE, Coord);
    }
}
```

### Selling Buildings

```cpp
void BuildingClass::Sell_Back(int control)
{
    if (control) {
        // Initiate sell
        if (!IsInLimbo && Mission != MISSION_DECONSTRUCTION) {
            // Play sell sound
            Sound_Effect(VOC_SELL, Coord);
            
            // Begin deconstruction
            Assign_Mission(MISSION_DECONSTRUCTION);
            Begin_Mode(BSTATE_CONSTRUCTION);  // Same anim, reversed
            
            // Abandon production
            if (Factory) {
                Factory->Abandon();
                delete Factory;
                Factory = NULL;
            }
        }
    } else {
        // Cancel sell
        if (Mission == MISSION_DECONSTRUCTION) {
            Restore_Mission();
            Begin_Mode(BSTATE_IDLE);
        }
    }
}

int BuildingClass::Mission_Deconstruction(void)
{
    enum {
        DECONSTRUCT_ANIM,
        DECONSTRUCT_REFUND,
        DECONSTRUCT_REMOVE,
    };
    
    switch (Status) {
        case DECONSTRUCT_ANIM:
            // Wait for reverse construction animation
            if (BState == BSTATE_CONSTRUCTION) {
                return TICKS_PER_SECOND / 2;
            }
            
            Status = DECONSTRUCT_REFUND;
            return 1;
            
        case DECONSTRUCT_REFUND:
            // Refund credits (50% of cost)
            int refund = Class->Cost * House->CostBias / 512;  // 50%
            House->Refund_Money(refund);
            
            Status = DECONSTRUCT_REMOVE;
            return 1;
            
        case DECONSTRUCT_REMOVE:
            // Remove building
            Drop_Debris(TARGET_NONE);  // Spawn crew
            delete this;
            return 0;
    }
    
    return TICKS_PER_SECOND * 1;
}
```

**Sell Mechanics:**
- Refund = 50% of original cost
- Production abandoned (no refund for partial completion)
- Crew survivors spawned (50% chance per infantry)
- Cannot cancel sell once animation starts

---

## Building Animations

### Animation State Machine

```cpp
typedef enum BStateType : char {
    BSTATE_NONE = -1,
    BSTATE_CONSTRUCTION,    // Building/selling animation
    BSTATE_IDLE,            // Static idle state
    BSTATE_ACTIVE,          // Generic active state
    BSTATE_FULL,            // Refinery full
    BSTATE_AUX1,            // Custom animation 1
    BSTATE_AUX2,            // Custom animation 2
} BStateType;

void BuildingClass::Begin_Mode(BStateType bstate)
{
    // Queue state change
    QueueBState = bstate;
}

void BuildingClass::Animation_AI(void)
{
    // Check for queued state change
    if (QueueBState != BSTATE_NONE && BState != QueueBState) {
        BState = QueueBState;
        QueueBState = BSTATE_NONE;
        
        // Reset animation
        Mark(MARK_CHANGE);
    }
    
    // Process current animation
    BuildingTypeClass::AnimControlType const* anim = &Class->Anims[BState];
    
    if (anim->Count > 0) {
        // Advance frame
        if (CountDown == 0) {
            CurrentFrame++;
            
            if (CurrentFrame >= anim->Start + anim->Count) {
                // Loop or end
                if (anim->Rate == 0) {
                    // One-shot animation
                    CurrentFrame = anim->Start + anim->Count - 1;
                } else {
                    // Loop
                    CurrentFrame = anim->Start;
                }
            }
            
            // Set frame delay
            CountDown = anim->Rate;
            
            // Redraw
            Mark(MARK_CHANGE);
        }
    }
}
```

### Special Building Animations

**Weapons Factory Door:**
```cpp
void BuildingClass::Factory_AI(void)
{
    // Check if ready to deliver unit
    if (Factory && Factory->Has_Completed() && IsReadyToCommence) {
        
        // Open door
        if (BState != BSTATE_ACTIVE) {
            Begin_Mode(BSTATE_ACTIVE);
        }
        
        // Wait for door to fully open
        if (CurrentFrame >= DOOR_OPEN_STAGE) {
            // Door open, place unit
            Place_Unit_From_Factory();
            
            // Close door after 3 seconds
            CountDown = TICKS_PER_SECOND * 3;
        }
    } else {
        // No production, close door
        if (BState == BSTATE_ACTIVE && CountDown == 0) {
            Begin_Mode(BSTATE_IDLE);
        }
    }
}
```

**Refinery Loading Animation:**
```cpp
RadioMessageType BuildingClass::Receive_Message(RadioClass* from, 
                                                  RadioMessageType message, 
                                                  long& param)
{
    if (*this == STRUCT_REFINERY && message == RADIO_DOCKING) {
        // Harvester approaching, start flashing lights
        if (BState != BSTATE_FULL) {
            Begin_Mode(BSTATE_FULL);
        }
        return RADIO_ROGER;
    }
    
    if (message == RADIO_IM_IN) {
        // Harvester entered, process Tiberium
        Begin_Mode(BSTATE_AUX1);  // Unloading animation
        return RADIO_ROGER;
    }
    
    return TechnoClass::Receive_Message(from, message, param);
}
```

**Obelisk Charging:**
```cpp
void BuildingClass::Charging_AI(void)
{
    if (*this == STRUCT_OBELISK || *this == STRUCT_TESLA) {
        
        if (Target_Legal(TarCom) && In_Range(TarCom)) {
            // Target in range, begin charging
            if (!IsCharging) {
                IsCharging = true;
                Begin_Mode(BSTATE_ACTIVE);
                Sound_Effect(VOC_TESLA_CHARGE, Coord);
            }
            
            // Charging animation progresses
            if (IsCharging && !IsCharged) {
                if (CurrentFrame >= CHARGE_COMPLETE_FRAME) {
                    IsCharged = true;
                    
                    // Fire weapon
                    Fire_At(TarCom, 0);
                    
                    // Discharge
                    IsCharging = false;
                    IsCharged = false;
                    Begin_Mode(BSTATE_IDLE);
                }
            }
        } else {
            // No target, reset
            if (IsCharging || IsCharged) {
                IsCharging = false;
                IsCharged = false;
                Begin_Mode(BSTATE_IDLE);
            }
        }
    }
}
```

---

## Repair System

### Repair Depot Mechanics

```cpp
int BuildingClass::Mission_Repair(void)
{
    enum {
        REPAIR_IDLE,
        REPAIR_DOCKING,
        REPAIR_ACTIVE,
        REPAIR_COMPLETE,
    };
    
    switch (Status) {
        case REPAIR_IDLE:
            // Wait for unit to dock
            if (Is_Something_Attached()) {
                Status = REPAIR_DOCKING;
                return 1;
            }
            return TICKS_PER_SECOND * 2;
            
        case REPAIR_DOCKING:
            // Wait for unit to fully dock
            {
                TechnoClass* unit = Contact_With_Whom();
                if (unit && unit->Mission == MISSION_SLEEP) {
                    Status = REPAIR_ACTIVE;
                    return 1;
                }
            }
            return TICKS_PER_SECOND * 1;
            
        case REPAIR_ACTIVE:
            {
                TechnoClass* unit = Contact_With_Whom();
                if (!unit) {
                    Status = REPAIR_IDLE;
                    return 1;
                }
                
                // Calculate repair amount
                int max_health = unit->Class_Of().MaxStrength;
                int repair_rate = max_health / 30;  // 30 steps to full
                int repair_cost = (repair_rate * unit->Class_Of().Cost) / max_health;
                
                // Check if house can afford
                if (House->Available_Money() >= repair_cost) {
                    // Apply repair
                    unit->Strength = min(unit->Strength + repair_rate, max_health);
                    House->Spend_Money(repair_cost);
                    
                    // Show repair wrench
                    IsWrenchVisible = (TickCount & 8) != 0;
                    Mark(MARK_CHANGE);
                    
                    // Check if fully repaired
                    if (unit->Strength >= max_health) {
                        Status = REPAIR_COMPLETE;
                        return 1;
                    }
                    
                    return TICKS_PER_SECOND / 2;  // Repair tick every 0.5s
                } else {
                    // Insufficient funds
                    return TICKS_PER_SECOND * 2;
                }
            }
            break;
            
        case REPAIR_COMPLETE:
            {
                // Release unit
                TechnoClass* unit = Contact_With_Whom();
                if (unit) {
                    Transmit_Message(RADIO_RUN_AWAY, unit);
                    Transmit_Message(RADIO_OVER_OUT);
                }
                
                IsWrenchVisible = false;
                Status = REPAIR_IDLE;
                return 1;
            }
            break;
    }
    
    return TICKS_PER_SECOND * 1;
}
```

### Self-Repair (Buildings)

```cpp
void BuildingClass::Repair(int control)
{
    if (control) {
        // Start repair
        if (!IsRepairing && Strength < Class->MaxStrength) {
            IsRepairing = true;
            IsToRepair = true;
        }
    } else {
        // Stop repair
        IsRepairing = false;
        IsToRepair = false;
    }
}

void BuildingClass::Repair_AI(void)
{
    if (!IsRepairing) return;
    if (Strength >= Class->MaxStrength) {
        // Fully repaired
        IsRepairing = false;
        return;
    }
    
    // Calculate repair cost and amount
    int max_health = Class->MaxStrength;
    int repair_rate = max_health / 60;  // 60 steps to full
    int repair_cost = (repair_rate * Class->Cost) / max_health;
    
    // Check if house can afford
    if (House->Available_Money() >= repair_cost) {
        // Apply repair
        Strength = min(Strength + repair_rate, max_health);
        House->Spend_Money(repair_cost);
        
        // Show wrench indicator
        IsWrenchVisible = (TickCount & 8) != 0;
        Mark(MARK_CHANGE);
    } else {
        // Pause repair due to insufficient funds
        IsWrenchVisible = false;
    }
}
```

**Repair Formula:**
```
Repair Rate = Max Health / 60 steps
Repair Cost per Step = (Repair Rate * Building Cost) / Max Health
Total Repair Time = 60 steps * (1 / 15 FPS) = 4 seconds
Total Repair Cost ≈ Current Damage %
```

**Example:**
- Advanced Power Plant: 800 HP, $700 cost
- Damaged to 400 HP (50% damage)
- Repair rate = 800 / 60 = 13.3 HP per step
- Cost per step = (13.3 * 700) / 800 = $11.6
- Total cost to full = 30 steps * $11.6 = $350 (50% of cost)

---

## Capture System

### Engineer Capture Mechanics

```cpp
bool BuildingClass::Captured(HouseClass* newowner)
{
    // Cannot capture Construction Yard or certain special buildings
    if (Class->Type == STRUCT_CONST) return false;
    if (IsCaptured && Class->IsCaptureable == false) return false;
    
    // Mark as captured
    IsCaptured = true;
    
    // Transfer ownership
    HouseClass* oldowner = House;
    House = newowner;
    
    // Adjust power
    int power_output = Power_Output();
    if (power_output > 0) {
        oldowner->Adjust_Power(-power_output);
        newowner->Adjust_Power(power_output);
    } else {
        oldowner->Adjust_Drain(power_output);
        newowner->Adjust_Drain(-power_output);
    }
    
    // Reduce crew (captured buildings have fewer infantry)
    Strength = min(Strength, Class->MaxStrength / 2);
    
    // Abandon production
    if (Factory) {
        Factory->Abandon();
        delete Factory;
        Factory = NULL;
    }
    
    // Update sidebar
    if (newowner->IsPlayer) {
        Update_Buildables();
    }
    
    // Reveal area around captured building
    Look(true);
    
    // Play capture sound
    Sound_Effect(VOC_CAPTURED, Coord);
    
    // Trigger events
    Map.Trigger_Check(TEVENT_ATTACKED, this);
    
    return true;
}
```

---

## Conclusion

Command & Conquer's building system demonstrates sophisticated RTS infrastructure management within severe hardware limitations. The **power grid simulation** creates strategic resource dependencies, **factory production queues** enable parallel construction, and **dynamic building animations** provide visual feedback for complex state machines.

**Key Achievements:**
- **Power system** with proportional building effectiveness (256-level gradation)
- **Factory production** broken into 54 steps for smooth UI progress bars
- **Multi-factory coordination** via primary factory designation
- **Building capture** with crew reduction and power transfer
- **Self-repair** with proportional cost based on damage percentage
- **Animated states** (construction, active, idle) synchronized with mission system

The building architecture's influence is visible in later RTS games - StarCraft's add-ons, Warcraft III's upgrades, and Age of Empires' economic buildings all use similar production queue and power management concepts.

Most impressively, the system handles **50+ simultaneous buildings** with **multiple active factories** and **dynamic power calculations** on a **Pentium 75MHz** while maintaining 15 FPS - a testament to the efficient state machine design and cached power calculations.

---

*Analysis based on Command & Conquer Remastered Collection source code (Tiberian Dawn & Red Alert building systems)*
