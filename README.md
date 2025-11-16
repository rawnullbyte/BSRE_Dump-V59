## PHASE 1: MEMORY SCANNER INITIALIZATION

### Step 1.1: Memory Scanner Process Creation

**Line 1-10: Function Prologue and Stack Setup**
```asm
; _0x2a94 function begins
55                    push rbp                ; Save old base pointer
48 89 E5              mov rbp, rsp            ; Set new base pointer
48 83 EC 40           sub rsp, 0x40           ; Allocate 64 bytes stack space
53                    push rbx                ; Save RBX (callee-saved)
41 54                 push r12                ; Save R12
41 55                 push r13                ; Save R13  
41 56                 push r14                ; Save R14
41 57                 push r15                ; Save R15
```

**Example**: When function is called, RSP was 0x7FFFFFFFE000. After these instructions:
- RBP = 0x7FFFFFFFE000 (frame pointer)
- RSP = 0x7FFFFFFFDFB0 (64 bytes allocated + 5 registers pushed)
- Stack contains saved registers for later restoration

**Line 11-20: 8KB Scan Buffer Allocation via mmap**
```asm
; mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)
48 C7 C0 09 00 00 00  mov rax, 9              ; mmap syscall number
48 C7 C7 00 00 00 00  mov rdi, 0              ; NULL address (let kernel choose)
48 C7 C6 00 20 00 00  mov rsi, 0x2000         ; 8192 bytes size
48 C7 C2 03 00 00 00  mov rdx, 3              ; PROT_READ|PROT_WRITE
49 C7 C2 22 00 00 00  mov r10, 0x22           ; MAP_PRIVATE|MAP_ANONYMOUS
49 C7 C0 FF FF FF FF  mov r8, -1              ; No file descriptor
4D 31 C9              xor r9, r9              ; Offset 0
0F 05                 syscall                 ; Execute mmap
```

**Example**: After syscall returns:
- RAX = 0x7FFD1000 (kernel-chosen address for our buffer)
- Memory region 0x7FFD1000-0x7FFD3000 now allocated and writable

**Line 21-30: Memory Protection Setting**
```asm
; mprotect(0x7FFD1000, 8192, PROT_READ|PROT_WRITE)
48 C7 C0 0A 00 00 00  mov rax, 10             ; mprotect syscall number
48 BF 00 10 FD 7F 00  mov rdi, 0x7FFD1000     ; Buffer address
00 00 00
48 C7 C6 00 20 00 00  mov rsi, 0x2000         ; 8192 bytes
48 C7 C2 03 00 00 00  mov rdx, 3              ; PROT_READ|PROT_WRITE
0F 05                 syscall                 ; Execute mprotect
```

**Line 31-40: Mutex Creation for Thread Safety**
```asm
; pthread_mutex_init(&BSRE_SCAN_MUTEX, NULL)
48 8D 7D F8           lea rdi, [rbp-0x8]      ; Load mutex address (RBP-8)
48 31 F6              xor rsi, rsi            ; NULL attributes
48 B8 20 10 40 00 00  mov rax, 0x401020       ; pthread_mutex_init PLT entry
00 00 00
FF D0                 call rax                ; Call pthread_mutex_init
```

**Example**: Creates recursive mutex at address 0x7FFFFFFFDFE8 that allows the same thread to lock it multiple times without deadlock.

**Line 41-50: Global Variable Storage**
```asm
; Store mutex pointer in global variable
48 B8 00 50 FD 7F 00  mov rax, 0x7FFD5000     ; Global variable section
00 00 00
48 8B 55 F8           mov rdx, [rbp-0x8]      ; Load mutex pointer
48 89 10              mov [rax], rdx          ; Store in _0x89389a

; Set initialization flags
C6 40 04 01           mov byte [rax+4], 1     ; scanner_initialized = true
48 C7 40 08 00 10 FD  mov qword [rax+8], 0x7FFD1000 ; scan_buffer_ptr
7F
48 C7 40 10 00 30 FD  mov qword [rax+0x10], 0x7FFD3000 ; read_buffer_ptr
7F
```

