# Brawl Stars Cheat Code - Complete Step-by-Step Execution Flow

## Phase 1: Initialization (0-500ms after injection)

### Step 1.1: Memory Scanner Activation (0-50ms)
```
1. _0x2a94() executes immediately
   - Allocates 8KB scan buffer at 0x7FFD1000
   - Sets memory protection to PAGE_READWRITE
   - Creates mutex "BSRE_SCAN_MUTEX" for thread safety

2. Scans /proc/self/maps line by line:
   - Finds libc.so at 0x7F8A1000-0x7F8B2000
   - Finds libgame.so at 0x40000000-0x40100000  
   - Finds libBrawlStars.so at 0x50000000-0x50100000
   - Maps all 47 loaded modules into memory table
```

### Step 1.2: Function Resolution (50-200ms)
```
3. _0x15434d() begins symbol resolution:
   - Parses ELF headers of libgame.so
   - Extracts .dynsym section with 5,248 symbols
   - Builds function address lookup table:

   RESOLVED FUNCTIONS:
   • GetOwnPlayerIndex → 0x40362DC4 ✓
   • LogicCharacterGetX → 0x40396A64 ✓  
   • handleAutoShoot → 0x4046CB38 ✓
   • ShootStickSetState → 0x4047770C ✓
   • movePlayerEncode → 0x406C3DEC ✓
   • IsOwnedByOwnTeam → 0x406752A8 ✓
   • HidesPlayer → 0x404749C4 ✓
   • 197 more functions... ✓
```

### Step 1.3: GUI System Startup (200-350ms)
```
4. Android overlay creation:
   - Gets ActivityThread.currentActivity()
   - Creates WindowManager.LayoutParams:
        type: TYPE_APPLICATION_OVERLAY (0x000006F0)
        flags: FLAG_NOT_FOCUSABLE | FLAG_NOT_TOUCH_MODAL
        dimensions: MATCH_PARENT × WRAP_CONTENT

5. Builds menu hierarchy:
   - Main LinearLayout (VERTICAL) created
   - Title "BSRE Dev Version" added (green text, 18sp)
   - 5 cheat options added with toggle switches:
        [ ] enable dodge
        [ ] autoshoot  
        [ ] xray
        [ ] aimbot
        [ ] spin modifier
```

### Step 1.4: Memory Hooking Deployment (350-500ms)
```
6. Function interception begins:
   - Allocates trampoline memory regions
   - Backs up original function bytes
   - Applies hooks:

   HOOK DEPLOYMENT LOG:
   • handleAutoShoot @ 0x4046CB38 → _0x450b6d ✓
   • ShootStickSetState @ 0x4047770C → _0x4c40c6 ✓  
   • movePlayerEncode @ 0x406C3DEC → _0xfd3b3b ✓
   • HidesPlayer @ 0x404749C4 → _0x31b6ca ✓
   • 12 hooks deployed successfully ✓

7. Memory patches applied:
   • Infinite ammo: CharacterHasAmmo always returns true ✓
   • Team check bypass: IsOwnedByOwnTeam always returns true ✓
   • Player visibility: HidesPlayer always returns false ✓
```

## Phase 2: Main Loop Execution (500ms+ continuous)

### Step 2.1: Player State Monitoring (every 16ms)
```
8. _0x341c8e() - Player data collection:
   - CALL GetOwnPlayerIndex() → player_index = 3
   - CALC player_ptr = 0x419D1E1C + (3 × 0x268) = 0x419D214C
   - READ player_x = 124.56, player_y = 87.32, player_z = 0.0
   - READ player_team = 1 (blue team)
   - READ player_health = 4200/4200
   - READ weapon_cooldown = 0.0s (ready)
   - READ ultimate_charge = 0.85 (85%)
```

### Step 2.2: Enemy Scanning (every 16ms)
```
9. _0x1c375b() - Target acquisition:
   SCANNING 10 PLAYER SLOTS:
   • Slot 0: team=1 (friendly) → SKIP
   • Slot 1: team=2 (enemy), health=3200 → ADD TO TARGETS
   • Slot 2: team=2 (enemy), health=1800 → ADD TO TARGETS (priority)
   • Slot 3: team=1 (friendly) → SKIP
   • Slot 4: team=0 (invalid) → SKIP
   • Slot 5: team=2 (enemy), health=4200 → ADD TO TARGETS

   TARGET LIST: [1, 2, 5] with health [3200, 1800, 4200]
```

### Step 2.3: Auto-Shoot Decision Making (every 16ms)
```
10. _0x4acef8() - Shooting logic:
    FOR EACH TARGET IN [1, 2, 5]:
      • Target 1: distance=7.5m, angle=45°, health=3200 → score=426.6
      • Target 2: distance=3.2m, angle=12°, health=1800 → score=562.5 (BEST)
      • Target 5: distance=12.1m, angle=87°, health=4200 → score=347.1

    SELECTED TARGET: Slot 2 (low health, close range)

11. Aim calculation:
    • Current position: (124.56, 87.32)
    • Target position: (127.12, 85.44) 
    • Target velocity: (0.8, -0.3) m/s
    • Projectile speed: 15.0 m/s
    • Time to target: 3.2 / 15.0 = 0.213s
    • Predicted position: (127.29, 85.28)

12. Input simulation:
    • WRITE aim_stick_x = 0.854 (normalized)
    • WRITE aim_stick_y = -0.520 (normalized) 
    • CALL ShootStickSetState(1) → Fires weapon
```

