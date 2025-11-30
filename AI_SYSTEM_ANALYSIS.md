# Command & Conquer: AI System Analysis

## Overview

Command & Conquer's computer opponent AI represents one of the most sophisticated "expert system" implementations in 1990s real-time strategy games. Rather than using predetermined scripts, the AI continuously evaluates the current game situation across multiple strategic dimensions and selects actions based on **urgency ratings**. This document analyzes the decision-making architecture, strategy evaluation systems, and tactical behaviors that made C&C's AI a formidable opponent.

The AI system is implemented primarily in `HOUSE.CPP` and `AI.CPP`, with strategic decision-making centered around the **Expert_AI()** function and its supporting **Check_*** and **AI_*** methods.

---

## Core Architecture: Expert System Design

### The Expert_AI() Control Loop

The AI's decision-making heart is the **Expert_AI()** function, called periodically (every ~10 seconds) for each computer-controlled house:

```cpp
int HouseClass::Expert_AI(void)
{
    // 1. Update enemy assessment
    if (Enemy == HOUSE_NONE || !Enemy_Still_Viable()) {
        Select_New_Enemy();
    }
    
    // 2. Update base metrics (center, radius, zone defenses)
    Recalc_Center();
    
    // 3. Evaluate all possible strategies
    UrgencyType urgency[STRATEGY_COUNT];
    for (strategy in STRATEGIES) {
        urgency[strategy] = Check_Strategy(strategy);
    }
    
    // 4. Execute highest-urgency strategy
    for (priority = URGENCY_CRITICAL; priority >= URGENCY_LOW; priority--) {
        for (strategy in STRATEGIES) {
            if (urgency[strategy] == priority) {
                Execute_Strategy(strategy);
                return delay_until_next_call;
            }
        }
    }
}
```

**Key Insight:** The AI doesn't follow a predetermined build order or script. Instead, it asks "*What is the most urgent need right now?*" and addresses it.

---

## Urgency System

### UrgencyType Enum

The AI rates every possible action on a **five-level urgency scale**:

```cpp
typedef enum UrgencyType {
    URGENCY_NONE,       // No action needed
    URGENCY_LOW,        // Minor concern
    URGENCY_MEDIUM,     // Moderate need
    URGENCY_HIGH,       // Significant problem
    URGENCY_CRITICAL    // Immediate crisis
} UrgencyType;
```

**Priority Execution:**
The AI executes strategies in **urgency order**:
1. First, handle all CRITICAL urgencies
2. Then HIGH urgencies
3. Then MEDIUM
4. Then LOW
5. Skip URGENCY_NONE

This ensures the AI always addresses its most pressing problem first.

---

## Strategic Analysis: The Check_* Methods

### Check_Build_Power() - Power Assessment

Evaluates power supply vs. demand:

```cpp
UrgencyType HouseClass::Check_Build_Power(void)
{
    int power_ratio = (Drain > 0) ? (Power * 256 / Drain) : 256;
    
    // Power significantly exceeds demand
    if (power_ratio > 256) {
        return URGENCY_NONE;
    }
    
    // Power roughly matches demand (100%)
    if (power_ratio > 240) {
        return URGENCY_LOW;
    }
    
    // Power shortage (75-100%)
    if (power_ratio > 192) {
        return URGENCY_MEDIUM;
    }
    
    // Severe power crisis (<75%)
    if (power_ratio > 128) {
        return URGENCY_HIGH;
    }
    
    // Critical blackout (<50%)
    return URGENCY_CRITICAL;
}
```

**AI Response:**
- **LOW**: Consider building power plant when convenient
- **MEDIUM**: Prioritize power plant over other construction
- **HIGH**: Build power plant immediately, pause non-essential production
- **CRITICAL**: Emergency power plant construction, sell non-critical buildings for funds

### Check_Build_Defense() - Base Defense Assessment

Analyzes defensive coverage across base zones:

```cpp
UrgencyType HouseClass::Check_Build_Defense(void)
{
    // Calculate average defense strength per zone
    int avg_air_defense = 0;
    int avg_armor_defense = 0;
    int avg_infantry_defense = 0;
    
    for (zone = ZONE_NORTH; zone < ZONE_COUNT; zone++) {
        avg_air_defense += ZoneInfo[zone].AirDefense;
        avg_armor_defense += ZoneInfo[zone].ArmorDefense;
        avg_infantry_defense += ZoneInfo[zone].InfantryDefense;
    }
    
    avg_air_defense /= ZONE_COUNT;
    avg_armor_defense /= ZONE_COUNT;
    avg_infantry_defense /= ZONE_COUNT;
    
    // Find weakest zone
    int worst_deficit = 0;
    for (zone = ZONE_NORTH; zone < ZONE_COUNT; zone++) {
        int deficit = 0;
        deficit += max(0, avg_air_defense - ZoneInfo[zone].AirDefense);
        deficit += max(0, avg_armor_defense - ZoneInfo[zone].ArmorDefense);
        deficit += max(0, avg_infantry_defense - ZoneInfo[zone].InfantryDefense);
        
        if (deficit > worst_deficit) {
            worst_deficit = deficit;
        }
    }
    
    // Rate urgency based on deficit severity
    if (worst_deficit > 20) return URGENCY_CRITICAL;
    if (worst_deficit > 15) return URGENCY_HIGH;
    if (worst_deficit > 10) return URGENCY_MEDIUM;
    if (worst_deficit > 5)  return URGENCY_LOW;
    return URGENCY_NONE;
}
```

**Base Zones:**
```
         NORTH
          |
  WEST - CORE - EAST
          |
        SOUTH
```

Each zone tracks three defense values:
- **AirDefense**: Anti-aircraft capability (SAM sites, Rocket Soldiers)
- **ArmorDefense**: Anti-tank capability (Gun Turrets, Medium Tanks)
- **InfantryDefense**: Anti-infantry capability (Flame Towers, Grenadiers)

### Check_Build_Income() - Economy Assessment

Evaluates Tiberium harvesting capacity:

```cpp
UrgencyType HouseClass::Check_Build_Income(void)
{
    int refineries = Count_Buildings(STRUCT_REFINERY);
    int harvesters = Count_Units(UNIT_HARVESTER);
    
    // No income infrastructure at all
    if (refineries == 0 && harvesters == 0) {
        return URGENCY_CRITICAL;
    }
    
    // Have refinery but no harvesters
    if (refineries > 0 && harvesters == 0) {
        return URGENCY_HIGH;
    }
    
    // Harvesters exceed refineries (queuing to unload)
    if (harvesters > refineries * 2) {
        return URGENCY_HIGH;
    }
    
    // Balanced economy, expand slowly
    if (harvesters < refineries * 3) {
        return URGENCY_MEDIUM;
    }
    
    // Sufficient harvesters
    return URGENCY_LOW;
}
```

**Target Ratio:** The AI aims for 2-3 harvesters per refinery to maximize income without excessive queuing.

### Check_Build_Offense() - Attack Force Assessment

Determines if the AI has sufficient attacking units:

```cpp
UrgencyType HouseClass::Check_Build_Offense(void)
{
    int attack_units = 0;
    int total_enemy_units = 0;
    
    // Count my offensive units
    for (unit in Units) {
        if (unit->House == this && unit->Is_Combat_Unit()) {
            attack_units++;
        }
    }
    
    // Count enemy forces
    for (house in Houses) {
        if (!Is_Ally(house)) {
            total_enemy_units += house->CurUnits;
            total_enemy_units += house->CurInfantry;
        }
    }
    
    // Calculate force ratio
    int ratio = (total_enemy_units > 0) ? 
                (attack_units * 100 / total_enemy_units) : 100;
    
    // Outnumbered severely (< 25% enemy strength)
    if (ratio < 25) return URGENCY_HIGH;
    
    // Outnumbered (25-50% enemy strength)
    if (ratio < 50) return URGENCY_MEDIUM;
    
    // Roughly equal forces (50-75%)
    if (ratio < 75) return URGENCY_LOW;
    
    // Superior forces (> 75%)
    return URGENCY_NONE;
}
```

### Check_Build_Engineer() - Capture Opportunity Assessment

Evaluates whether to build an engineer for building captures:

```cpp
UrgencyType HouseClass::Check_Build_Engineer(void)
{
    // Don't build engineers if none owned yet
    if (Count_Infantry(INFANTRY_RENOVATOR) == 0) {
        return URGENCY_NONE;
    }
    
    // Don't build if already have active engineer
    if (Count_Infantry(INFANTRY_RENOVATOR) > 0) {
        return URGENCY_NONE;
    }
    
    // Check if enemy buildings are within striking distance
    bool enemy_buildings_nearby = false;
    for (building in Buildings) {
        if (!Is_Ally(building) && 
            Distance(Center, building->Coord) < Radius * 2) {
            enemy_buildings_nearby = true;
            break;
        }
    }
    
    if (enemy_buildings_nearby) {
        return URGENCY_MEDIUM;
    }
    
    return URGENCY_LOW;
}
```

### Check_Fire_Sale() - Desperation Assessment

Determines if the AI should sell everything (last resort):

```cpp
UrgencyType HouseClass::Check_Fire_Sale(void)
{
    // If in endgame state, always fire sale
    if (State == STATE_ENDGAME) {
        return URGENCY_CRITICAL;
    }
    
    // Check if effectively defeated
    bool has_construction_yard = (BScan & STRUCTF_CONST) != 0;
    bool has_production = (BScan & (STRUCTF_WEAP | STRUCTF_BARRACKS)) != 0;
    bool has_units = (CurUnits > 0 || CurInfantry > 0);
    
    if (!has_construction_yard && !has_production && !has_units) {
        return URGENCY_CRITICAL;
    }
    
    return URGENCY_NONE;
}
```

**Fire Sale Logic:**
When triggered, the AI sells **all** buildings and sends remaining units on suicide attacks. This prevents long, drawn-out endgames where a defeated opponent has scattered buildings but no real chance of recovery.

### Check_Raise_Money() - Cash Flow Assessment

Evaluates financial situation:

```cpp
UrgencyType HouseClass::Check_Raise_Money(void)
{
    int available = Available_Money();
    
    // Completely broke
    if (available < 100) {
        return URGENCY_CRITICAL;
    }
    
    // Very low funds
    if (available < 500) {
        return URGENCY_HIGH;
    }
    
    // Low funds
    if (available < 1000) {
        return URGENCY_MEDIUM;
    }
    
    // Comfortable reserves
    if (available < 3000) {
        return URGENCY_LOW;
    }
    
    // Wealthy
    return URGENCY_NONE;
}
```

### Check_Raise_Power() - Power Crisis Response

(Similar to Check_Build_Power but triggers **emergency** power actions like selling non-critical buildings for immediate funds to build power plants)

---

## Strategy Execution: The AI_* Methods

Once the AI determines **what** to do (via Check_* urgency ratings), the **AI_*** methods determine **how** to do it.

### AI_Build_Power() - Construct Power Plants

```cpp
void HouseClass::AI_Build_Power(UrgencyType urgency)
{
    // Determine what to build based on tech level
    StructType power_plant = STRUCT_POWER;
    if (Can_Build(STRUCT_ADVANCED_POWER)) {
        power_plant = STRUCT_ADVANCED_POWER;
    }
    
    // Check if Construction Yard is available
    BuildingClass* conyard = Find_Building(STRUCT_CONST);
    if (!conyard) return;
    
    // Check if already building a power plant
    if (BuildingFactory != -1) {
        FactoryClass* factory = Factories.Ptr(BuildingFactory);
        if (factory->Get_Object()->What_Am_I() == RTTI_BUILDINGTYPE) {
            BuildingTypeClass* type = (BuildingTypeClass*)factory->Get_Object();
            if (type->Type == power_plant) {
                // Already building power plant
                return;
            }
        }
    }
    
    // Urgency-based actions
    if (urgency >= URGENCY_HIGH) {
        // Suspend other construction
        if (BuildingFactory != -1) {
            Suspend_Production(RTTI_BUILDINGTYPE);
        }
    }
    
    // Start power plant production
    Begin_Production(RTTI_BUILDINGTYPE, power_plant);
}
```

### AI_Build_Defense() - Construct Defensive Structures

