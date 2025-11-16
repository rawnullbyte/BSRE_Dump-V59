# PCAPNG: https://files.catbox.moe/u1xl71.pcapng

## PHASE 1: MEMORY SCANNER INITIALIZATION

### Step 1.1: Memory Scanner Process Creation

**Line 1-10: Function Prologue and Stack Setup**
```asm
; =============================================
; _0x2a94 - MAIN MEMORY SCANNER INITIALIZATION
; =============================================
; This function sets up the memory scanning environment
; It allocates buffers, creates mutexes, and prepares for memory analysis

55                    push rbp                ; Save the current base pointer to stack
48 89 E5              mov rbp, rsp            ; Set RBP to current RSP value
48 83 EC 40           sub rsp, 0x40           ; Allocate 64 bytes of stack space for local variables
53                    push rbx                ; Save callee-saved register RBX
41 54                 push r12                ; Save callee-saved register R12
41 55                 push r13                ; Save callee-saved register R13
41 56                 push r14                ; Save callee-saved register R14  
41 57                 push r15                ; Save callee-saved register R15
```

**Line 11-20: 8KB Scan Buffer Allocation via mmap**
```asm
; =============================================
; ALLOCATE 8KB SCAN BUFFER USING MMAP SYSCALL
; =============================================
; This buffer will store module information, symbol tables, and scan results

48 C7 C0 09 00 00 00  mov rax, 9              ; Set RAX to 9 - syscall number for mmap
48 C7 C7 00 00 00 00  mov rdi, 0              ; Set RDI to 0 (NULL) - let kernel choose address
48 C7 C6 00 20 00 00  mov rsi, 0x2000         ; Set RSI to 0x2000 (8192 bytes = 8KB)
48 C7 C2 03 00 00 00  mov rdx, 3              ; Set RDX to 3 (PROT_READ | PROT_WRITE)
49 C7 C2 22 00 00 00  mov r10, 0x22           ; Set R10 to 0x22 (MAP_PRIVATE | MAP_ANONYMOUS)
49 C7 C0 FF FF FF FF  mov r8, -1              ; Set R8 to -1 (no file descriptor)
4D 31 C9              xor r9, r9              ; Set R9 to 0 (offset = 0)
0F 05                 syscall                 ; Execute the mmap syscall
```

**Line 21-30: Memory Protection Setting**
```asm
; =============================================
; SET MEMORY PROTECTION FOR SCAN BUFFER
; =============================================
; Even though mmap already set protections, we explicitly set them again

48 C7 C0 0A 00 00 00  mov rax, 10             ; Set RAX to 10 - syscall number for mprotect
48 BF 00 10 FD 7F 00  mov rdi, 0x7FFD1000     ; Set RDI to our allocated buffer address
00 00 00              
48 C7 C6 00 20 00 00  mov rsi, 0x2000         ; Set RSI to 0x2000 (8192 bytes)
48 C7 C2 03 00 00 00  mov rdx, 3              ; Set RDX to 3 (PROT_READ | PROT_WRITE)
0F 05                 syscall                 ; Execute the mprotect syscall
```

**Line 31-40: Mutex Creation for Thread Safety**
```asm
; =============================================
; CREATE MUTEX FOR THREAD SAFETY
; =============================================
; Since this cheat runs in parallel with the game, we need synchronization

48 8D 7D F8           lea rdi, [rbp-0x8]      ; Load Effective Address of mutex storage
48 31 F6              xor rsi, rsi            ; Set RSI to 0 (NULL attributes)
48 B8 20 10 40 00 00  mov rax, 0x401020       ; Load address of pthread_mutex_init
00 00 00              
FF D0                 call rax                ; Call pthread_mutex_init(&mutex, NULL)
```

**Line 41-50: Global Variable Storage**
```asm
; =============================================
; STORE CRITICAL REFERENCES IN GLOBAL VARIABLES
; =============================================
; We need to store key addresses so other functions can access them

48 B8 00 50 FD 7F 00  mov rax, 0x7FFD5000     ; Load global variable section address
00 00 00              
48 8B 55 F8           mov rdx, [rbp-0x8]      ; Load mutex pointer from stack
48 89 10              mov [rax], rdx          ; Store mutex pointer in global variable _0x89389a
C6 40 04 01           mov byte [rax+4], 1     ; Set scanner_initialized flag to true (1)
48 C7 40 08 00 10 FD  mov qword [rax+8], 0x7FFD1000 ; Store scan buffer pointer
7F              
48 C7 40 10 00 30 FD  mov qword [rax+0x10], 0x7FFD3000 ; Store read buffer pointer
7F              
```

