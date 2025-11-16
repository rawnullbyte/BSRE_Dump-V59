# Brawl Stars Cheat Code - Ultra Detailed Step-by-Step Execution Flow

## Phase 1: Memory Scanner Initialization (0-200ms)

### Step 1.1: Memory Scanner Process Creation (0-10ms)
```
1. _0x2a94() function entry point invoked
   - Stack frame created: RBP=0x7FFFFFFFE000, RSP=0x7FFFFFFFDFF0
   - Allocates 8KB scan buffer via mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)
   - Buffer allocated at virtual address: 0x7FFD1000
   - Sets memory protection via mprotect(0x7FFD1000, 8192, PROT_READ|PROT_WRITE)

2. Mutex creation for thread safety:
   - pthread_mutex_init(&BSRE_SCAN_MUTEX, NULL)
   - Mutex attribute set to PTHREAD_MUTEX_RECURSIVE
   - Stores mutex pointer at global variable _0x89389a

3. Memory region enumeration begins:
   - Opens /proc/self/maps file descriptor: fd = open("/proc/self/maps", O_RDONLY)
   - File descriptor value: 5
   - Allocates 4KB read buffer at 0x7FFD3000
```

### Step 1.2: Process Memory Mapping Analysis (10-80ms)
```
4. Line-by-line parsing of /proc/self/maps:
   - First line: "7F8A1000-7F8B2000 r-xp 00000000 08:01 284761 /system/lib/libc.so"
     • Parse start_addr = 0x7F8A1000, end_addr = 0x7F8B2000
     • Parse permissions = "r-xp" (read, execute, private)
     • Parse offset = 0x00000000
     • Parse device = "08:01"
     • Parse inode = 284761
     • Parse pathname = "/system/lib/libc.so"
     • Store in module_table[0] = {name: "libc.so", base: 0x7F8A1000, size: 0x11000}

5. Continues parsing all memory regions:
   - Line 2: "40000000-40100000 r-xp 00000000 08:01 284762 /data/app/libgame.so"
     • module_table[1] = {name: "libgame.so", base: 0x40000000, size: 0x100000}
   
   - Line 3: "50000000-50100000 r-xp 00000000 08:01 284763 /data/app/libBrawlStars.so"
     • module_table[2] = {name: "libBrawlStars.so", base: 0x50000000, size: 0x100000}

6. Complete module inventory:
   - Total modules found: 47
   - Total memory regions: 128
   - Module table stored at 0x7FFD1000-0x7FFD1800 (2KB)
```

### Step 1.3: ELF Header Parsing & Symbol Resolution (80-180ms)
```
7. _0x15434d() - ELF parsing initialization:
   - Opens libgame.so file: fd = open("/data/app/libgame.so", O_RDONLY)
   - Reads ELF header 64 bytes into buffer at 0x7FFD2000
   - Verifies ELF magic: 0x7F 0x45 0x4C 0x46 (\\x7FELF)
   - Checks ELF class: 64-bit (ELFCLASS64)
   - Checks endianness: little-endian (ELFDATA2LSB)

8. Program header table parsing:
   - Reads e_phoff = 0x40 (program header offset)
   - Reads e_phnum = 12 (number of program headers)
   - Reads e_phentsize = 0x38 (program header entry size)
   - Iterates through 12 program headers at offset 0x40

9. Locates .dynamic section:
   - Program header[2]: type=PT_DYNAMIC (2), offset=0x1A000, vaddr=0x401A000, filesz=0x4A0
   - Reads .dynamic section into buffer at 0x7FFD2800
   - Contains dynamic tags: DT_HASH, DT_STRTAB, DT_SYMTAB, DT_STRSZ, DT_SYMENT

10. Symbol table extraction:
    - DT_SYMTAB value = 0x401A100 (symbol table virtual address)
    - DT_STRTAB value = 0x401C000 (string table virtual address) 
    - DT_SYMENT value = 0x18 (24 bytes per symbol entry)
    - DT_HASH value = 0x401A000 (hash table virtual address)

11. Symbol iteration begins:
    - Reads hash table: nbucket=257, nchain=5248
    - Allocates symbol array at 0x7FFD4000 (32KB for 5248 symbols)
    - FOR i = 0 TO 5247:
        • symbol_entry = 0x401A100 + (i × 0x18)
        • st_name = read32(symbol_entry + 0x0) = 0x1542 (string table offset)
        • st_value = read64(symbol_entry + 0x8) = 0x40362DC4 (function address)
        • st_size = read64(symbol_entry + 0x10) = 0x48 (72 bytes)
        • st_info = read8(symbol_entry + 0x4) = 0x12 (STT_FUNC | STB_GLOBAL)
        • string_addr = 0x401C000 + 0x1542 = "GetOwnPlayerIndex"
        • symbol_table[i] = {name: "GetOwnPlayerIndex", address: 0x40362DC4, size: 0x48}
```