### Step 1.2: Process Memory Mapping Analysis

**Line 51-60: Open /proc/self/maps**
```asm
; open("/proc/self/maps", O_RDONLY)
48 C7 C0 02 00 00 00  mov rax, 2              ; open syscall
48 BF 00 34 01 40 00  mov rdi, 0x4013400      ; "/proc/self/maps" string
00 00 00
48 31 F6              xor rsi, rsi            ; O_RDONLY = 0
0F 05                 syscall                 ; Execute open
48 89 45 F0           mov [rbp-0x10], rax     ; Store file descriptor
```

**Example**: Opens the memory maps file which contains:
```
7F8A1000-7F8B2000 r-xp 00000000 08:01 284761 /system/lib/libc.so
40000000-40100000 r-xp 00000000 08:01 284762 /data/app/libgame.so
50000000-50100000 r-xp 00000000 08:01 284763 /data/app/libBrawlStars.so
60000000-60100000 rw-p 00000000 00:00 0 [heap]
7FFFFFFDE000-7FFFFFFE0000 rw-p 00000000 00:00 0 [stack]
```

**Line 61-80: Read Maps File Line by Line**
```asm
; read(fd, buffer, 4096)
mov rax, 0              ; read syscall
mov rdi, [rbp-0x10]     ; file descriptor (5)
mov rsi, 0x7FFD3000     ; read buffer
mov rdx, 4096           ; bytes to read
syscall
mov [rbp-0x18], rax     ; store bytes read (1024)

; Parse first line "7F8A1000-7F8B2000 r-xp 00000000 08:01 284761 /system/lib/libc.so"
mov rdi, 0x7FFD3000     ; start of buffer
call parse_maps_line    ; Custom parsing function
```

**The parse_maps_line function works like this:**
```c
void parse_maps_line(char* line) {
    // Find first '-' character
    char* dash = strchr(line, '-');
    *dash = '\0';  // Null terminate start address
    
    // Extract start address
    uint64_t start_addr = strtoul(line, NULL, 16);  // "7F8A1000" -> 0x7F8A1000
    
    // Move past dash
    char* rest = dash + 1;
    
    // Find space after end address
    char* space = strchr(rest, ' ');
    *space = '\0';  // Null terminate end address
    
    // Extract end address  
    uint64_t end_addr = strtoul(rest, NULL, 16);  // "7F8B2000" -> 0x7F8B2000
    
    // Calculate size
    uint64_t size = end_addr - start_addr;  // 0x11000 bytes
    
    // Continue parsing permissions, offset, device, inode, pathname...
}
```

**Line 81-100: Module Table Population**
```asm
; Store module information in module_table[0]
mov rax, 0x7FFD1100          ; module_table[0] base
mov rdx, 0x7F8A1000
mov [rax], rdx               ; base_address = 0x7F8A1000
mov rdx, 0x7F8B2000  
mov [rax+8], rdx             ; end_address = 0x7F8B2000
mov rdx, 0x11000
mov [rax+16], rdx            ; size = 0x11000
mov byte [rax+24], 0x7       ; permissions = r-xp
mov dword [rax+28], 0        ; offset = 0
mov byte [rax+32], 8         ; device_major = 8
mov byte [rax+33], 1         ; device_minor = 1  
mov dword [rax+34], 284761   ; inode = 284761

; Copy module name
lea rdi, [rax+40]            ; name field at offset 40
mov rsi, 0x4013500           ; "/system/lib/libc.so" string
call strcpy                  ; Copy string
```

This continues for all 47 modules found in the maps file.

## PHASE 2: ANDROID GUI SYSTEM

### Step 2.1: Android Context Acquisition