### Step 1.2: Process Memory Mapping Analysis

**Line 51-60: Open /proc/self/maps**
```asm
; =============================================
; OPEN /proc/self/maps FOR READING
; =============================================
; /proc/self/maps shows the memory layout of our current process

48 C7 C0 02 00 00 00  mov rax, 2              ; Set RAX to 2 - syscall number for open
48 BF 00 34 01 40 00  mov rdi, 0x4013400      ; Set RDI to address of filename string
00 00 00                                      ; 0x4013400 contains "/proc/self/maps\0"
48 31 F6              xor rsi, rsi            ; Set RSI to 0 (O_RDONLY)
0F 05                 syscall                 ; Execute the open syscall
48 89 45 F0           mov [rbp-0x10], rax     ; Store file descriptor on stack at RBP-0x10
```

**Line 61-80: Read Maps File Line by Line**
```asm
; =============================================
; READ MEMORY MAPS FILE INTO BUFFER
; =============================================
; We read the entire maps file into our buffer for parsing

mov rax, 0              ; Set RAX to 0 - syscall number for read
mov rdi, [rbp-0x10]     ; Set RDI to our file descriptor
mov rsi, 0x7FFD3000     ; Set RSI to our read buffer address (0x7FFD3000)
mov rdx, 4096           ; Set RDX to 4096 (bytes to read)
syscall                 ; Execute the read syscall
mov [rbp-0x18], rax     ; Store bytes read count on stack at RBP-0x18

; Now parse the first line in the buffer
mov rdi, 0x7FFD3000     ; Set RDI to start of read buffer
call parse_maps_line    ; Call our parsing function
```

**The parse_maps_line function in detail:**
```c
// =============================================
// PARSE ONE LINE FROM /proc/self/maps
// =============================================
// Input: char* line - pointer to a null-terminated line from maps file
// Output: Adds module information to global module_table

void parse_maps_line(char* line) {
    // Step 1: Find the dash separating start and end addresses
    char* dash = strchr(line, '-');
    if (!dash) return;
    *dash = '\0';                    // Replace '-' with null terminator
    
    // Step 2: Extract start address (hex string to integer)
    uint64_t start_addr = strtoul(line, NULL, 16);
    
    // Step 3: Move past the dash we null-terminated
    char* rest = dash + 1;
    
    // Step 4: Find space after end address  
    char* space = strchr(rest, ' ');
    if (!space) return;
    *space = '\0';                   // Replace space with null terminator
    
    // Step 5: Extract end address
    uint64_t end_addr = strtoul(rest, NULL, 16);
    
    // Step 6: Calculate memory region size
    uint64_t size = end_addr - start_addr;
    
    // Continue parsing permissions, offset, device, inode, and pathname...
    
    // Step 10: Add to module table if it's a file-backed mapping
    if (pathname[0] != '\0' && pathname[0] != '[') {
        add_to_module_table(start_addr, end_addr, size, perm_bits, 
                           offset, dev_major, dev_minor, inode, pathname);
    }
}
```

**Line 81-100: Module Table Population (Continued)**
```asm
; =============================================
; POPULATE MODULE TABLE WITH PARSED INFORMATION
; =============================================
; We store each module's information in a structured table

; Store libc.so information in module_table[0]
mov rax, 0x7FFD1100          ; module_table[0] base address
mov rdx, 0x7F8A1000          ; libc.so base address from parsing
mov [rax], rdx               ; Store base_address at offset 0
mov rdx, 0x7F8B2000          ; libc.so end address  
mov [rax+8], rdx             ; Store end_address at offset 8
mov rdx, 0x11000             ; libc.so size
mov [rax+16], rdx            ; Store size at offset 16
mov byte [rax+24], 0x7       ; Store permissions: r-xp = 0x7
mov dword [rax+28], 0        ; Store offset: 0x00000000
mov byte [rax+32], 8         ; Store device major number: 8
mov byte [rax+33], 1         ; Store device minor number: 1  
mov dword [rax+34], 284761   ; Store inode number: 284761

; Copy module name "/system/lib/libc.so" to name field
lea rdi, [rax+40]            ; Load address of name field (offset 40)
mov rsi, 0x4013500           ; Load address of string "/system/lib/libc.so"
call strcpy                  ; Copy string from [RSI] to [RDI]

; Now repeat for libgame.so in module_table[1]
mov rax, 0x7FFD1200          ; module_table[1] base address
mov rdx, 0x40000000          ; libgame.so base address
mov [rax], rdx               ; module_table[1].base_address = 0x40000000
mov rdx, 0x40100000          ; libgame.so end address
mov [rax+8], rdx             ; module_table[1].end_address = 0x40100000
mov rdx, 0x100000            ; libgame.so size (1MB)
mov [rax+16], rdx            ; module_table[1].size = 0x100000

; Copy name "libgame.so"
lea rdi, [rax+40]            ; name field at 0x7FFD1228
mov rsi, 0x4013600           ; "libgame.so" string address
call strcpy

; Set module count in global variable
mov rax, 0x7FFD5000          ; Global variable section
mov dword [rax+0x20], 47     ; Store module_count = 47
```

