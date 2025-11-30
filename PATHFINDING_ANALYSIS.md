# Pathfinding & Unit Movement AI Analysis
## Command & Conquer Remastered Collection

---

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Class Hierarchy](#class-hierarchy)
4. [Pathfinding Algorithm](#pathfinding-algorithm)
5. [Movement Types](#movement-types)
6. [Path Calculation Process](#path-calculation-process)
7. [Movement Physics](#movement-physics)
8. [Collision Detection](#collision-detection)
9. [Special Cases](#special-cases)
10. [Performance Optimizations](#performance-optimizations)
11. [Known Issues & Edge Cases](#known-issues--edge-cases)
12. [Code Examples](#code-examples)

---

## Overview

The pathfinding and unit movement system in C&C is one of the most complex subsystems in the engine. It must handle:

- **Real-time pathfinding** for dozens of simultaneous units
- **Multiple terrain types** (land, water, air) with different movement rules
- **Dynamic obstacles** (buildings, units, terrain changes)
- **Formation movement** for groups of units
- **Network determinism** - identical paths must be calculated on all clients
- **Performance constraints** - must run at 60 FPS on 1990s hardware

The system uses a **modified A\* algorithm** with multiple heuristic strategies, integrated with a **physics-based movement system** and sophisticated **collision avoidance**.

---

## Architecture

### Core Components

```
FootClass (Base movement class)
    ├── InfantryClass (Soldiers, 5 positions per cell)
    ├── DriveClass (Vehicles, tracked/wheeled movement)
    │   └── UnitClass (Tanks, harvesters, APCs)
    └── AircraftClass (Helicopters, planes, special flight model)
```

### File Organization

| File | Purpose |
|------|---------|
| **FOOT.CPP/H** | Base class for all moving units, pathfinding core |
| **DRIVE.CPP/H** | Vehicle-specific movement and rotation |
| **INFANTRY.CPP/H** | Infantry movement and sub-cell positioning |
| **AIRCRAFT.CPP/H** | Aircraft flight model and movement |
| **CCMPATH.CPP** | A* pathfinding implementation |
| **MAP.CPP/H** | Terrain and cell management |
| **CELL.CPP/H** | Individual cell logic and occupancy |

---

## Class Hierarchy

### FootClass - Base Movement Class

**Key Responsibilities:**
- Path calculation (`Basic_Path()`)
- Navigation computer (`NavCom`)
- Movement mission states
- Pathfinding delay timers
- Path storage

**Critical Members:**
```cpp
class FootClass : public TechnoClass {
    FacingType Path[CONQUER_PATH_MAX];  // Stored path directions
    TARGET NavCom;                       // Navigation destination
    int PathDelay;                       // Time before recalculating
    bool IsPlanningToLook;              // Planning to scan ahead
    // ...
};
```

### DriveClass - Ground Vehicle Movement

**Key Responsibilities:**
- Track-based rotation
- Speed management
- Terrain interaction
- Turret vs body facing
- Path following

**Critical Members:**
```cpp
class DriveClass : public FootClass {
    int TrackNumber;        // Current track segment
    int TrackIndex;         // Position within track
    bool IsNewNavCom;       // Just received new destination
    MPHType SpeedAccum;     // Speed accumulator for sub-cell movement
    // ...
};
```

### InfantryClass - Soldier Movement

**Key Features:**
- **5 sub-cell positions** per map cell
- Independent movement for each soldier
- Formation maintenance
- Prone/standing state affects movement
- Can occupy cells with other infantry

---

## Pathfinding Algorithm

### A* Implementation

The pathfinding uses **A\* with multiple cost heuristics**:

```cpp
PathType* Find_Path(
    CELL source,           // Starting cell
    CELL dest,            // Destination cell
    FacingType* path,     // Output path buffer
    int maxlen,           // Max path length
    MoveType move         // Movement type/difficulty
);
```

### Cost Calculation

**Total Cost = G + H**

- **G (Actual Cost)**: Distance traveled so far from start
- **H (Heuristic)**: Estimated distance to goal (Manhattan distance)

**Cost Modifiers:**
```cpp
int cost = base_cost;
cost += terrain_difficulty;      // Rough terrain costs more
cost += threat_assessment;       // Avoid enemy fire zones
cost += congestion_penalty;      // Avoid crowded areas
cost += turn_penalty;            // Turning costs movement
```

### Open/Closed Lists

Classic A* implementation with optimization:

```cpp
// Open list: Cells to be evaluated (priority queue)
// Closed list: Already evaluated cells

while (!openlist.empty()) {
    current = openlist.pop_min_cost();
    
    if (current == destination) {
        reconstruct_path();
        return path;
    }
    
    closedlist.add(current);
    
    for (neighbor in current.neighbors()) {
        if (closedlist.contains(neighbor)) continue;
        if (!can_enter(neighbor)) continue;
        
        tentative_g = current.g + cost(current, neighbor);
        
        if (!openlist.contains(neighbor) || tentative_g < neighbor.g) {
            neighbor.parent = current;
            neighbor.g = tentative_g;
            neighbor.h = heuristic(neighbor, destination);
            neighbor.f = neighbor.g + neighbor.h;
            openlist.add(neighbor);
        }
    }
}
```

---

## Movement Types

The pathfinding system evaluates multiple movement strategies, represented by the `MoveType` enum:

### MoveType Enumeration

```cpp
typedef enum MoveType : unsigned char {
    MOVE_OK = 0,           // Cell is completely clear
    MOVE_CLOAK,            // Can move, ignore cloaked units
    MOVE_MOVING_BLOCK,     // Can move, but other units are moving through
    MOVE_DESTROYABLE,      // Can move by destroying obstacle (wall)
    MOVE_TEMP,             // Temporarily blocked, wait
    MOVE_NO                // Cannot move into this cell
} MoveType;
```

### Strategy Selection

The pathfinding tries **multiple strategies** and picks the best:

```cpp
// From FOOT.CPP - Basic_Path()

// 1. Try most aggressive pathfinding first
path = Find_Path(cell, &workpath1[0], sizeof(workpath1), maxtype);
if (path && path->Cost) {
    memcpy(&path1, path, sizeof(path1));
    found1 = true;
    
    // 2. Try easiest path (MOVE_CLOAK)
    path = Find_Path(cell, &workpath2[0], sizeof(workpath2), MOVE_CLOAK);
    if (path && path->Cost && path->Cost < max((path1.Cost + (path1.Cost/2)), 3)) {
        // Easiest path is significantly better, use it
        memcpy(&path1, path, sizeof(path1));
        memcpy(workpath1, workpath2, sizeof(workpath1));
    } else {
        // Try intermediate strategies
        for (MoveType move = MOVE_MOVING_BLOCK; move < maxtype; move++) {
            path = Find_Path(cell, &workpath2[0], sizeof(workpath2), move);
            if (path && path->Cost && path->Cost < max((path1.Cost + (path1.Cost/2)), 3)) {
                memcpy(&path1, path, sizeof(path1));
                memcpy(workpath1, workpath2, sizeof(workpath1));
            }
        }
    }
}
```

### Why Multiple Strategies?

**Scenario**: Unit needs to reach a destination

1. **MOVE_CLOAK** (easiest): Go around all obstacles
   - Path: 100 cells, Cost: 100
   
2. **MOVE_DESTROYABLE**: Destroy one wall, direct path
   - Path: 30 cells + destroy wall, Cost: 35
   
3. **MOVE_MOVING_BLOCK**: Push through moving units
   - Path: 40 cells, some delay, Cost: 50

**Decision**: Choose strategy #2 (destroy wall) as optimal

---

## Path Calculation Process

### Step-by-Step Execution

#### 1. Mission Assignment
```cpp
// Player orders unit to move
unit->Assign_Mission(MISSION_MOVE);
unit->Assign_Destination(target_cell);
```

#### 2. Path Delay Check
```cpp
if (PathDelay > 0) {
    PathDelay--;
    return;  // Wait before recalculating
}
```

**Purpose**: Prevents expensive pathfinding every frame. Default delay:
```cpp
PathDelay = Rule.PathDelay * TICKS_PER_MINUTE;  // ~1 second
```

#### 3. Basic_Path() Invocation

**From FOOT.CPP:**
```cpp
bool FootClass::Basic_Path(void)
{
    CELL destcell = ::As_Cell(NavCom);
    
    // Early exit checks
    if (!Target_Legal(NavCom)) return false;
    if (Distance(destcell) < CELL_LEPTON_W * 2) return true;
    
    // Determine maximum movement type this unit can do
    MoveType maxtype = MOVE_OK;
    if (IsCrateGoodie) maxtype = MOVE_CLOAK;
    // ... more logic to determine maxtype
    
    // Mark cell as occupied (self) before pathfinding
    Mark(MARK_UP);
    
    // Try multiple pathfinding strategies
    PathType path1;
    FacingType workpath1[MAX_PATH_LENGTH];
    FacingType workpath2[MAX_PATH_LENGTH];
    bool found1 = false;
    
    // [Strategy comparison code - see Movement Types section]
    
    if (found1) {
        Fixup_Path(&path1);
        memcpy(&Path[0], &workpath1[0], min(path->Length, (int)sizeof(Path)));
    }
    
    Mark(MARK_DOWN);  // Restore cell state
    
    PathDelay = Rule.PathDelay * TICKS_PER_MINUTE;
    return (Path[0] != FACING_NONE);
}
```

#### 4. Path Storage

The path is stored as **facing directions**:

```cpp
FacingType Path[CONQUER_PATH_MAX];  // Array of directions

// Example path:
Path[0] = FACING_N;    // North
Path[1] = FACING_NE;   // Northeast
Path[2] = FACING_E;    // East
Path[3] = FACING_NONE; // End of path marker
```

**Why store directions instead of cells?**
- **Memory efficient**: 1 byte per step vs 2 bytes for cell coordinates
- **Flexible**: Can adapt to minor terrain changes
- **Simple following**: Just turn to face direction and move

---

## Movement Physics

### Movement State Machine

```cpp
enum MissionType {
    MISSION_GUARD,      // Idle, standing still
    MISSION_MOVE,       // Moving to destination
    MISSION_ATTACK,     // Moving to attack target
    MISSION_HARVEST,    // Harvester-specific movement
    MISSION_HUNT,       // Search and destroy
    // ... more mission types
};
```

### DriveClass::AI() - Core Movement Loop

**Executed every game frame:**

```cpp
void DriveClass::AI(void)
{
    FootClass::AI();  // Base class updates
    
    // 1. Rotation AI - turn toward facing direction
    if (PrimaryFacing != desired_facing) {
        Rotation_AI();
    }
    
    // 2. Movement AI - physical movement
    if (Speed > 0) {
        if (Start_Of_Move()) {
            // Just entered a new cell
            Per_Cell_Process(false);
        }
    }
    
    // 3. Track following - for curved paths
    if (TrackNumber != -1) {
        Track_AI();
    }
}
```

### Speed Calculation

**From UNIT.CPP - Set_Speed():**

```cpp
void UnitClass::Set_Speed(int speed)
{
    if (speed) {
        // Convert MPH to leptons per frame
        int maxspeed = MPH_to_Leptons(Class->MaxSpeed);
        
        // Apply terrain penalties
        CELL cell = Coord_Cell(Coord);
        LandType land = Map[cell].Land_Type();
        
        switch (land) {
            case LAND_ROAD:
                maxspeed = (maxspeed * 3) / 2;  // 150% speed on roads
                break;
            case LAND_ROUGH:
                maxspeed = (maxspeed * 3) / 4;  // 75% speed on rough terrain
                break;
            case LAND_ROCK:
                maxspeed = maxspeed / 2;         // 50% speed on rocks
                break;
        }
        
        // Apply damage penalty
        int health_percent = (Strength * 100) / Class->MaxStrength;
        if (health_percent < 50) {
            maxspeed = (maxspeed * health_percent) / 50;
        }
        
        Speed = maxspeed;
    } else {
        Speed = 0;
    }
}
```

### Physics Integration

**Position Update (every frame):**

```cpp
COORDINATE new_coord = Physics(current_coord, facing);

// Physics() calculates:
// 1. Speed vector based on facing direction
// 2. Apply speed to current coordinate
// 3. Handle sub-pixel positioning (leptons)
// 4. Check for cell boundary crossing
```

**Lepton System:**
- 1 cell = 256 leptons (sub-cell precision)
- Allows smooth movement at 60 FPS
- Essential for accurate collision detection

---

## Collision Detection

### Start_Of_Move() - Cell Entry Logic

**From DRIVE.CPP - Critical function called when entering a new cell:**

```cpp
bool DriveClass::Start_Of_Move(void)
{
    if (Path[0] == FACING_NONE) {
        if (Mission == MISSION_MOVE) {
            Enter_Idle_Mode();
        }
        return false;
    }
    
    // Get destination cell from path
    FacingType facing = Path[0];
    CELL destcell = Adjacent_Cell(Coord_Cell(Coord), facing);
    
    // Check if we can enter the cell
    MoveType move = Can_Enter_Cell(destcell, facing);
    
    switch (move) {
        case MOVE_OK:
            // Cell is clear, proceed
            break;
            
        case MOVE_CLOAK:
            // Ignore cloaked units, proceed
            break;
            
        case MOVE_MOVING_BLOCK:
            // Another unit is moving through
            if (IsNewNavCom) {
                // Just started moving, wait
                SpeedAccum = 0;
                return false;
            }
            // Otherwise push through
            break;
            
        case MOVE_DESTROYABLE:
            // Wall or destructible object in the way
            if (Mission == MISSION_MOVE) {
                // Attack the obstacle
                if (Map[destcell].Cell_Object()) {
                    Override_Mission(MISSION_ATTACK, 
                        Map[destcell].Cell_Object()->As_Target(), 
                        TARGET_NONE);
                } else if (Map[destcell].Overlay != OVERLAY_NONE && 
                           OverlayTypeClass::As_Reference(Map[destcell].Overlay).IsWall) {
                    Override_Mission(MISSION_ATTACK, 
                        ::As_Target(destcell), 
                        TARGET_NONE);
                }
                IsNewNavCom = false;
                TrackIndex = 0;
                return true;
            }
            break;
            
        case MOVE_TEMP:
            // Temporarily blocked, wait a bit
            if (TryTryAgain <= 0) {
                // Waited too long, recalculate path
                Path[0] = FACING_NONE;
                if (Mission == MISSION_MOVE) {
                    Set_Speed(0);
                }
            }
            return false;
            
        case MOVE_NO:
            // Cannot move into cell at all
            // Shift path forward and try next step
            memcpy(&Path[0], &Path[1], CONQUER_PATH_MAX - 1);
            Path[CONQUER_PATH_MAX - 1] = FACING_NONE;
            return Start_Of_Move();  // Recursive retry
    }
    
    // Start driver to physically move into cell
    IsNewNavCom = false;
    TrackIndex = 0;
    if (!Start_Driver(destcell)) {
        TrackNumber = -1;
        Path[0] = FACING_NONE;
        Set_Speed(0);
    }
    
    return false;
}
```

### Can_Enter_Cell() - Detailed Checks

**Evaluates if a unit can enter a specific cell:**

```cpp
MoveType FootClass::Can_Enter_Cell(CELL cell, FacingType facing)
{
    // 1. Bounds check
    if (!Map.In_Radar(cell)) return MOVE_NO;
    
    // 2. Check terrain type
    LandType land = Map[cell].Land_Type();
    if (land == LAND_WATER && !IsMarine) return MOVE_NO;
    if (land != LAND_WATER && IsMarine) return MOVE_NO;
    
    // 3. Check terrain objects (trees, rocks)
    if (Map[cell].Overlay != OVERLAY_NONE) {
        OverlayType overlay = Map[cell].Overlay;
        if (IsWall(overlay)) {
            // Can crush walls if we're a tank
            if (IsCrusher) return MOVE_DESTROYABLE;
            return MOVE_NO;
        }
    }
    
    // 4. Check occupying object
    ObjectClass* obj = Map[cell].Cell_Object();
    if (obj) {
        // Is it an enemy?
        if (!House->Is_Ally(obj->Owner())) {
            if (obj->Is_Techno()) {
                TechnoClass* techno = (TechnoClass*)obj;
                // Can we crush it?
                if (IsCrusher && techno->IsCrushable) {
                    return MOVE_DESTROYABLE;
                }
            }
            return MOVE_NO;
        }
        
        // Is it one of our units moving?
        if (obj->Is_Foot() && obj->Is_Active()) {
            FootClass* foot = (FootClass*)obj;
            if (foot->IsDriving) {
                return MOVE_MOVING_BLOCK;  // Might move out of the way
            }
        }
        
        // Static object in the way
        return MOVE_NO;
    }
    
    // 5. Check if cell is reserved
    if (Map[cell].Flag.Composite & CELL_RESERVED) {
        return MOVE_TEMP;  // Someone reserved it, wait
    }
    
    // 6. Check shroud (fog of war)
    if (!Map[cell].Is_Visible(House)) {
        return MOVE_CLOAK;  // Can move, but can't see threats
    }
    
    return MOVE_OK;
}
```

---

## Special Cases

### 1. Infantry Sub-Cell Positioning

Infantry can share cells using **5 sub-positions**:

```
 ┌─────┐
 │1   2│  1: Upper Left
 │  0  │  0: Center
 │3   4│  2: Upper Right
 └─────┘  3: Lower Left
          4: Lower Right
```

**InfantryClass Implementation:**

```cpp
class InfantryGroup {
    Infantry* Members[5];  // 5 soldiers per cell
    
    static Point stoppingLocations[5];
    // Calculated in static constructor:
    // [0] = (Width/2, Height/2)        // Center
    // [1] = (Width/4, Height/4)        // Upper Left
    // [2] = (3*Width/4, Height/4)      // Upper Right
    // [3] = (Width/4, 3*Height/4)      // Lower Left
    // [4] = (3*Width/4, 3*Height/4)    // Lower Right
};
```

**Movement Logic:**
```cpp
InfantryStoppingType Infantry::Find_Best_Position(CELL cell)
{
    InfantryGroup* group = Map[cell].InfantryGroup;
    
    // Try center first
    if (!group->Members[0]) return INFANTRY_CENTER;
    
    // Try corners
    for (int i = 1; i < 5; i++) {
        if (!group->Members[i]) return (InfantryStoppingType)i;
    }
    
    return INFANTRY_NONE;  // Cell is full
}
```

### 2. Aircraft Movement

Aircraft use a **completely different system**:

**No Pathfinding:**
- Aircraft move in **straight lines** to destination
- No terrain obstacles (except buildings)
- Can overlap other aircraft

**Flight Model:**

```cpp
void AircraftClass::Movement_AI(void)
{
    // No pathfinding, just physics
    if (Speed != 0) {
        if (In_Which_Layer() == LAYER_GROUND) {
            // Taking off or landing
            Mark(MARK_UP);
            Physics(Coord, PrimaryFacing);
            Mark(MARK_DOWN);
        } else {
            // Flying
            Mark(MARK_CHANGE_REDRAW);
            if (Physics(Coord, PrimaryFacing) != RESULT_NONE) {
                Mark(MARK_CHANGE_REDRAW);
            }
        }
    }
}
```

**Altitude System:**
```cpp
enum LayerType {
    LAYER_GROUND,     // On ground (landed)
    LAYER_SURFACE,    // Just above ground (tanks, infantry)
    LAYER_AIR         // Flying altitude
};
```

### 3. Bridge Traversal

**Problem**: Bridges are special terrain that can be destroyed

**Solution**: Dynamic passability checking

```cpp
bool Can_Cross_Bridge(CELL bridge_cell)
{
    // Check bridge health
    if (Map[bridge_cell].Overlay == OVERLAY_BRIDGE_DESTROYED) {
        return false;
    }
    
    // Check if it's our type of bridge
    LandType bridge_land = Map[bridge_cell].Land_Type();
    if (IsMarine && bridge_land != LAND_WATER_BRIDGE) {
        return false;  // Ships need water bridges
    }
    if (!IsMarine && bridge_land != LAND_LAND_BRIDGE) {
        return false;  // Ground units need land bridges
    }
    
    return true;
}
```

### 4. Tiberium/Ore Harvesting

Harvesters have **special pathfinding**:

```cpp
bool UnitClass::Tiberium_Check(CELL &center, int x, int y)
{
    // Scan for nearest Tiberium
    CELL cell = center;
    int radius = 0;
    
    while (radius < MAX_HARVEST_RADIUS) {
        // Spiral search pattern
        for (int angle = 0; angle < 360; angle += 45) {
            CELL check = Adj_Cell(center, angle, radius);
            
            if (Map[check].Contains_Tiberium()) {
                // Found Tiberium, calculate path
                if (Basic_Path_To(check)) {
                    center = check;
                    return true;
                }
            }
        }
        radius++;
    }
    
    return false;  // No Tiberium found
}
```

### 5. Formation Movement

**Problem**: Multiple units selected move together

**Solution**: Staggered path calculation

```cpp
void Group_Move(DynamicVectorClass<FootClass*>& units, TARGET dest)
{
    // Sort units by distance to destination
    units.Sort_By_Distance(dest);
    
    // Leader calculates path first
    units[0]->Assign_Destination(dest);
    
    // Followers offset their destinations
    for (int i = 1; i < units.Count(); i++) {
        CELL leader_dest = As_Cell(dest);
        CELL offset_dest = Apply_Formation_Offset(leader_dest, i);
        units[i]->Assign_Destination(offset_dest);
    }
    
    // Stagger pathfinding to avoid frame spike
    for (int i = 0; i < units.Count(); i++) {
        units[i]->PathDelay = i * FORMATION_DELAY;
    }
}
```

---

## Performance Optimizations

### 1. Path Caching

**Problem**: Pathfinding is expensive

**Solution**: Only recalculate when necessary

```cpp
// Delay between recalculations
PathDelay = Rule.PathDelay * TICKS_PER_MINUTE;  // ~60 frames

// Reasons to recalculate early:
if (destination_moved || 
    path_blocked ||
    mission_changed ||
    under_attack) {
    PathDelay = 0;  // Force immediate recalc
}
```

### 2. Hierarchical Pathfinding

**For long distances**, use waypoint system:

```cpp
// Break long path into segments
CELL waypoint1 = Intermediate_Point(source, dest, 0.25);
CELL waypoint2 = Intermediate_Point(source, dest, 0.50);
CELL waypoint3 = Intermediate_Point(source, dest, 0.75);

// Calculate to first waypoint only
Path* path = Find_Path(source, waypoint1, ...);
// When reached, calculate to waypoint2, etc.
```

### 3. Path Length Limiting

**From pathfinding code:**

```cpp
#define MAX_PATH_LENGTH 100  // Maximum path length

PathType* Find_Path(CELL source, CELL dest, 
                   FacingType* path, int maxlen, MoveType move)
{
    // Early exit if too far
    int straight_line = Distance(source, dest);
    if (straight_line > MAX_PATH_LENGTH) {
        // Calculate path to intermediate point
        CELL intermediate = Point_Along_Line(source, dest, MAX_PATH_LENGTH);
        return Find_Path(source, intermediate, path, maxlen, move);
    }
    
    // ... A* algorithm
}
```

### 4. Dirty Rectangle Optimization

**Only recalculate for units in active area:**

```cpp
RECT active_area = Get_Tactical_View_Rect();

for (each unit) {
    if (!unit.In_Rect(active_area)) {
        // Off-screen unit, skip pathfinding this frame
        continue;
    }
    
    if (unit.PathDelay == 0) {
        unit.Basic_Path();
    }
}
```

### 5. Pre-computed Direction Tables

**Fast facing calculations:**

```cpp
// Pre-computed lookup tables
static int DirDX[256];  // X offset for each facing
static int DirDY[256];  // Y offset for each facing

// Initialize once at startup
void Init_Direction_Tables()
{
    for (int dir = 0; dir < 256; dir++) {
        float angle = (dir * 2 * PI) / 256.0;
        DirDX[dir] = (int)(cos(angle) * 256);
        DirDY[dir] = (int)(sin(angle) * 256);
    }
}

// Use in pathfinding (no trigonometry at runtime)
COORDINATE Move_In_Direction(COORDINATE coord, FacingType dir, int distance)
{
    int x = Coord_X(coord) + ((DirDX[dir] * distance) >> 8);
    int y = Coord_Y(coord) + ((DirDY[dir] * distance) >> 8);
    return XY_Coord(x, y);
}
```

---

## Known Issues & Edge Cases

### 1. **Unit Clumping**

**Problem**: Multiple units trying to occupy same cell

**Manifestation:**
```
Units ordered to same destination
    ↓
All calculate same path
    ↓
Arrive at destination simultaneously
    ↓
Fight over final cell
    ↓
Units oscillate/block each other
```

**Mitigation**:
```cpp
// Add random jitter to final destination
CELL destination = player_order_destination;
destination = Random_Adjacent_Cell(destination, unit->ID % 8);
```

### 2. **Path Thrashing**

**Problem**: Unit rapidly recalculates path

**Scenario:**
```
Frame 100: Calculate path around obstacle
Frame 101: Obstacle moves slightly
Frame 102: Recalculate path (different route)
Frame 103: Path blocked by new unit
Frame 104: Recalculate path (original route)
... infinite loop
```

**Solution:**
```cpp
// Hysteresis in path cost comparison
if (new_path_cost < old_path_cost * 0.75) {
    // Only switch paths if new path is significantly better
    use_new_path();
}
```

### 3. **Corner Cutting**

**Problem**: Units cut corners around buildings

**Visual:**
```
Building corner:
 ╔════╗
 ║    ║
 ╚═..═╝  <-- Unit tries to clip corner
   ↓
```

**Cause**: Path nodes are cell centers, unit hitbox is larger

**Fix**: Inflate obstacle boundaries in pathfinding:
```cpp
bool Is_Cell_Blocked(CELL cell, CELL from_cell)
{
    if (Map[cell].Cell_Building()) {
        // Check diagonal neighbors if moving diagonally
        if (Is_Diagonal_Move(from_cell, cell)) {
            CELL corner1 = Adjacent_Corner(from_cell, cell, 0);
            CELL corner2 = Adjacent_Corner(from_cell, cell, 1);
            if (Map[corner1].Cell_Building() || 
                Map[corner2].Cell_Building()) {
                return true;  // Would clip corner
            }
        }
    }
    return false;
}
```

### 4. **Stuck Units**

**Problem**: Unit gets permanently stuck

**Common Causes:**
1. Surrounded by buildings
2. Bridge destroyed while crossing
3. Transported into enclosed space
4. Pathfinding bug finds invalid path

**Detection:**
```cpp
void FootClass::AI()
{
    if (IsActive && Mission == MISSION_MOVE) {
        if (Coord == LastCoord) {
            StuckCounter++;
            if (StuckCounter > STUCK_THRESHOLD) {
                // Been stuck too long
                Handle_Stuck_Unit();
            }
        } else {
            StuckCounter = 0;
            LastCoord = Coord;
        }
    }
}
```

**Recovery:**
```cpp
void FootClass::Handle_Stuck_Unit()
{
    // Try scattering to random adjacent cell
    Scatter(0, true);
    
    // If still stuck, teleport slightly
    if (StuckCounter > STUCK_THRESHOLD * 2) {
        CELL current = Coord_Cell(Coord);
        CELL escape = Find_Nearest_Free_Cell(current);
        if (escape != current) {
            Unlimbo(Cell_Coord(escape));
        }
    }
}
```

### 5. **Harvester Oscillation**

**Problem**: Harvester bounces between two Tiberium patches

**Scenario:**
```
Frame N:   Go to Patch A (10 cells away)
Frame N+5: Patch B appears (2 cells away)
Frame N+6: Recalculate, go to Patch B
Frame N+10: Patch C appears near Patch A
Frame N+11: Recalculate, go to Patch A
... oscillation
```

**Fix**: Commit to target for minimum time:
```cpp
if (Mission == MISSION_HARVEST && Target_Legal(TarCom)) {
    if (Frame - MissionStartFrame < MIN_HARVEST_COMMITMENT) {
        // Don't reconsider target yet
        return;
    }
}
```

### 6. **Diagonal Movement Speed**

**Problem**: Diagonal movement is √2 faster (physics bug)

**Should be:**
- Horizontal: 1 cell = 256 leptons
- Diagonal: 1 cell = 362 leptons (256 * √2)

**Actually is:**
- Horizontal: 1 cell = 256 leptons
- Diagonal: 1 cell = 256 leptons (incorrect!)

**Result**: Units move 41% faster diagonally

**Historical Note**: This bug was in the original game, kept for authenticity

---

## Code Examples

### Example 1: Simple Unit Movement

```cpp
// Order unit to move to cell 100
UnitClass* tank = Find_Unit(UNIT_HTANK);
CELL destination = 100;

// 1. Assign mission
tank->Assign_Mission(MISSION_MOVE);
tank->Assign_Destination(Cell_Coord(destination));

// 2. Each frame, AI() is called:
void UnitClass::AI()
{
    DriveClass::AI();  // Handles movement
    
    // AI calls footclass which calls Basic_Path() when needed
    // Then follows the path by calling Start_Of_Move()
}

// 3. Movement continues until:
if (Distance(Coord, NavCom) < CLOSE_ENOUGH) {
    Enter_Idle_Mode();
    Assign_Mission(MISSION_GUARD);
}
```

### Example 2: Attack-Move

```cpp
// Move to location, attack anything encountered
void TechnoClass::Attack_Move(TARGET destination)
{
    Assign_Mission(MISSION_HUNT);
    Assign_Destination(destination);
    Set_Mission_State(MISSION_STATE_SEEKING);
}

// In mission AI:
int TechnoClass::Mission_Hunt(void)
{
    switch (Get_Mission_State()) {
        case MISSION_STATE_SEEKING:
            // Look for enemies while moving
            TARGET threat = Greatest_Threat(THREAT_RANGE);
            if (threat != TARGET_NONE) {
                Assign_Target(threat);
                Set_Mission_State(MISSION_STATE_ATTACKING);
            }
            // Continue movement
            if (!Is_Moving()) {
                if (Basic_Path()) {
                    Set_Speed(100);
                }
            }
            break;
            
        case MISSION_STATE_ATTACKING:
            // Fight enemy
            if (!Target_Legal(TarCom)) {
                Set_Mission_State(MISSION_STATE_SEEKING);
            }
            break;
    }
}
```

### Example 3: Harvester Behavior

```cpp
int UnitClass::Mission_Harvest(void)
{
    switch (Get_Mission_State()) {
        case HARVEST_STATE_FIND:
            // Search for Tiberium
            CELL tiberium_cell;
            if (Tiberium_Check(tiberium_cell)) {
                Assign_Destination(Cell_Coord(tiberium_cell));
                Set_Mission_State(HARVEST_STATE_APPROACHING);
            } else {
                // No Tiberium nearby, return to refinery
                BuildingClass* refinery = Find_My_Refinery();
                if (refinery) {
                    Assign_Destination(refinery->As_Target());
                    Set_Mission_State(HARVEST_STATE_RETURNING);
                }
            }
            break;
            
        case HARVEST_STATE_APPROACHING:
            // Move to Tiberium
            if (Distance(TarCom) < HARVEST_RANGE) {
                Set_Mission_State(HARVEST_STATE_HARVESTING);
                Set_Speed(0);  // Stop
            } else {
                // Continue moving
                if (!Is_Moving() && Basic_Path()) {
                    Set_Speed(100);
                }
            }
            break;
            
        case HARVEST_STATE_HARVESTING:
            // Collect Tiberium
            Tiberium_Load++;
            if (Tiberium_Load >= Max_Capacity()) {
                Set_Mission_State(HARVEST_STATE_RETURNING);
            }
            break;
            
        case HARVEST_STATE_RETURNING:
            // Return to refinery
            // ... unload logic
            break;
    }
}
```

### Example 4: Custom Pathfinding Hook

```cpp
// Override pathfinding for special unit behavior
class CustomUnit : public UnitClass
{
    virtual bool Basic_Path() override
    {
        // Custom logic: Prefer roads
        CELL dest = As_Cell(NavCom);
        
        // Find path that maximizes road usage
        PathType* road_path = Find_Road_Path(
            Coord_Cell(Coord), 
            dest
        );
        
        if (road_path && road_path->Cost < Normal_Path_Cost * 0.8) {
            // Road path is good, use it
            memcpy(&Path[0], road_path->Path, road_path->Length);
            return true;
        }
        
        // Fall back to normal pathfinding
        return UnitClass::Basic_Path();
    }
    
    PathType* Find_Road_Path(CELL source, CELL dest)
    {
        // A* with road preference
        // Reduce cost multiplier for road cells
        // Increase cost for off-road cells
        // ...
    }
};
```

---

## Conclusion

The pathfinding and movement system in C&C is a **masterclass in real-time AI programming under constraints**:

### Key Achievements:
1. **Deterministic**: Identical results across all network clients
2. **Fast**: Handles dozens of units at 60 FPS on 1990s CPUs
3. **Robust**: Handles dynamic obstacles, terrain changes, combat
4. **Intelligent**: Multiple strategies, obstacle avoidance, formation movement
5. **Memory Efficient**: Compact path storage, pre-computed tables

### Key Challenges Overcome:
1. A* pathfinding with multiple cost heuristics
2. Real-time dynamic replanning
3. Collision avoidance and resolution
4. Physics-based movement with sub-cell precision
5. Network synchronization requirements

### Modern Relevance:
Even today, game developers can learn from this system:
- **Simplicity**: Path as array of facing directions (not full node list)
- **Incremental**: Follow path one step at a time, allowing interruption
- **Practical**: Multiple strategies with cost comparison beats single "perfect" algorithm
- **Responsive**: Path delays and dynamic recalculation balance CPU vs responsiveness

This is not just legacy code—it's a **sophisticated, battle-tested AI system** that successfully shipped in multiple AAA games and entertained millions of players.