**Line 101-120: Java Environment Setup**
```java
// Hook into Android system using Frida
Java.perform(function() {
    // Get ActivityThread class
    var ActivityThread = Java.use("android.app.ActivityThread");
    
    // Get current activity thread
    var currentThread = ActivityThread.currentActivityThread();
    
    // Get activities from mActivities field (HashMap)
    var activities = currentThread.mActivities.value;
    
    // Get first activity record
    var activityRecord = null;
    var iterator = activities.values().iterator();
    if (iterator.hasNext()) {
        activityRecord = iterator.next();
    }
    
    // Extract activity object from activityRecord
    var activity = activityRecord.activity.value;
    
    // Store activity in global variable
    _0x5d878e = activity;
});
```

**Line 121-140: Window Manager Acquisition**
```java
// Get window manager from activity
var wm = activity.getWindowManager();

// Get display metrics for screen dimensions
var display = wm.getDefaultDisplay();
var metrics = Java.use("android.util.DisplayMetrics").$new();
display.getMetrics(metrics);

// Store screen information
_0x32b402.screen_width = metrics.widthPixels;    // 1080
_0x32b402.screen_height = metrics.heightPixels;  // 2340  
_0x32b402.screen_density = metrics.density;      // 3.0
```

### Step 2.2: Overlay Window Creation

**Line 141-160: Window Layout Parameters Construction**
```java
// Create WindowManager.LayoutParams
var WindowManager = Java.use("android.view.WindowManager");
var LayoutParams = Java.use("android.view.WindowManager$LayoutParams");
var Gravity = Java.use("android.view.Gravity");
var PixelFormat = Java.use("android.graphics.PixelFormat");

// Instantiate layout parameters
var params = LayoutParams.$new();

// Configure parameters
params.width = LayoutParams.MATCH_PARENT;    // -1
params.height = LayoutParams.WRAP_CONTENT;   // -2  
params.type = LayoutParams.TYPE_APPLICATION_OVERLAY;  // 2038
params.flags = LayoutParams.FLAG_NOT_FOCUSABLE | 
               LayoutParams.FLAG_NOT_TOUCH_MODAL |
               LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH;
params.format = PixelFormat.TRANSLUCENT;     // -3
params.gravity = Gravity.TOP | Gravity.START; // 0x33

// Set initial position
params.x = 0;
params.y = 100;  // 100 pixels from top
```

**Line 161-180: Window Manager Addition**
```java
// Create the main overlay view
var LinearLayout = Java.use("android.widget.LinearLayout");
var mainLayout = LinearLayout.$new(activity);

// Configure main layout orientation and background
mainLayout.setOrientation(LinearLayout.VERTICAL);  // 1
mainLayout.setBackgroundColor(0x80000000);  // semi-transparent black

// Add the overlay to window manager
wm.addView(mainLayout, params);

// Store references for later access
_0x39f4b2 = mainLayout;  // main layout
_0x2e7801 = params;      // layout parameters
```

### Step 2.3: Main Menu Layout Construction

**Line 181-200: Title View Creation**
```java
// Create title TextView
var TextView = Java.use("android.widget.TextView");
var Typeface = Java.use("android.graphics.Typeface");

var titleView = TextView.$new(activity);

// Configure title appearance
titleView.setText("BSRE Dev Version");
titleView.setTextColor(0xFF00FF00);  // Green
titleView.setTextSize(18.0);         // 18 scaled pixels
titleView.setGravity(Gravity.CENTER); // 0x11
titleView.setTypeface(Typeface.DEFAULT_BOLD);
titleView.setPadding(0, 20, 0, 20);  // top and bottom padding

// Create layout parameters for title
var LayoutParams = Java.use("android.widget.LinearLayout$LayoutParams");
var titleParams = LayoutParams.$new(
    LayoutParams.MATCH_PARENT,  // width = -1
    LayoutParams.WRAP_CONTENT   // height = -2
);
titleParams.gravity = Gravity.CENTER_HORIZONTAL;

// Add title to main layout  
mainLayout.addView(titleView, titleParams);
```

**Line 201-250: Cheat Option Creation Loop**