## PHASE 2: ANDROID GUI SYSTEM

### Step 2.1: Android Context Acquisition

**Line 101-120: Java Environment Setup**
```java
// =============================================
// HOOK INTO ANDROID JAVA ENVIRONMENT USING FRIDA
// =============================================
// Frida allows us to inject JavaScript into the Java VM

Java.perform(function() {
    // Get the ActivityThread class reference
    var ActivityThread = Java.use("android.app.ActivityThread");
    
    // Get the current activity thread instance
    var currentThread = ActivityThread.currentActivityThread();
    
    // Get the activities map from the activity thread
    var activities = currentThread.mActivities.value;
    
    // Get the first activity record from the map
    var activityRecord = null;
    var iterator = activities.values().iterator();
    
    if (iterator.hasNext()) {
        activityRecord = iterator.next();
    }
    
    // Extract the actual Activity object from the activity record
    var activity = activityRecord.activity.value;
    
    // Store activity reference in global variable for later use
    _0x5d878e = activity;
});
```

**Line 121-140: Window Manager Acquisition**
```java
// =============================================
// GET WINDOW MANAGER AND DISPLAY INFORMATION
// =============================================
// The WindowManager allows us to create overlay windows

// Get the window manager from the activity
var wm = activity.getWindowManager();

// Get display metrics for screen dimensions
var display = wm.getDefaultDisplay();
var metrics = Java.use("android.util.DisplayMetrics").$new();
display.getMetrics(metrics);

// Store screen information in global variables for later use
_0x32b402.screen_width = metrics.widthPixels;    // e.g., 1080 pixels
_0x32b402.screen_height = metrics.heightPixels;  // e.g., 2340 pixels
_0x32b402.screen_density = metrics.density;      // e.g., 3.0 (xxxhdpi device)
_0x32b402.density_dpi = metrics.densityDpi;      // e.g., 480 dpi

// Log screen information for debugging
console.log("Screen: " + metrics.widthPixels + "x" + metrics.heightPixels + 
           ", Density: " + metrics.density + ", DPI: " + metrics.densityDpi);
```

### Step 2.2: Overlay Window Creation

**Line 141-160: Window Layout Parameters Construction**
```java
// =============================================
// CREATE WINDOW LAYOUT PARAMETERS FOR OVERLAY
// =============================================
// WindowManager.LayoutParams defines how our overlay window behaves

// Import required Android classes
var WindowManager = Java.use("android.view.WindowManager");
var LayoutParams = Java.use("android.view.WindowManager$LayoutParams");
var Gravity = Java.use("android.view.Gravity");
var PixelFormat = Java.use("android.graphics.PixelFormat");

// Create a new LayoutParams instance
var params = LayoutParams.$new();

// Configure window dimensions
params.width = LayoutParams.MATCH_PARENT;    // Width: -1 (fill entire screen width)
params.height = LayoutParams.WRAP_CONTENT;   // Height: -2 (wrap to content height)

// Set window type - this is critical for overlay behavior
params.type = LayoutParams.TYPE_APPLICATION_OVERLAY;  // Type: 2038

// Set window flags - control window behavior and interaction
params.flags = LayoutParams.FLAG_NOT_FOCUSABLE |      // Flag: 0x00000008
               LayoutParams.FLAG_NOT_TOUCH_MODAL |    // Flag: 0x00000020  
               LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH; // Flag: 0x00040000

// Set pixel format for transparency
params.format = PixelFormat.TRANSLUCENT;     // Format: -3

// Set window gravity (positioning)
params.gravity = Gravity.TOP | Gravity.START; // Gravity: 0x00000033

// Set initial window position
params.x = 0;                               // X offset: 0 pixels from left
params.y = 100;                             // Y offset: 100 pixels from top

// Set window transparency (alpha)
params.alpha = 0.95f;                       // Alpha: 0.95 (95% opaque)

// Store layout parameters in global variable
_0x2e7801 = params;
```