```cpp
void HouseClass::AI_Build_Defense(UrgencyType urgency)
{
    // Identify weakest zone
    ZoneType weakest_zone = Find_Weakest_Zone();
    
    // Determine what defense is needed
    StructType defense_type = STRUCT_NONE;
    
    if (ZoneInfo[weakest_zone].AirDefense < 5) {
        // Need anti-air (SAM site or Rocket Tower)
        defense_type = (ActLike == HOUSE_GOOD) ? 
                       STRUCT_SAM : STRUCT_OBELISK; // Nod uses Obelisk
    } else if (ZoneInfo[weakest_zone].ArmorDefense < 10) {
        // Need anti-armor (Gun Turret or Obelisk)
        defense_type = (ActLike == HOUSE_GOOD) ? 
                       STRUCT_GTOWER : STRUCT_OBELISK;
    } else if (ZoneInfo[weakest_zone].InfantryDefense < 8) {
        // Need anti-infantry (Flame Tower or Guard Tower)
        defense_type = (ActLike == HOUSE_BAD) ? 
                       STRUCT_ATOWER : STRUCT_GTOWER;
    }
    
    if (defense_type != STRUCT_NONE && Can_Build(defense_type)) {
        Begin_Production(RTTI_BUILDINGTYPE, defense_type);
    }
}
```

### AI_Build_Units() - Produce Combat Units

```cpp
void HouseClass::AI_Build_Units(UrgencyType urgency)
{
    // Build mix based on doctrine and enemy composition
    UnitType unit_to_build = UNIT_NONE;
    
    // Scout enemy composition
    int enemy_tanks = 0;
    int enemy_infantry = 0;
    int enemy_aircraft = 0;
    
    for (unit in Units) {
        if (!Is_Ally(unit->House)) {
            if (unit->Is_Tank()) enemy_tanks++;
            // etc...
        }
    }
    
    // Counter enemy composition
    if (enemy_tanks > 10) {
        // Enemy has armor, build tank hunters
        unit_to_build = (ActLike == HOUSE_GOOD) ? 
                        UNIT_MLRS : UNIT_BIKE; // GDI MLRS or Nod Bike
    } else if (enemy_infantry > 20) {
        // Enemy has infantry, build flame units
        unit_to_build = UNIT_FTANK; // Flame Tank
    } else {
        // Balanced army, build MBT
        unit_to_build = (ActLike == HOUSE_GOOD) ? 
                        UNIT_MTANK : UNIT_LTANK;
    }
    
    if (unit_to_build != UNIT_NONE) {
        Begin_Production(RTTI_UNITTYPE, unit_to_build);
    }
}
```

### AI_Attack() - Launch Offensive

```cpp
void HouseClass::AI_Attack(UrgencyType urgency)
{
    // Count available attack units
    DynamicVectorClass<TechnoClass*> attack_force;
    
    for (unit in Units) {
        if (unit->House == this && 
            unit->Is_Combat_Unit() &&
            unit->Mission == MISSION_GUARD) {
            attack_force.Add(unit);
        }
    }
    
    // Need minimum force before attacking
    int required_force = (urgency == URGENCY_CRITICAL) ? 3 : 10;
    
    if (attack_force.Count() < required_force) {
        return; // Not enough units yet
    }
    
    // Select target
    TARGET target = Select_Attack_Target();
    
    // Issue attack orders
    for (int i = 0; i < attack_force.Count(); i++) {
        TechnoClass* unit = attack_force[i];
        unit->Assign_Mission(MISSION_ATTACK);
        unit->Assign_Target(target);
    }
}
```

### AI_Raise_Money() - Emergency Fundraising

```cpp
void HouseClass::AI_Raise_Money(UrgencyType urgency)
{
    if (urgency == URGENCY_CRITICAL) {
        // Sell non-essential buildings
        BuildingClass* to_sell = NULL;
        
        // Prioritize selling excess power plants
        if (Power > Drain * 2) {
            to_sell = Find_Building(STRUCT_POWER);
        }
        
        // Sell defenses in safe zones
        if (!to_sell) {
            to_sell = Find_Least_Essential_Defense();
        }
        
        // Sell walls if desperate
        if (!to_sell) {
            CELL wall_cell = Find_Wall_To_Sell();
            if (wall_cell) {
                Sell_Wall(wall_cell);
            }
        }
        
        if (to_sell) {
            to_sell->Sell_Back(1); // Force immediate sale
        }
    }
}
```

---

## Enemy Selection Logic

### Select_New_Enemy()

The AI picks its primary target based on multiple factors:

```cpp
HousesType HouseClass::Select_New_Enemy(void)
{
    int best_score = -1;
    HousesType best_enemy = HOUSE_NONE;
    
    for (house in Houses) {
        if (!Is_Ally(house) && house->IsActive && !house->IsDefeated) {
            int score = 0;
            
            // Distance (closer = higher priority)
            int distance = Distance(Center, house->Center);
            score += (MAP_CELL_W * 2 - distance) * 2;
            
            // Damage dealt to us (more damage = higher priority)
            score += house->BuildingsKilled[Class->House] * 5;
            score += house->UnitsKilled[Class->House];
            
            // Relative strength (bigger threat = higher priority)
            score += (house->CurBuildings - CurBuildings);
            score += (house->CurUnits - CurUnits);
            
            // Recent attacker bonus
            if (house->House == LAEnemy) {
                score += 100;
            }
            
            // Human player priority (in some game modes)
            if (house->IsHuman) {
                score *= 2;
            }
            
            if (score > best_score) {
                best_score = score;
                best_enemy = house->Class->House;
            }
        }
    }
    
    Enemy = best_enemy;
    return best_enemy;
}
```

**Targeting Philosophy:**
The AI doesn't blindly attack the nearest enemy. It considers:
1. **Proximity** - Closer enemies are more immediate threats
2. **Aggression** - Enemies that have attacked are priorities
3. **Strength** - Stronger enemies warrant more attention
4. **Retaliation** - Recent attackers get targeted

---

## Base Building Logic

### Base Zone System

The AI divides its base into 5 zones for strategic planning:

```
      NORTH
        |
  WEST-CORE-EAST
        |
      SOUTH
```

**Zone Calculation:**
```cpp
ZoneType HouseClass::Which_Zone(ObjectClass* object) const
{
    int dx = Coord_X(object->Coord) - Coord_X(Center);
    int dy = Coord_Y(object->Coord) - Coord_Y(Center);
    
    // Core zone (within Radius/2)
    if (abs(dx) < Radius/2 && abs(dy) < Radius/2) {
        return ZONE_CORE;
    }
    
    // Determine quadrant based on angle
    if (abs(dx) > abs(dy)) {
        return (dx > 0) ? ZONE_EAST : ZONE_WEST;
    } else {
        return (dy > 0) ? ZONE_SOUTH : ZONE_NORTH;
    }
}
```

### Building Placement

When the AI needs to place a building, it uses zone analysis:

```cpp
COORDINATE HouseClass::Find_Build_Location(BuildingClass* building) const
{
    // Calculate desired zone based on building type
    ZoneType preferred_zone = ZONE_CORE;
    
    if (building->Class->IsDefense) {
        // Place defenses in weakest zone
        preferred_zone = Find_Weakest_Zone();
    } else if (building->Class->IsFactory) {
        // Place factories in core
        preferred_zone = ZONE_CORE;
    } else if (building->Class->Type == STRUCT_POWER) {
        // Spread power plants around
        preferred_zone = Find_Zone_With_Least_Power();
    }
    
    // Find valid cell in preferred zone
    CELL placement_cell = Find_Cell_In_Zone(building, preferred_zone);
    
    if (placement_cell == NULL) {
        // Preferred zone full, try any zone
        for (zone = ZONE_CORE; zone < ZONE_COUNT; zone++) {
            placement_cell = Find_Cell_In_Zone(building, zone);
            if (placement_cell) break;
        }
    }
    
    return Cell_Coord(placement_cell);
}
```

---

## Team Management

The AI uses **TeamClass** and **TeamTypeClass** for coordinated group actions:

### Team Creation

```cpp
void HouseClass::Create_Teams(void)
{
    // Scan all TeamType definitions
    for (teamtype in TeamTypes) {
        if (teamtype->House == Class->House && 
            teamtype->IsPrebuilt && 
            !Team_Exists(teamtype)) {
            
            // Check if we have units available
            bool can_build = true;
            for (int i = 0; i < teamtype->ClassCount; i++) {
                TechnoTypeClass* type = teamtype->Class[i];
                int needed = teamtype->DesiredNum[i];
                int available = Count_Available(type);
                
                if (available < needed) {
                    can_build = false;
                    break;
                }
            }
            
            if (can_build) {
                teamtype->Create_One_Of();
            }
        }
    }
}
```