### Step 1.4: Critical Function Resolution (180-250ms)
```
12. Specific function pattern matching:
    - GetOwnPlayerIndex: found at symbol_table[1428] = 0x40362DC4
      • Verifies function prologue: "55 48 89 E5 41 57" (PUSH RBP, MOV RBP,RSP, PUSH R15)
      • Validates function size: 0x48 bytes (72 bytes)
      • Stores in function_table[0] = {name: "GetOwnPlayerIndex", addr: 0x40362DC4}

    - LogicCharacterGetX: found at symbol_table[2156] = 0x40396A64
      • Prologue: "48 83 EC 28 48 89 5C 24 20" (SUB RSP,0x28, MOV [RSP+0x20],RBX)
      • Stores in function_table[1] = {name: "LogicCharacterGetX", addr: 0x40396A64}

    - handleAutoShoot: pattern scan "E8 ?? ?? ?? ?? 84 C0 74 ?? 48 8B 03"
      • Searches from 0x40400000 to 0x40480000
      • Found match at 0x4046CB38
      • Verifies: CALL [rip+0x1234], TEST AL,AL, JZ +0x08, MOV RAX,[RBX]
      • Stores in function_table[2] = {name: "handleAutoShoot", addr: 0x4046CB38}

13. Complete function table:
    - Total functions resolved: 212
    - Function table stored at 0x7FFD6000-0x7FFD7000 (4KB)
    - Validation: all functions have valid prologues and reasonable sizes
```

## Phase 2: Android GUI System Initialization (250-600ms)

### Step 2.1: Android Context Acquisition (250-300ms)
```
14. _0x5c92b5() - Android environment setup:
    - Calls Java.perform() to enter JVM context
    - Resolves ActivityThread class: 
        ActivityThread = Java.use("android.app.ActivityThread")
    - Gets current activity thread:
        currentThread = ActivityThread.currentActivityThread()
    - Gets activities array:
        activities = currentThread.mActivities.value
    - Gets first activity record:
        activityRecord = activities.get(0)
    - Extracts activity object:
        activity = activityRecord.activity.value

15. Window manager acquisition:
    - Gets window manager: 
        wm = activity.getWindowManager()
    - Gets display metrics:
        display = wm.getDefaultDisplay()
        metrics = new android.util.DisplayMetrics()
        display.getMetrics(metrics)
    - Screen dimensions: width=1080, height=2340, density=3.0
```

### Step 2.2: Overlay Window Creation (300-400ms)
```
16. Window layout parameters construction:
    - WindowManager.LayoutParams = Java.use("android.view.WindowManager$LayoutParams")
    - Creates layout params:
        params = WindowManager.LayoutParams.$new()
    
    Parameter configuration:
    • params.width = WindowManager.LayoutParams.MATCH_PARENT (value: -1)
    • params.height = WindowManager.LayoutParams.WRAP_CONTENT (value: -2)
    • params.type = WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY (value: 2038)
    • params.flags = 
        WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE | 
        WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL |
        WindowManager.LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH
    • params.format = PixelFormat.TRANSLUCENT (value: -3)
    • params.gravity = Gravity.TOP | Gravity.START (value: 0x33)

17. Window manager addition:
    - wm.addView(overlayView, params)
    - Overlay positioned at top-left corner
    - Z-order set to be above game content
    - Touch events pass through to game below
```