**Line 161-180: Window Manager Addition**
```java
// =============================================
// CREATE MAIN OVERLAY LAYOUT AND ADD TO WINDOW
// =============================================
// Now we create the actual overlay view and add it to the window manager

// Create the main overlay layout
var LinearLayout = Java.use("android.widget.LinearLayout");
var mainLayout = LinearLayout.$new(activity);

// Configure main layout orientation and appearance
mainLayout.setOrientation(LinearLayout.VERTICAL);  // Orientation: 1 (vertical)
mainLayout.setBackgroundColor(0x80000000);  // Background: semi-transparent black

// Set layout parameters for the main layout
var ViewGroup = Java.use("android.view.ViewGroup");
var layoutParams = ViewGroup.LayoutParams.$new(
    ViewGroup.LayoutParams.MATCH_PARENT,    // Width: -1 (match parent)
    ViewGroup.LayoutParams.WRAP_CONTENT     // Height: -2 (wrap content)
);
mainLayout.setLayoutParams(layoutParams);

// Add the overlay to window manager
wm.addView(mainLayout, params);

// Store references for later access
_0x39f4b2 = mainLayout;  // Store main layout reference
_0x2e7801 = params;      // Store layout parameters reference  

// Set initial visibility state
_0x32b402.overlay_visible = true;
_0x32b402.overlay_created = true;

// Log successful overlay creation
console.log("Overlay created successfully");
```

### Step 2.3: Main Menu Layout Construction

**Line 181-200: Title View Creation**
```java
// =============================================
// CREATE TITLE VIEW FOR CHEAT MENU
// =============================================
// The title appears at the top of the menu and shows the cheat version

// Create title TextView
var TextView = Java.use("android.widget.TextView");
var Typeface = Java.use("android.graphics.Typeface");

var titleView = TextView.$new(activity);

// Configure title text content
titleView.setText("BSRE Dev Version");

// Configure title text color (green for visibility)
titleView.setTextColor(0xFF00FF00);  // Color: bright green

// Configure title text size
titleView.setTextSize(18.0);         // Text size: 18 scaled pixels

// Configure title text alignment
titleView.setGravity(Gravity.CENTER); // Gravity: 0x11 (center)

// Configure title font style
titleView.setTypeface(Typeface.DEFAULT_BOLD); // Font: default bold

// Configure title padding (space around text)
titleView.setPadding(0, 20, 0, 20);  // Padding: left=0, top=20, right=0, bottom=20

// Create layout parameters for title view
var LayoutParams = Java.use("android.widget.LinearLayout$LayoutParams");
var titleParams = LayoutParams.$new(
    LayoutParams.MATCH_PARENT,  // Width: -1 (fill available width)
    LayoutParams.WRAP_CONTENT   // Height: -2 (wrap to text height)
);
titleParams.gravity = Gravity.CENTER_HORIZONTAL; // Horizontal alignment: center

// Add title view to main layout
mainLayout.addView(titleView, titleParams);

// Store title reference for potential later use
_0x39f4b2_title = titleView;

// Log title creation
console.log("Title view created: 'BSRE Dev Version'");
```