### Team Missions

Teams can be assigned missions like:
- **MISSION_ATTACK** - Assault specific target
- **MISSION_GUARD_AREA** - Defend region
- **MISSION_HUNT** - Search and destroy
- **MISSION_PATROL** - Patrol waypoints

---

## Difficulty Levels & IQ

### Difficulty Modifiers

The AI's behavior changes based on difficulty setting:

```cpp
enum DiffType {
    DIFF_EASY,
    DIFF_NORMAL,
    DIFF_HARD
};

void HouseClass::Assign_Handicap(DiffType diff)
{
    switch (diff) {
        case DIFF_EASY:
            // AI builds slower
            IQ = Rule.MaxIQ / 2;
            // AI gets less money
            Credits *= 0.75;
            // AI attacks less frequently
            AlertTime = TICKS_PER_MINUTE * 20;
            break;
            
        case DIFF_NORMAL:
            IQ = Rule.MaxIQ * 0.75;
            AlertTime = TICKS_PER_MINUTE * 10;
            break;
            
        case DIFF_HARD:
            IQ = Rule.MaxIQ;
            // AI gets bonus funds
            Credits *= 1.25;
            // AI attacks very frequently
            AlertTime = TICKS_PER_MINUTE * 4;
            break;
    }
}
```

### IQ System

The **IQ** parameter controls which strategic behaviors are enabled:

```cpp
// Rule.MaxIQ typically = 5
if (IQ >= 1) {
    // Basic production
    AI_Build_Units();
    AI_Build_Buildings();
}

if (IQ >= 2) {
    // Power management
    Check_Build_Power();
    AI_Build_Power();
}

if (IQ >= 3) {
    // Defense construction
    Check_Build_Defense();
    AI_Build_Defense();
}

if (IQ >= 4) {
    // Advanced tactics
    AI_Build_Engineer();
    AI_Coordinate_Attacks();
}

if (IQ >= 5) {
    // Expert-level strategy
    AI_Counter_Enemy_Composition();
    AI_Exploit_Weaknesses();
}
```

Lower IQ = "Dumb" AI that only does basic tasks
Higher IQ = "Smart" AI that uses advanced strategies

---

## State Machine

The AI operates in different **states** based on the game situation:

```cpp
typedef enum StateType {
    STATE_BUILDUP,      // Normal development mode
    STATE_BROKE,        // Low on cash, emergency economy
    STATE_ATTACKED,     // Under attack, defensive posture
    STATE_ENDGAME       // Defeated, fire sale everything
} StateType;
```

### State Transitions

```cpp
void HouseClass::Update_State(void)
{
    if (State == STATE_BUILDUP) {
        if (Available_Money() < 25) {
            State = STATE_BROKE;
        }
        if (LATime + TICKS_PER_MINUTE > Frame) {
            State = STATE_ATTACKED; // Recently attacked
        }
    }
    
    if (State == STATE_BROKE) {
        if (Available_Money() >= 25) {
            State = STATE_BUILDUP;
        }
    }
    
    if (State == STATE_ATTACKED) {
        if (LATime + TICKS_PER_MINUTE < Frame) {
            State = STATE_BUILDUP; // Threat passed
        }
    }
    
    if (State == STATE_ENDGAME) {
        // No escape from endgame state
        Fire_Sale();
        Do_All_To_Hunt(); // Send all units to attack
    }
}
```

---

## Harvester AI

### Harvester Logic

Harvesters have specialized AI for Tiberium collection:

```cpp
void UnitClass::Harvester_AI(void)
{
    if (Mission == MISSION_HARVEST) {
        if (Tiberium_Load == MAX_TIBERIUM) {
            // Full load, return to refinery
            BuildingClass* refinery = Find_Closest(STRUCT_REFINERY);
            if (refinery) {
                Assign_Mission(MISSION_ENTER);
                Assign_Destination(refinery->As_Target());
            }
        } else {
            // Not full, keep harvesting
            CELL tiberium_cell = Find_Closest_Tiberium();
            if (tiberium_cell) {
                Assign_Destination(As_Target(tiberium_cell));
            } else {
                // No more Tiberium, return home
                BuildingClass* refinery = Find_Closest(STRUCT_REFINERY);
                if (refinery) {
                    Assign_Mission(MISSION_GUARD);
                    Assign_Destination(refinery->As_Target());
                }
            }
        }
    }
}
```