This creates each cheat option (dodge, autoshoot, xray, aimbot, spin modifier):

```java
// Array of cheat options
var cheatOptions = ["enable dodge", "autoshoot", "xray", "aimbot", "spin modifier"];

for (var i = 0; i < cheatOptions.length; i++) {
    var optionName = cheatOptions[i];
    
    // Create horizontal container for this option
    var optionLayout = LinearLayout.$new(activity);
    optionLayout.setOrientation(LinearLayout.HORIZONTAL);
    optionLayout.setBackgroundColor(0x40000000);  // dark semi-transparent
    optionLayout.setPadding(32, 16, 32, 16);      // left, top, right, bottom
    
    // Create toggle switch
    var Switch = Java.use("android.widget.Switch");
    var toggle = Switch.$new(activity);
    toggle.setChecked(false);
    toggle.setId(0x7F0A0001 + i);  // Generate unique ID
    
    // Create layout parameters for toggle
    var toggleParams = LayoutParams.$new(
        LayoutParams.WRAP_CONTENT,  // width
        LayoutParams.WRAP_CONTENT   // height  
    );
    toggleParams.gravity = Gravity.CENTER_VERTICAL;
    toggleParams.setMargins(0, 0, 16, 0);  // 16dp right margin
    
    // Create label
    var label = TextView.$new(activity);
    label.setText(optionName);
    label.setTextColor(0xFFFFFFFF);  // white
    label.setTextSize(16.0);         // 16sp
    label.setTypeface(Typeface.DEFAULT);
    
    // Create layout parameters for label  
    var labelParams = LayoutParams.$new(
        0,                          // width = 0 (use weight)
        LayoutParams.WRAP_CONTENT,  // height
        1.0f                        // weight = 1.0 (take remaining space)
    );
    labelParams.gravity = Gravity.CENTER_VERTICAL;
    
    // Set up toggle listener
    var OnCheckedChangeListener = Java.use("android.widget.CompoundButton$OnCheckedChangeListener");
    var listener = OnCheckedChangeListener.$new();
    
    listener.onCheckedChanged.implementation = function(buttonView, isChecked) {
        // Handle cheat toggle
        handleCheatToggle(optionName, isChecked);
    };
    
    toggle.setOnCheckedChangeListener(listener);
    
    // Add views to option layout
    optionLayout.addView(toggle, toggleParams);
    optionLayout.addView(label, labelParams);
    
    // Add option layout to main layout
    mainLayout.addView(optionLayout);
}
```

## PHASE 3: MEMORY HOOKING INFRASTRUCTURE

### Step 3.1: Function Hook Deployment

**Line 251-300: handleAutoShoot Hook**

```asm
; Original function at 0x4046CB38:
; 55 48 89 E5 41 57 41 56 49 89 D6 48 83 EC 20 4C

; Step 1: Allocate trampoline memory
mov rax, 9              ; mmap syscall
mov rdi, 0              ; NULL address
mov rsi, 4096           ; 4KB
mov rdx, 0x7            ; PROT_READ|PROT_WRITE|PROT_EXEC
mov r10, 0x22           ; MAP_PRIVATE|MAP_ANONYMOUS
mov r8, -1              ; no fd
mov r9, 0               ; offset 0
syscall
; RAX = 0x7FE00000 (trampoline memory)

; Step 2: Copy original function bytes to trampoline
mov rsi, 0x4046CB38     ; original function
mov rdi, 0x7FE00200     ; trampoline location
mov rcx, 14             ; bytes to copy
rep movsb               ; copy original bytes

; Step 3: Add jump back to original function + 14
mov byte [rdi], 0x48    ; MOV RAX, 0x4046CB46
mov byte [rdi+1], 0xB8
mov qword [rdi+2], 0x4046CB46
mov word [rdi+10], 0xE0FF ; JMP RAX

; Step 4: Create hook bytes
; JMP relative to our handler at 0x40702100
; Calculate relative jump: 0x40702100 - (0x4046CB38 + 5) = 0x255C3
mov rdi, 0x4046CB38     ; original function
mov byte [rdi], 0xE9    ; JMP opcode
mov dword [rdi+1], 0x255C3 ; relative offset
mov qword [rdi+5], 0x9090909090909090 ; NOP padding

; Step 5: Update trampoline table
mov rax, 0x7FE01000     ; trampoline_table[0]
mov qword [rax], 0x4046CB38    ; original_addr
mov qword [rax+8], 0x7FE00200  ; trampoline_addr
; Store original bytes: 55 48 89 E5 41 57 41 56 49 89 D6 48 83 EC
mov rsi, 0x4046CB38
mov rdi, rax
add rdi, 16
mov rcx, 14
rep movsb
```