**Line 201-250: Cheat Option Creation Loop**
```java
// =============================================
// CREATE CHEAT OPTIONS WITH TOGGLE SWITCHES
// =============================================
// This loop creates one menu item for each cheat feature

// Array of cheat options to create
var cheatOptions = ["enable dodge", "autoshoot", "xray", "aimbot", "spin modifier"];

// Store cheat option states in global array
_0x32b402.cheat_states = {};  // Object to track enabled/disabled states

// Loop through each cheat option
for (var i = 0; i < cheatOptions.length; i++) {
    var optionName = cheatOptions[i];
    
    // Initialize cheat state to disabled (false)
    _0x32b402.cheat_states[optionName] = false;
    
    // Create horizontal container for this option
    var optionLayout = LinearLayout.$new(activity);
    optionLayout.setOrientation(LinearLayout.HORIZONTAL);
    optionLayout.setBackgroundColor(0x40000000);
    optionLayout.setPadding(32, 16, 32, 16);
    
    // Create toggle switch for enable/disable
    var Switch = Java.use("android.widget.Switch");
    var toggle = Switch.$new(activity);
    toggle.setChecked(false);        // Initial state: unchecked (disabled)
    toggle.setId(0x7F0A0001 + i);   // Set unique view ID
    
    // Create label for cheat option
    var label = TextView.$new(activity);
    label.setText(optionName);           // Set text to cheat option name
    label.setTextColor(0xFFFFFFFF);      // Text color: white
    label.setTextSize(16.0);             // Text size: 16 scaled pixels
    label.setTypeface(Typeface.DEFAULT); // Font: default (not bold)
    
    // Set up toggle event listener
    var OnCheckedChangeListener = Java.use("android.widget.CompoundButton$OnCheckedChangeListener");
    var listener = OnCheckedChangeListener.$new();
    
    // Override the onCheckedChanged method
    listener.onCheckedChanged.implementation = function(buttonView, isChecked) {
        // Handle the cheat toggle
        handleCheatToggle(optionName, isChecked);
        
        // Update global state
        _0x32b402.cheat_states[optionName] = isChecked;
        
        // Log the toggle action for debugging
        console.log("Cheat '" + optionName + "' " + (isChecked ? "ENABLED" : "DISABLED"));
    };
    
    // Attach the listener to the toggle switch
    toggle.setOnCheckedChangeListener(listener);
    
    // Assemble the option layout
    // Add toggle and label to the horizontal layout
    var toggleParams = LayoutParams.$new(
        LayoutParams.WRAP_CONTENT,  // Width: wrap to switch size
        LayoutParams.WRAP_CONTENT   // Height: wrap to switch size
    );
    toggleParams.gravity = Gravity.CENTER_VERTICAL;
    toggleParams.setMargins(0, 0, 16, 0);  // Margins: right=16dp
    
    var labelParams = LayoutParams.$new(
        0,                          // Width: 0 (use weight system)
        LayoutParams.WRAP_CONTENT,  // Height: wrap to text height
        1.0f                        // Weight: 1.0 (take all remaining space)
    );
    labelParams.gravity = Gravity.CENTER_VERTICAL;
    
    // Add views to option layout
    optionLayout.addView(toggle, toggleParams);  // Add switch first (left side)
    optionLayout.addView(label, labelParams);    // Add label second (right side)
    
    // Add completed option layout to main layout
    mainLayout.addView(optionLayout);
    
    // Store references for potential later access
    _0x39f4b2_options[optionName] = {
        layout: optionLayout,
        toggle: toggle,
        label: label
    };
}

// Log completion of menu creation
console.log("Cheat menu created with " + cheatOptions.length + " options");
```

## PHASE 3: GAME FUNCTION HOOKING

### Step 3.1: Memory Function Hooking

**Line 251-280: Critical Function Hook Installation**
```javascript
// =============================================
// INSTALL MEMORY HOOKS FOR GAME FUNCTIONALITY
// =============================================
// These hooks intercept game functions to implement cheat features

function installGameHooks() {
    console.log("[+] Installing game function hooks...");
    
    // Hook auto-shoot function
    const autoShootAddr = _0x32b402.gameBase.add(0x0046CB38);
    Interceptor.attach(autoShootAddr, {
        onEnter: function(args) {
            if (_0x32b402.cheat_states["autoshoot"]) {
                // Let our auto-shoot system handle shooting
                this.skip = true;
            }
        }
    });
    
    // Hook player visibility function for X-Ray
    const hidePlayerAddr = _0x32b402.gameBase.add(0x004749C4);
    Interceptor.attach(hidePlayerAddr, {
        onEnter: function(args) {
            if (_0x32b402.cheat_states["xray"]) {
                // Force return false to always show players
                this.returnValue = 0;
            }
        }
    });
    
    // Hook team check function
    const teamCheckAddr = _0x32b402.gameBase.add(0x006752A8);
    Interceptor.attach(teamCheckAddr, {
        onEnter: function(args) {
            // Bypass team checks for targeting
            this.returnValue = 1;
        }
    });
    
    console.log("[+] Game hooks installed successfully");
}
```

### Step 3.2: Memory Patches