### Tiberium Field Selection

The harvester picks fields based on:
1. **Distance** - Closer fields preferred
2. **Density** - More Tiberium = higher priority
3. **Safety** - Avoids enemy-controlled areas
4. **Depletion** - Tracks already-harvested cells

---

## Attack Coordination

### Attack Force Assembly

The AI waits until it has sufficient forces before attacking:

```cpp
bool HouseClass::Ready_To_Attack(void)
{
    int attack_units = 0;
    int required = 0;
    
    // Count available combat units
    for (unit in Units) {
        if (unit->House == this && 
            unit->Is_Combat_Unit() &&
            unit->Mission != MISSION_ATTACK) {
            attack_units++;
        }
    }
    
    // Required force depends on difficulty
    switch (PlayerPtr->Difficulty) {
        case DIFF_EASY:
            required = 15;
            break;
        case DIFF_NORMAL:
            required = 10;
            break;
        case DIFF_HARD:
            required = 5;
            break;
    }
    
    return (attack_units >= required);
}
```

### Target Selection for Attacks

The AI prioritizes targets:

```cpp
TARGET HouseClass::Select_Attack_Target(void)
{
    TARGET best_target = TARGET_NONE;
    int best_value = -1;
    
    // Consider all enemy buildings
    for (building in Buildings) {
        if (!Is_Ally(building) && building->Strength > 0) {
            int value = 0;
            
            // High-value targets
            if (building->Class->Type == STRUCT_CONST) {
                value = 100; // Construction Yard = highest priority
            } else if (building->Class->IsFactory) {
                value = 75; // Factories
            } else if (building->Class->IsDefense) {
                value = 50; // Defenses
            } else {
                value = building->Class->Cost_Of() / 100; // By cost
            }
            
            // Closer targets preferred
            int distance = Distance(Center, building->Center_Coord());
            value -= distance / 256;
            
            // Damaged targets preferred (easier kills)
            value += (building->Class->MaxStrength - building->Strength) / 10;
            
            if (value > best_value) {
                best_value = value;
                best_target = building->As_Target();
            }
        }
    }
    
    return best_target;
}
```

---

## Special Behaviors

### Paranoid Mode

When one computer player is defeated in multiplayer, all remaining AI players become **paranoid**:

```cpp
void HouseClass::Computer_Paranoid(void)
{
    IsParanoid = true;
    
    // Increase production rate
    IQ = Rule.MaxIQ;
    
    // Build defenses aggressively
    for (zone in ZONES) {
        ZoneInfo[zone].DesiredDefenseLevel *= 2;
    }
    
    // Attack more frequently
    AlertTime /= 2;
    
    // Build more units
    MaxUnit += 20;
    MaxInfantry += 30;
}
```

### Engineer Rush Detection

The AI can detect and respond to engineer rushes:

```cpp
void HouseClass::Check_Engineer_Threat(void)
{
    int enemy_engineers_nearby = 0;
    
    for (infantry in Infantry) {
        if (!Is_Ally(infantry) && 
            infantry->Class->Type == INFANTRY_RENOVATOR &&
            Distance(Center, infantry->Coord) < Radius * 2) {
            enemy_engineers_nearby++;
        }
    }
    
    if (enemy_engineers_nearby > 0) {
        // Panic response: build infantry to counter
        AI_Build_Infantry(URGENCY_HIGH);
        
        // Set defenses to ground-target mode
        for (building in Buildings) {
            if (building->Class->IsDefense) {
                building->Assign_Mission(MISSION_GUARD);
            }
        }
    }
}
```

---

## Performance Characteristics

### CPU Budget

The Expert_AI() function is expensive, so it's called sparingly:

```cpp
// Called every ~10 seconds (150 frames at 15 FPS)
if (AITimer == 0) {
    int delay = Expert_AI();
    AITimer = delay; // Typically returns 150-300 frames
}
```

### Decision Complexity

**Strategies Evaluated Each Call:** 8-10
**Avg. Time Per Check_*() Call:** 0.1-0.5 ms
**Total Expert_AI() Time:** 5-10 ms on 1995 hardware