## PHASE 4: AUTO-SHOOT SYSTEM

### Step 4.1: Player State Monitoring

**Line 301-350: Get Player Information**

```c
// Called every 16ms (60 times per second)
void updateAutoShoot() {
    // Get local player index
    int player_index = call_function(0x40362DC4); // GetOwnPlayerIndex()
    
    // Calculate player object pointer
    uint64_t player_ptr = 0x419D1E1C + (player_index * 0x268);
    
    // Read player position
    float player_x = read_float(player_ptr + 0x40);
    float player_y = read_float(player_ptr + 0x44); 
    float player_z = read_float(player_ptr + 0x48);
    
    // Read player team
    uint8_t player_team = read_byte(player_ptr + 0x34);
    
    // Read player health
    int player_health = read_int(player_ptr + 0x50);
    int max_health = read_int(player_ptr + 0x54);
    
    // Read weapon state
    float weapon_cooldown = read_float(player_ptr + 0xE0);
    bool can_shoot = (weapon_cooldown <= 0.0f);
    
    // Store in global player state
    _0x32b402.player_x = player_x;
    _0x32b402.player_y = player_y;
    _0x32b402.player_team = player_team;
    _0x32b402.can_shoot = can_shoot;
}
```

**Line 351-400: Enemy Scanning Algorithm**

```c
void scanEnemies() {
    Target best_target = {0};
    float best_score = 0.0f;
    
    // Scan all 10 player slots
    for (int i = 0; i < 10; i++) {
        uint64_t enemy_ptr = 0x419D1E1C + (i * 0x268);
        
        // Read enemy team
        uint8_t enemy_team = read_byte(enemy_ptr + 0x34);
        
        // Skip if same team or invalid
        if (enemy_team == _0x32b402.player_team || enemy_team == 0xFF) {
            continue;
        }
        
        // Read enemy position
        float enemy_x = read_float(enemy_ptr + 0x40);
        float enemy_y = read_float(enemy_ptr + 0x44);
        float enemy_z = read_float(enemy_ptr + 0x48);
        
        // Read enemy health
        int enemy_health = read_int(enemy_ptr + 0x50);
        
        // Skip if dead
        if (enemy_health <= 0) {
            continue;
        }
        
        // Calculate distance
        float dx = enemy_x - _0x32b402.player_x;
        float dy = enemy_y - _0x32b402.player_y;
        float distance = sqrtf(dx*dx + dy*dy);
        
        // Check if in range
        float casting_range = getCastingRange(_0x32b402.player_ptr);
        if (distance > casting_range) {
            continue;
        }
        
        // Calculate target score
        float score = calculateTargetScore(i, enemy_x, enemy_y, enemy_health, distance);
        
        // Update best target if better score
        if (score > best_score) {
            best_score = score;
            best_target.index = i;
            best_target.x = enemy_x;
            best_target.y = enemy_y;
            best_target.health = enemy_health;
            best_target.distance = distance;
        }
    }
    
    // Store best target
    _0x32b402.best_target = best_target;
}
```

**Line 401-450: Target Scoring Algorithm**