**Line 281-300: Critical Memory Patches**
```javascript
// =============================================
// APPLY MEMORY PATCHES FOR CHEAT FUNCTIONALITY
// =============================================
// These patches modify game code to enable cheat features

function applyMemoryPatches() {
    console.log("[+] Applying memory patches...");
    
    // Patch ammo check to always return true (infinite ammo)
    const ammoCheckAddr = _0x32b402.gameBase.add(0x037C048);
    Memory.patchCode(ammoCheckAddr, 4, function(code) {
        const writer = new X86Writer(code, { pc: ammoCheckAddr });
        writer.putMovRegU32('eax', 0x1);  // Always return true
        writer.putRet();
        writer.flush();
    });
    
    // Patch cooldown checks
    const cooldownAddr = _0x32b402.gameBase.add(0x0340F60);
    Memory.patchCode(cooldownAddr, 8, function(code) {
        const writer = new X86Writer(code, { pc: cooldownAddr });
        writer.putNop();
        writer.putNop();
        writer.putNop();
        writer.putNop();
        writer.putNop();
        writer.putNop();
        writer.putNop();
        writer.putNop();
        writer.flush();
    });
    
    console.log("[+] Memory patches applied successfully");
}
```

## PHASE 4: CHEAT FUNCTIONALITY IMPLEMENTATION

### Step 4.1: Aimbot System

**Line 301-350: Aimbot Logic Implementation**
```javascript
// =============================================
// AIMBOT SYSTEM IMPLEMENTATION
// =============================================
// This system automatically aims at enemies and optionally shoots

function updateAimbot() {
    if (!_0x32b402.cheat_states["aimbot"] && !_0x32b402.cheat_states["autoshoot"]) return;
    
    const currentTime = Date.now();
    if (currentTime - _0x32b402.lastAimbotUpdate < 16) return; // ~60 FPS
    _0x32b402.lastAimbotUpdate = currentTime;
    
    // Update player data
    updatePlayerData();
    
    // Find best target
    const target = findBestTarget();
    if (!target) return;
    
    // Calculate aim angles
    const aimData = calculateAimAngles(_0x32b402.localPlayer, target);
    
    // Write aim coordinates to game memory
    if (_0x32b402.cheat_states["aimbot"]) {
        Memory.writeFloat(_0x32b402.aimStickX, aimData.aimX);
        Memory.writeFloat(_0x32b402.aimStickY, aimData.aimY);
    }
    
    // Auto shoot if enabled
    if (_0x32b402.cheat_states["autoshoot"] && aimData.inRange) {
        triggerAutoShoot();
    }
}

function findBestTarget() {
    let bestTarget = null;
    let bestScore = 0;
    
    for (let i = 0; i < 10; i++) {
        if (i === _0x32b402.localPlayer.index) continue;
        
        const enemyPtr = _0x32b402.playerArray.add(i * 0x268);
        const enemyTeam = Memory.readU8(enemyPtr.add(0x34));
        
        // Skip friendly players and invalid entries
        if (enemyTeam === _0x32b402.localPlayer.team || enemyTeam === 0xFF) continue;
        
        const enemyHealth = Memory.readS32(enemyPtr.add(0x50));
        if (enemyHealth <= 0) continue; // Skip dead players
        
        const enemyX = Memory.readFloat(enemyPtr.add(0x40));
        const enemyY = Memory.readFloat(enemyPtr.add(0x44));
        
        // Calculate distance
        const dx = enemyX - _0x32b402.localPlayer.x;
        const dy = enemyY - _0x32b402.localPlayer.y;
        const distance = Math.sqrt(dx * dx + dy * dy);
        
        // Calculate target score (closer + lower health = better)
        const score = (1000 / distance) * (100 / Math.max(1, enemyHealth));
        
        if (score > bestScore) {
            bestScore = score;
            bestTarget = {
                index: i,
                x: enemyX,
                y: enemyY,
                health: enemyHealth,
                distance: distance,
                pointer: enemyPtr
            };
        }
    }
    
    return bestTarget;
}
```

### Step 4.2: Dodge System