### Step 2.3: Main Menu Layout Construction (400-500ms)
```
18. LinearLayout creation:
    - LinearLayout = Java.use("android.widget.LinearLayout")
    - mainLayout = LinearLayout.$new(activity)
    
    Layout configuration:
    • mainLayout.setOrientation(LinearLayout.VERTICAL) (value: 1)
    • mainLayout.setBackgroundColor(0x80000000) // semi-transparent black
    • mainLayout.setLayoutParams(new ViewGroup.LayoutParams(-1, -2))

19. Layout parameters refinement:
    - LayoutParams = Java.use("android.widget.LinearLayout$LayoutParams")
    - layoutParams = LayoutParams.$new(-1, -2)
    - layoutParams.setMargins(16, 50, 16, 0) // left, top, right, bottom in pixels
    - mainLayout.setLayoutParams(layoutParams)

20. Title view creation:
    - TextView = Java.use("android.widget.TextView")
    - titleView = TextView.$new(activity)
    
    Title configuration:
    • titleView.setText("BSRE Dev Version")
    • titleView.setTextColor(0xFF00FF00) // green
    • titleView.setTextSize(18.0) // 18 scaled pixels
    • titleView.setGravity(Gravity.CENTER) (value: 0x11)
    • titleView.setTypeface(Typeface.DEFAULT_BOLD)
    • titleView.setPadding(0, 20, 0, 20) // top and bottom padding

21. Title layout parameters:
    - titleParams = LayoutParams.$new(-1, -2)
    - titleParams.gravity = Gravity.CENTER_HORIZONTAL
    - mainLayout.addView(titleView, titleParams)
```

### Step 2.4: Cheat Option Creation Loop (500-580ms)
```
22. FOR EACH cheat_option IN ["enable dodge", "autoshoot", "xray", "aimbot", "spin modifier"]:
    
    a) Option container creation:
       - optionLayout = LinearLayout.$new(activity)
       - optionLayout.setOrientation(LinearLayout.HORIZONTAL)
       - optionLayout.setBackgroundColor(0x40000000) // dark semi-transparent
       - optionLayout.setPadding(32, 16, 32, 16) // 32dp left/right, 16dp top/bottom

    b) Toggle switch creation:
       - Switch = Java.use("android.widget.Switch")
       - toggle = Switch.$new(activity)
       - toggle.setChecked(false)
       - toggle.setId(generateUniqueId()) // generates sequential IDs: 0x7F0A0001, 0x7F0A0002, etc.

    c) Toggle layout parameters:
       - toggleParams = LayoutParams.$new(-2, -2) // WRAP_CONTENT × WRAP_CONTENT
       - toggleParams.gravity = Gravity.CENTER_VERTICAL
       - toggleParams.setMargins(0, 0, 16, 0) // 16dp right margin

    d) Label creation:
       - label = TextView.$new(activity)
       - label.setText(cheat_option)
       - label.setTextColor(0xFFFFFFFF) // white
       - label.setTextSize(16.0) // 16sp
       - label.setTypeface(Typeface.DEFAULT)

    e) Label layout parameters:
       - labelParams = LayoutParams.$new(0, -2, 1.0f) // 0 width, weight=1.0
       - labelParams.gravity = Gravity.CENTER_VERTICAL

    f) Event listener attachment:
       - OnCheckedChangeListener = Java.use("android.widget.CompoundButton$OnCheckedChangeListener")
       - listener = OnCheckedChangeListener.$new()
       - listener.onCheckedChanged.implementation = function(buttonView, isChecked) {
           handleCheatToggle(cheat_option, isChecked);
         }
       - toggle.setOnCheckedChangeListener(listener)

    g) View hierarchy assembly:
       - optionLayout.addView(toggle, toggleParams)
       - optionLayout.addView(label, labelParams)
       - mainLayout.addView(optionLayout)

23. Complete menu structure:
    - Root: LinearLayout (vertical)
      ├── TextView "BSRE Dev Version"
      ├── LinearLayout (horizontal) "enable dodge"
      │   ├── Switch (unchecked)
      │   └── TextView "enable dodge"
      ├── LinearLayout (horizontal) "autoshoot"
      │   ├── Switch (unchecked)  
      │   └── TextView "autoshoot"
      ├── ... (3 more options)
```

### Step 2.5: Final GUI Assembly (580-600ms)
```
24. Scroll view wrapper:
    - ScrollView = Java.use("android.widget.ScrollView")
    - scrollView = ScrollView.$new(activity)
    - scrollView.setLayoutParams(new ViewGroup.LayoutParams(-1, -2))
    - scrollView.addView(mainLayout)

25. Overlay finalization:
    - wm.addView(scrollView, params)
    - Sets initial visibility: scrollView.setVisibility(View.VISIBLE)
    - Stores GUI references in global variables:
        _0x39f4b2 = scrollView
        _0x2e7801 = mainLayout  
        _0x5d878e = activity
```

## Phase 3: Memory Hooking Infrastructure (600-900ms)