```c
float calculateTargetScore(int enemy_index, float x, float y, int health, float distance) {
    float score = 0.0f;
    
    // Base score from distance (closer = better)
    score += (1000.0f / distance);
    
    // Health modifier (lower health = better)
    score *= (100.0f / health);
    
    // Check if enemy is using ultimate
    bool is_ulting = read_byte(enemy_ptr + 0xF4) > 0;
    if (is_ulting) {
        score *= 1.5f;  // Prioritize ulting enemies
    }
    
    // Check if enemy is low health
    if (health < 1000) {
        score *= 2.0f;  // Finish low health enemies
    }
    
    // Check if enemy is aiming at us
    float aim_x = read_float(enemy_ptr + 0x80);
    float aim_y = read_float(enemy_ptr + 0x84);
    float aim_angle = atan2f(aim_y, aim_x);
    float to_player_angle = atan2f(_0x32b402.player_y - y, _0x32b402.player_x - x);
    float angle_diff = fabsf(aim_angle - to_player_angle);
    
    if (angle_diff < 0.5f) {  // ~28 degrees
        score *= 1.3f;  // Enemy is aiming at us, higher priority
    }
    
    return score;
}
```

## ORIGINAL UNOBFUSCATED CODE EXAMPLE

Here's how the original unobfuscated code would have looked:

### Memory Scanner (Original)
```c
// Original clear function names and structure
void initialize_memory_scanner() {
    // Allocate scan buffer
    void* scan_buffer = mmap(NULL, 8192, PROT_READ|PROT_WRITE, 
                           MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
    
    // Set memory protection
    mprotect(scan_buffer, 8192, PROT_READ|PROT_WRITE);
    
    // Create mutex for thread safety
    pthread_mutex_t scan_mutex;
    pthread_mutex_init(&scan_mutex, NULL);
    
    // Open memory maps
    int maps_fd = open("/proc/self/maps", O_RDONLY);
    
    // Parse each line
    char line[256];
    while (read_line(maps_fd, line, sizeof(line))) {
        parse_maps_line(line);
    }
    
    close(maps_fd);
}
```

### GUI System (Original)
```java
// Clear Android GUI code
public void createOverlayMenu() {
    // Get current activity
    Activity activity = getCurrentActivity();
    
    // Create window parameters
    WindowManager.LayoutParams params = new WindowManager.LayoutParams(
        WindowManager.LayoutParams.MATCH_PARENT,
        WindowManager.LayoutParams.WRAP_CONTENT,
        WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,
        WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE |
        WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL,
        PixelFormat.TRANSLUCENT
    );
    
    // Create main layout
    LinearLayout mainLayout = new LinearLayout(activity);
    mainLayout.setOrientation(LinearLayout.VERTICAL);
    mainLayout.setBackgroundColor(0x80000000);
    
    // Create title
    TextView title = new TextView(activity);
    title.setText("Brawl Stars Cheat Menu");
    title.setTextColor(Color.GREEN);
    title.setTextSize(18);
    title.setGravity(Gravity.CENTER);
    mainLayout.addView(title);
    
    // Add to window manager
    WindowManager wm = activity.getWindowManager();
    wm.addView(mainLayout, params);
}
```

### Auto-Shoot System (Original)
```c
// Clear auto-shoot logic
void auto_shoot_update() {
    // Get local player
    Player* local_player = get_local_player();
    if (!local_player) return;
    
    // Find best target
    Player* best_target = find_best_target(local_player);
    if (!best_target) return;
    
    // Calculate aim angle
    Vector2f aim_vector = calculate_aim_vector(local_player, best_target);
    
    // Predict movement
    Vector2f predicted_aim = predict_target_movement(local_player, best_target, aim_vector);
    
    // Normalize aim coordinates
    Vector2f normalized_aim = normalize_aim_vector(predicted_aim);
    
    // Write to game memory
    write_aim_stick(normalized_aim.x, normalized_aim.y);
    
    // Auto shoot if ready
    if (can_shoot(local_player)) {
        trigger_auto_shoot();
    }
}
```