**Line 351-400: Dodge Logic Implementation**
```javascript
// =============================================
// DODGE SYSTEM IMPLEMENTATION
// =============================================
// This system automatically dodges incoming projectiles

function updateDodgeSystem() {
    if (!_0x32b402.cheat_states["enable dodge"]) return;
    
    const currentTime = Date.now();
    if (currentTime - _0x32b402.lastDodgeUpdate < 8) return; // 120 FPS for faster reaction
    _0x32b402.lastDodgeUpdate = currentTime;
    
    // Scan for incoming projectiles
    const dangerousProjectile = findDangerousProjectile();
    if (!dangerousProjectile) return;
    
    // Calculate dodge vector
    const dodgeVector = calculateDodgeVector(dangerousProjectile);
    
    // Apply dodge movement
    Memory.writeFloat(_0x32b402.moveStickX, dodgeVector.x);
    Memory.writeFloat(_0x32b402.moveStickY, dodgeVector.y);
}

function findDangerousProjectile() {
    for (let i = 0; i < 50; i++) {
        const projPtr = _0x32b402.projectileArray.add(i * 0x60);
        const isActive = Memory.readU8(projPtr);
        
        if (isActive === 1) {
            const projX = Memory.readFloat(projPtr.add(0x10));
            const projY = Memory.readFloat(projPtr.add(0x14));
            const projVX = Memory.readFloat(projPtr.add(0x20));
            const projVY = Memory.readFloat(projPtr.add(0x24));
            const radius = Memory.readFloat(projPtr.add(0x30));
            const owner = Memory.readU8(projPtr.add(0x38));
            
            // Skip own projectiles
            if (owner === _0x32b402.localPlayer.index) continue;
            
            // Calculate collision time
            const collisionTime = calculateCollisionTime(
                _0x32b402.localPlayer.x, _0x32b402.localPlayer.y,
                projX, projY, projVX, projVY, radius
            );
            
            // If projectile will hit within 0.5 seconds, it's dangerous
            if (collisionTime > 0 && collisionTime < 0.5) {
                return {
                    x: projX,
                    y: projY,
                    velocityX: projVX,
                    velocityY: projVY,
                    radius: radius,
                    collisionTime: collisionTime
                };
            }
        }
    }
    return null;
}
```

### Step 4.3: X-Ray System

**Line 401-450: X-Ray Implementation**
```javascript
// =============================================
// X-RAY SYSTEM IMPLEMENTATION
// =============================================
// This system makes walls transparent and highlights enemies

function toggleXRay(enabled) {
    if (enabled) {
        enableXRay();
    } else {
        disableXRay();
    }
}

function enableXRay() {
    console.log("[+] X-Ray enabled - making walls transparent");
    
    // Make all players visible by patching visibility function
    // This is already handled by our hook
    
    // Additional: highlight enemies
    setEnemyHighlight(true);
}

function disableXRay() {
    console.log("[+] X-Ray disabled");
    setEnemyHighlight(false);
}

function setEnemyHighlight(enabled) {
    for (let i = 0; i < 10; i++) {
        if (i === _0x32b402.localPlayer.index) continue;
        
        const enemyPtr = _0x32b402.playerArray.add(i * 0x268);
        const enemyTeam = Memory.readU8(enemyPtr.add(0x34));
        
        if (enemyTeam !== _0x32b402.localPlayer.team && enemyTeam !== 0xFF) {
            const glowPtr = enemyPtr.add(0x100); // Glow effect address
            
            if (enabled) {
                Memory.writeFloat(glowPtr, 1.0);       // Red
                Memory.writeFloat(glowPtr.add(4), 0.0); // Green
                Memory.writeFloat(glowPtr.add(8), 0.0); // Blue
                Memory.writeFloat(glowPtr.add(12), 0.8); // Alpha
            } else {
                Memory.writeFloat(glowPtr, 0.0);       // Red
                Memory.writeFloat(glowPtr.add(4), 0.0); // Green
                Memory.writeFloat(glowPtr.add(8), 0.0); // Blue
                Memory.writeFloat(glowPtr.add(12), 0.0); // Alpha
            }
        }
    }
}
```

## PHASE 5: MAIN LOOP AND INITIALIZATION

### Step 5.1: Main Execution Loop