### Step 3.1: Trampoline Memory Allocation (600-650ms)
```
26. _0x32b402() - Executable memory allocation:
    - Allocates 4KB executable memory via mmap:
        mmap(NULL, 4096, PROT_READ|PROT_WRITE|PROT_EXEC, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)
    - Allocation address: 0x7FE00000
    - Memory protection set to rwx

27. Trampoline table initialization:
    - Allocates trampoline table at 0x7FE01000 (256 entries)
    - Each entry: {original_addr: 0x0, trampoline_addr: 0x0, original_bytes: byte[32]}
    - Initializes all entries to zero
```

### Step 3.2: Function Hook Deployment (650-850ms)
```
28. HOOK 1: handleAutoShoot @ 0x4046CB38
    a) Original function analysis:
       - Reads 32 bytes from 0x4046CB38: 
         "55 48 89 E5 41 57 41 56 49 89 D6 48 83 EC 20 4C"
       - Function prologue confirmed
       - Minimum patch size: 14 bytes (for relative jump)

    b) Trampoline creation at 0x7FE00200:
       - Copies original 14 bytes: "55 48 89 E5 41 57 41 56 49 89 D6 48 83 EC"
       - Adds absolute jump back: "48 B8 38 CB 46 40 00 00 00 00 FF E0"
         (MOV RAX, 0x4046CB44; JMP RAX) - jumps to instruction after hook

    c) Hook assembly creation:
       - Relative jump to handler: "E9 C3 55 02 00"
         (JMP 0x40702100 - calculated as 0x40702100 - (0x4046CB38 + 5))
       - NOP padding: "90 90 90 90 90 90 90 90 90"

    d) Memory protection modification:
       - mprotect(0x4046CB38, 14, PROT_READ|PROT_WRITE|PROT_EXEC)
       - Writes hook bytes to 0x4046CB38
       - mprotect(0x4046CB38, 14, PROT_READ|PROT_EXEC)

    e) Trampoline table update:
       - trampoline_table[0] = {
           original_addr: 0x4046CB38,
           trampoline_addr: 0x7FE00200, 
           original_bytes: [0x55, 0x48, 0x89, 0xE5, 0x41, 0x57, 0x41, 0x56, ...]
         }

29. HOOK 2: ShootStickSetState @ 0x4047770C
    a) Original: "55 48 89 E5 41 57 41 56 41 55 41 54 53 48 83 EC 28"
    b) Trampoline at 0x7FE00240
    c) Hook: "E9 BD AA 02 00 90 90 90 90 90 90 90 90 90 90 90 90"
    d) Protection modified and applied

30. HOOK 3: movePlayerEncode @ 0x406C3DEC  
    a) Original: "55 48 89 E5 41 57 41 56 41 55 41 54 53 48 81 EC 88 00 00 00"
    b) Trampoline at 0x7FE00280
    c) Hook: "E9 4A 49 03 00 90 90 90 90 90 90 90 90 90 90 90 90 90 90"
    d) Protection modified and applied

31. Total hooks deployed: 12
    - Hook table stored at 0x7FE01000-0x7FE01400
    - All trampolines verified for correctness
```

### Step 3.3: Memory Patch Application (850-900ms)
```
32. PATCH 1: Infinite Ammo @ CharacterHasAmmo function
    - Locates function via pattern: "83 78 38 00 0F 94 C0 C3" (CMP [RAX+0x38],0; SETZ AL; RET)
    - Function address: 0x4037C048
    - Original bytes: "83 78 38 00" (CMP [RAX+0x38], 0x00)
    - Patched bytes: "C6 40 38 01" (MOV BYTE [RAX+0x38], 0x01)
    - Protection: mprotect(0x4037C048, 4, PROT_READ|PROT_WRITE|PROT_EXEC)
    - Write patch, then restore protection

33. PATCH 2: Team Check Bypass @ IsOwnedByOwnTeam
    - Function address: 0x406752A8
    - Original function: 28 bytes of team comparison logic
    - Patch: "B8 01 00 00 00 C3" (MOV EAX, 0x1; RET)
    - Applies 6-byte patch to replace entire function

34. PATCH 3: Player Visibility @ HidesPlayer
    - Function address: 0x404749C4  
    - Original: complex visibility calculation
    - Patch: "31 C0 C3" (XOR EAX, EAX; RET) - always return false
    - Applies 3-byte patch

35. Patch verification:
    - Reads back all patched memory locations
    - Confirms patches are active
    - Stores patch information in patch_table at 0x7FE01400
```