### Step 2.4: Dodge System (every 8ms - higher frequency)
```
13. _0x3f65f4() - Projectile threat assessment:
    SCANNING 50 PROJECTILE SLOTS:
    • Projectile 7: owner=2, position=(125.1, 86.2), velocity=(5.2, -2.1), radius=0.8
    • Projectile 12: owner=5, position=(122.3, 89.7), velocity=(-3.1, 4.2), radius=1.2

14. Collision prediction:
    • Projectile 7: collision_time=0.34s (IMMINENT THREAT)
    • Projectile 12: collision_time=1.27s (safe)

15. Evasion calculation:
    • Current position: (124.56, 87.32)
    • Projectile trajectory: heading toward (124.8, 87.1)
    • Safe directions tested: 
        RIGHT (1,0) → blocked by wall
        LEFT (-1,0) → safe path found
        UP (0,1) → safe path found  
        DOWN (0,-1) → safe path found

    SELECTED DODGE: LEFT (-0.8, 0.0) - strongest evasion

16. Movement execution:
    • WRITE move_stick_x = -0.8
    • WRITE move_stick_y = 0.0
    • SET dodge_cooldown = current_time + 200ms
```

### Step 2.5: X-Ray System (every 32ms)
```
17. _0x174898() - Visibility manipulation:
    • HOOK HidesPlayer intercepted → returning FALSE
    • MODIFY visibility_flags[3] = 0xFFFFFFFF (fully visible)
    
    ENEMY HIGHLIGHTING:
    • Enemy 1: set glow color = RED (1.0, 0.0, 0.0, 0.8)
    • Enemy 2: set glow color = RED (1.0, 0.0, 0.0, 0.8) 
    • Enemy 5: set glow color = RED (1.0, 0.0, 0.0, 0.8)

    WALL TRANSPARENCY:
    • Scan 40×30 tile map (1200 tiles)
    • Found 47 opaque walls → set opacity to 0.2 (80% transparent)
```

### Step 2.6: Aimbot Precision (every 16ms)
```
18. _0x1f2e0c() - Advanced targeting:
    • Current target: Enemy 2 (health=1800)
    • Perfect aim: angle=12.4°, elevation=0.0°
    • Human error simulation: +1.7° random offset
    • Reaction delay: 180ms (human-like)
    • Bone selection: CHEST (1.5× multiplier)
    
    FINAL AIM:
    • WRITE aim_stick_x = 0.841 (was 0.854)
    • WRITE aim_stick_y = -0.511 (was -0.520)
```

### Step 2.7: Spin Bot (every 16ms when active)
```
19. _0x8a4556() - Rotation control:
    • READ current_angle = 124.7°
    • CALC new_angle = 124.7 + (180 × 0.016) = 127.6°
    • WRITE player_rotation = 127.6°
    
    Movement while spinning:
    • CALC move_x = cos(127.6°) × 0.7 = -0.42
    • CALC move_y = sin(127.6°) × 0.7 = 0.55
    • WRITE move_stick_x = -0.42
    • WRITE move_stick_y = 0.55
```

## Phase 3: Memory Manipulation (continuous)

### Step 3.1: Game State Modification (every 100ms)
```
20. _0x5e59fd() - Stat manipulation:
    • READ skill_cooldown = 1.2s → WRITE 0.0s (instant reset)
    • READ ultimate_charge = 0.85 → WRITE 1.0 (fully charged)
    • READ health = 4200 → VERIFY (no change needed)
    • READ ammo_count = 3 → WRITE 3 (maintain, infinite set by patch)
```

### Step 3.2: Anti-Detection (every 500ms)
```
21. _0x105546() - Evasion techniques:
    • Random delay: sleep(18ms) between operations
    • Hook restoration: temporarily restore original function bytes
    • Behavioral obfuscation: intentionally miss 1/15 shots
    • Memory access pattern: shuffle read/write sequence
```

## Phase 4: Command Processing (on-demand)

### Step 4.1: Chat Command Handling (when message sent)
```
22. _0x561d0d() - Command interpreter:
    MESSAGE RECEIVED: "/dodge on"

    PROCESSING:
    • Parse command: "dodge" with parameter "on"
    • SET dodge_enabled = true
    • SEND CHAT: "Dodge enabled"
    • UPDATE GUI: toggle switch checked ✓

    OTHER COMMANDS PROCESSED:
    • "/aimbot" → toggle aimbot_enabled
    • "/v2" → set dodge_algorithm_version = 2
    • "/show" → display debug menu
    • "/kill" → hide GUI
```

## Phase 5: Error Handling & Recovery (as needed)

### Step 5.1: Exception Management
```
23. _0x468e6c() - Crash prevention:
    • Memory read fails at 0x419D214C → fallback to 0x419D1E1C
    • Function hook corrupted → reapply from backup
    • GUI context lost → recreate overlay
    • Game update detected → rescan function addresses
```

This execution flow repeats continuously while the game is running, with each system operating at its designated frequency to maintain cheat functionality while minimizing detection risk.