**Line 451-500: Main Loop Implementation**
```javascript
// =============================================
// MAIN EXECUTION LOOP
// =============================================
// This loop runs continuously and updates all cheat systems

function mainLoop() {
    try {
        // Update all active cheat systems
        if (_0x32b402.cheat_states["aimbot"] || _0x32b402.cheat_states["autoshoot"]) {
            updateAimbot();
        }
        
        if (_0x32b402.cheat_states["enable dodge"]) {
            updateDodgeSystem();
        }
        
        if (_0x32b402.cheat_states["spin modifier"]) {
            updateSpinBot();
        }
        
        // Update anti-AFK system (always runs if enabled)
        updateAntiAFK();
        
    } catch (error) {
        console.log("[-] Error in main loop: " + error);
    }
    
    // Schedule next frame (run at ~1000 FPS for responsiveness)
    setTimeout(mainLoop, 1);
}

function updateSpinBot() {
    if (!_0x32b402.cheat_states["spin modifier"]) return;
    
    const currentTime = Date.now();
    if (currentTime - _0x32b402.lastSpinUpdate < 50) return; // 20 FPS for spinning
    _0x32b402.lastSpinUpdate = currentTime;
    
    // Calculate spin angle (360 degrees over 2 seconds)
    const spinSpeed = 360 / 2000; // degrees per millisecond
    const elapsed = currentTime % 2000; // Loop every 2 seconds
    const angle = (elapsed * spinSpeed) * (Math.PI / 180); // Convert to radians
    
    // Calculate movement vector based on angle
    const moveX = Math.cos(angle) * 0.7;
    const moveY = Math.sin(angle) * 0.7;
    
    // Apply spin movement
    Memory.writeFloat(_0x32b402.moveStickX, moveX);
    Memory.writeFloat(_0x32b402.moveStickY, moveY);
}

function updateAntiAFK() {
    // Every 30 seconds, simulate a small movement to prevent AFK detection
    const currentTime = Date.now();
    if (currentTime % 30000 < 16) { // Every ~30 seconds
        const randomX = (Math.random() - 0.5) * 0.1;
        const randomY = (Math.random() - 0.5) * 0.1;
        
        Memory.writeFloat(_0x32b402.moveStickX, randomX);
        Memory.writeFloat(_0x32b402.moveStickY, randomY);
    }
}
```

### Step 5.2: Complete Initialization

**Line 501-550: Final Initialization Sequence**
```javascript
// =============================================
// COMPLETE CHEAT INITIALIZATION
// =============================================
// This function orchestrates the entire cheat setup process

function initializeCheat() {
    console.log("[+] Starting Brawl Stars cheat initialization...");
    
    try {
        // Step 1: Initialize memory scanner and find game modules
        if (!initializeMemoryScanner()) {
            console.log("[-] Failed to initialize memory scanner");
            return false;
        }
        
        // Step 2: Create overlay GUI
        createOverlayGUI();
        
        // Step 3: Install memory hooks
        installGameHooks();
        
        // Step 4: Apply memory patches
        applyMemoryPatches();
        
        // Step 5: Initialize cheat state tracking
        initializeCheatState();
        
        // Step 6: Start main loop
        console.log("[+] Cheat initialized successfully - starting main loop");
        mainLoop();
        
        return true;
        
    } catch (error) {
        console.log("[-] Cheat initialization failed: " + error);
        return false;
    }
}

function initializeCheatState() {
    // Initialize timing variables
    _0x32b402.lastAimbotUpdate = 0;
    _0x32b402.lastDodgeUpdate = 0;
    _0x32b402.lastSpinUpdate = 0;
    
    // Initialize player data structure
    _0x32b402.localPlayer = {
        index: -1,
        x: 0,
        y: 0,
        z: 0,
        team: 0,
        health: 0,
        maxHealth: 0
    };
    
    // Resolve critical game addresses
    _0x32b402.playerArray = _0x32b402.gameBase.add(0x19D1E1C);
    _0x32b402.projectileArray = _0x32b402.gameBase.add(0x19D6998);
    _0x32b402.aimStickX = _0x32b402.gameBase.add(0x19D250C);
    _0x32b402.aimStickY = _0x32b402.gameBase.add(0x19D2510);
    _0x32b402.moveStickX = _0x32b402.gameBase.add(0x19D2514);
    _0x32b402.moveStickY = _0x32b402.gameBase.add(0x19D2518);
    
    console.log("[+] Cheat state initialized successfully");
}

// =============================================
// START THE CHEAT
// =============================================
// This is the entry point that gets called when the script loads

Java.perform(function() {
    console.log("[+] Frida script loaded - initializing cheat...");
    
    // Start the cheat initialization process
    setTimeout(function() {
        if (initializeCheat()) {
            console.log("[+] Brawl Stars cheat is now active!");
            console.log("[+] Overlay menu should be visible on screen");
            console.log("[+] Use the toggle switches to enable features");
        } else {
            console.log("[-] Cheat failed to initialize");
        }
    }, 2000); // Wait 2 seconds for game to fully load
});
```