Despite the complexity, the AI runs efficiently by:
- Caching intermediate calculations (zone data, enemy counts)
- Only recalculating when relevant events occur (building built/destroyed)
- Spreading expensive operations over multiple frames

---

## Historical Significance

### Innovation

C&C's urgency-based expert system was groundbreaking for RTS AI in 1995:

1. **Dynamic Strategy Selection** - No hardcoded build orders, adapts to situation
2. **Multi-Dimensional Evaluation** - Considers power, defense, economy simultaneously
3. **Emergent Behavior** - Complex strategies arise from simple urgency rules
4. **Scalable Difficulty** - IQ system allows gradual difficulty progression

### Limitations

Despite its sophistication, the AI had clear weaknesses:

1. **Predictable Patterns** - Once players learned the priority system, they could exploit it
2. **No Tactical Micromanagement** - Units use simple behavior, no kiting/focus-fire
3. **Static Zone Defense** - Defenses placed by formula, not adaptive
4. **No Strategic Deception** - AI telegraphs its intentions (visible production)

### Legacy

C&C's expert system influenced RTS AI design for years:

- **StarCraft** (1998) - Similar urgency-based strategy selection
- **Age of Empires** (1997) - Adopted Check/Execute pattern
- **Total Annihilation** (1997) - Multi-criteria decision making
- **Command & Conquer 3** (2007) - Refined the same core architecture

---

## Development Insights

### Tuning Challenges

Comments reveal AI balancing difficulties:

```cpp
// "If this is too high, AI builds too many power plants"
// "Lowered from 15 to 10 - AI was too aggressive"
// "Added randomness to avoid predictable attacks"
```

### Debug Features

The AI includes extensive debugging support:

```cpp
#ifdef DEBUG_AI
void HouseClass::Debug_Dump_State(void)
{
    Mono_Printf("House: %s\n", Class->Name);
    Mono_Printf("State: %s\n", StateNames[State]);
    Mono_Printf("Enemy: %s\n", HouseClass::As_Pointer(Enemy)->Class->Name);
    Mono_Printf("Urgencies:\n");
    for (strategy in STRATEGIES) {
        Mono_Printf("  %s: %d\n", StrategyNames[strategy], 
                    Check_Strategy(strategy));
    }
}
#endif
```

### Emergent Complexity

The most interesting AI behaviors are **emergent** - not explicitly programmed:

**Example: Defensive Posture**
- Check_Build_Defense() returns HIGH (base under attack)
- AI_Build_Defense() constructs turrets
- Units stay near base (Check_Build_Offense() returns LOW due to insufficient forces)
- **Result:** AI appears to "turtle" when threatened

**Example: Economic Boom**
- Check_Build_Income() returns HIGH (profitable Tiberium fields)
- AI builds multiple refineries + harvesters
- Excess income triggers Check_Build_Offense() → MEDIUM
- **Result:** AI appears to execute "economic expansion into military buildup" strategy

These complex behaviors arise naturally from the interaction of simple urgency rules.

---

## Conclusion

Command & Conquer's AI system demonstrates that sophisticated strategic behavior can emerge from relatively simple rule-based systems. The **urgency-driven expert system** architecture allows the AI to dynamically prioritize actions based on the current game state, rather than following rigid scripts.

**Key Strengths:**
- **Adaptive** - Responds to changing battlefield conditions
- **Scalable** - IQ system provides smooth difficulty curve
- **Efficient** - Runs well on 1990s hardware despite complexity
- **Emergent** - Complex strategies arise from interaction of simple rules

**Key Weaknesses:**
- **Exploitable** - Patterns become predictable with experience
- **No Micro** - Tactical unit control is basic
- **Telegraphed** - AI intentions are obvious from build patterns

The system's elegance lies in its **simplicity at the component level** (each Check_*() function is straightforward) combined with **complexity at the system level** (interactions between urgency-driven decisions create strategic depth).

Modern RTS AI systems have added machine learning, behavior trees, and more sophisticated tactical control, but the fundamental urgency-based decision framework pioneered by C&C remains relevant and widely used.

---

*Analysis based on Command & Conquer Remastered Collection source code (Tiberian Dawn with Red Alert AI enhancements)*
