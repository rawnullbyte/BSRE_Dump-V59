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
                                                    ; This preserves the caller's stack frame
                                                    ; Stack: [old_rbp] <- RSP points here

48 89 E5              mov rbp, rsp            ; Set RBP to current RSP value
                                                    ; This establishes our new stack frame
                                                    ; RBP now points to start of our frame

48 83 EC 40           sub rsp, 0x40           ; Allocate 64 bytes of stack space for local variables
                                                    ; This space will be used for:
                                                    ; - mutex object (8 bytes)
                                                    ; - file descriptor (8 bytes)  
                                                    ; - bytes read counter (8 bytes)
                                                    ; - temporary buffers (40 bytes)
                                                    ; RSP now at 0x7FFFFFFFDFB0

53                    push rbx                ; Save callee-saved register RBX
                                                    ; RBX must be preserved across function calls
                                                    ; Stack: [old_rbp][saved_rbx]

41 54                 push r12                ; Save callee-saved register R12
                                                    ; Stack: [old_rbp][saved_rbx][saved_r12]

41 55                 push r13                ; Save callee-saved register R13
                                                    ; Stack: [old_rbp][saved_rbx][saved_r12][saved_r13]

41 56                 push r14                ; Save callee-saved register R14  
                                                    ; Stack: [old_rbp][saved_rbx][saved_r12][saved_r13][saved_r14]

41 57                 push r15                ; Save callee-saved register R15
                                                    ; Stack now fully prepared for function execution
                                                    ; All callee-saved registers preserved
```

**Line 11-20: 8KB Scan Buffer Allocation via mmap**
```asm
; =============================================
; ALLOCATE 8KB SCAN BUFFER USING MMAP SYSCALL
; =============================================
; This buffer will store module information, symbol tables, and scan results
; We need executable memory because we'll be writing trampoline code here later

48 C7 C0 09 00 00 00  mov rax, 9              ; Set RAX to 9 - this is the syscall number for mmap
                                                    ; Linux syscall convention: RAX contains syscall number
                                                    ; mmap is syscall 9 on x86_64

48 C7 C7 00 00 00 00  mov rdi, 0              ; Set RDI to 0 (NULL) - let kernel choose address
                                                    ; First argument: desired address (NULL = any)
                                                    ; Kernel will pick a suitable virtual address

48 C7 C6 00 20 00 00  mov rsi, 0x2000         ; Set RSI to 0x2000 (8192 bytes = 8KB)
                                                    ; Second argument: size of mapping
                                                    ; 8KB is enough for module table + symbol cache

48 C7 C2 03 00 00 00  mov rdx, 3              ; Set RDX to 3 (PROT_READ | PROT_WRITE)
                                                    ; Third argument: memory protection flags
                                                    ; PROT_READ (1) | PROT_WRITE (2) = 3
                                                    ; We need write access to store scan results

49 C7 C2 22 00 00 00  mov r10, 0x22           ; Set R10 to 0x22 (MAP_PRIVATE | MAP_ANONYMOUS)
                                                    ; Fourth argument: mapping flags
                                                    ; MAP_PRIVATE (0x02) - changes are private
                                                    ; MAP_ANONYMOUS (0x20) - not file-backed
                                                    ; Combined: 0x02 | 0x20 = 0x22

49 C7 C0 FF FF FF FF  mov r8, -1              ; Set R8 to -1 (no file descriptor)
                                                    ; Fifth argument: file descriptor
                                                    ; -1 because this is anonymous memory

4D 31 C9              xor r9, r9              ; Set R9 to 0 (offset = 0)
                                                    ; Sixth argument: offset in file
                                                    ; Zero for anonymous mappings

0F 05                 syscall                 ; Execute the mmap syscall
                                                    ; This switches to kernel mode
                                                    ; Kernel allocates 8KB of memory
                                                    ; Returns address in RAX on success

; After syscall returns:
; RAX now contains the allocated memory address (e.g., 0x7FFD1000)
; This is our scan buffer base address
; Memory range 0x7FFD1000 - 0x7FFD3000 is now allocated to us
```

**Line 21-30: Memory Protection Setting**
```asm
; =============================================
; SET MEMORY PROTECTION FOR SCAN BUFFER
; =============================================
; Even though mmap already set protections, we explicitly set them again
; This ensures we have the correct permissions and serves as a verification step

48 C7 C0 0A 00 00 00  mov rax, 10             ; Set RAX to 10 - syscall number for mprotect
                                                    ; mprotect changes memory protection flags

48 BF 00 10 FD 7F 00  mov rdi, 0x7FFD1000     ; Set RDI to our allocated buffer address
00 00 00                                      ; First argument: memory address to protect
                                                    ; This is the start of our 8KB buffer

48 C7 C6 00 20 00 00  mov rsi, 0x2000         ; Set RSI to 0x2000 (8192 bytes)
                                                    ; Second argument: length of region to protect
                                                    ; Same size we allocated with mmap

48 C7 C2 03 00 00 00  mov rdx, 3              ; Set RDX to 3 (PROT_READ | PROT_WRITE)
                                                    ; Third argument: protection flags
                                                    ; Same as mmap: readable and writable

0F 05                 syscall                 ; Execute the mprotect syscall
                                                    ; Kernel updates page table entries
                                                    ; Now our buffer definitely has RW permissions

; Why do we call mprotect even though mmap already set permissions?
; 1. Verification - ensures permissions are actually set correctly
; 2. Defense in depth - some systems might have different default permissions
; 3. Code clarity - makes it explicit what permissions we want
```

**Line 31-40: Mutex Creation for Thread Safety**
```asm
; =============================================
; CREATE MUTEX FOR THREAD SAFETY
; =============================================
; Since this cheat runs in parallel with the game, we need synchronization
; The mutex protects against race conditions when multiple threads access shared data

48 8D 7D F8           lea rdi, [rbp-0x8]      ; Load Effective Address of mutex storage
                                                    ; RBP is our frame pointer (0x7FFFFFFFDFF0)
                                                    ; RBP-8 = 0x7FFFFFFFDFE8 (mutex location)
                                                    ; This is where we'll store the mutex object
                                                    ; LEA computes address, doesn't dereference

48 31 F6              xor rsi, rsi            ; Set RSI to 0 (NULL attributes)
                                                    ; Second argument: pthread_mutexattr_t*
                                                    ; NULL means use default attributes
                                                    ; XOR is smaller than MOV RSI, 0

48 B8 20 10 40 00 00  mov rax, 0x401020       ; Load address of pthread_mutex_init
00 00 00                                      ; This is the Procedure Linkage Table (PLT) entry
                                                    ; PLT handles dynamic linking to libc
                                                    ; 0x401020 is the PLT slot for pthread_mutex_init

FF D0                 call rax                ; Call pthread_mutex_init(&mutex, NULL)
                                                    ; This initializes the mutex at [RBP-8]
                                                    ; Mutex is now ready for use

; The mutex is recursive (PTHREAD_MUTEX_RECURSIVE) by default on most systems
; This means the same thread can lock it multiple times without deadlock
; Important because our code might call functions that need the mutex recursively
```

**Line 41-50: Global Variable Storage**
```asm
; =============================================
; STORE CRITICAL REFERENCES IN GLOBAL VARIABLES
; =============================================
; We need to store key addresses so other functions can access them
; Global variables are stored in our allocated memory region

48 B8 00 50 FD 7F 00  mov rax, 0x7FFD5000     ; Load global variable section address
00 00 00                                      ; This is 4KB into our allocated memory
                                                    ; 0x7FFD1000 + 0x4000 = 0x7FFD5000
                                                    ; We use this area for global variables

48 8B 55 F8           mov rdx, [rbp-0x8]      ; Load mutex pointer from stack
                                                    ; RBP-8 = 0x7FFFFFFFDFE8 (mutex location)
                                                    ; RDX now contains the mutex address

48 89 10              mov [rax], rdx          ; Store mutex pointer in global variable _0x89389a
                                                    ; [0x7FFD5000] = mutex address
                                                    ; Other functions can now access the mutex

; Set initialization flags to track what's been initialized
C6 40 04 01           mov byte [rax+4], 1     ; Set scanner_initialized flag to true (1)
                                                    ; [0x7FFD5004] = 1
                                                    ; This acts as a safety check - other functions
                                                    ; can verify the scanner is ready before use

48 C7 40 08 00 10 FD  mov qword [rax+8], 0x7FFD1000 ; Store scan buffer pointer
7F                                              ; [0x7FFD5008] = 0x7FFD1000
                                                    ; This is the base of our 8KB scan buffer

48 C7 40 10 00 30 FD  mov qword [rax+0x10], 0x7FFD3000 ; Store read buffer pointer
7F                                              ; [0x7FFD5010] = 0x7FFD3000
                                                    ; This is our 4KB read buffer for file I/O

; Now we have:
; _0x89389a[0] = mutex pointer
; _0x89389a[4] = initialization flag (1 = ready)
; _0x89389a[8] = scan buffer base (0x7FFD1000)
; _0x89389a[16] = read buffer base (0x7FFD3000)
```

### Step 1.2: Process Memory Mapping Analysis

**Line 51-60: Open /proc/self/maps**
```asm
; =============================================
; OPEN /proc/self/maps FOR READING
; =============================================
; /proc/self/maps shows the memory layout of our current process
; This file contains all memory regions: code, data, heap, stack, libraries

48 C7 C0 02 00 00 00  mov rax, 2              ; Set RAX to 2 - syscall number for open
                                                    ; open() opens a file and returns a file descriptor

48 BF 00 34 01 40 00  mov rdi, 0x4013400      ; Set RDI to address of filename string
00 00 00                                      ; First argument: path to open
                                                    ; 0x4013400 contains "/proc/self/maps\0"
                                                    ; This is in the code section (read-only)

48 31 F6              xor rsi, rsi            ; Set RSI to 0 (O_RDONLY)
                                                    ; Second argument: open flags
                                                    ; O_RDONLY = 0 (open for reading only)
                                                    ; We only need to read, not write

0F 05                 syscall                 ; Execute the open syscall
                                                    ; Kernel opens /proc/self/maps
                                                    ; Returns file descriptor in RAX on success

48 89 45 F0           mov [rbp-0x10], rax     ; Store file descriptor on stack at RBP-0x10
                                                    ; [0x7FFFFFFFDFE0] = file descriptor (e.g., 5)
                                                    ; We'll use this FD for subsequent read operations

; Example content of /proc/self/maps:
; 7F8A1000-7F8B2000 r-xp 00000000 08:01 284761 /system/lib/libc.so
; 40000000-40100000 r-xp 00000000 08:01 284762 /data/app/libgame.so
; 50000000-50100000 r-xp 00000000 08:01 284763 /data/app/libBrawlStars.so
; 60000000-60100000 rw-p 00000000 00:00 0 [heap]
; 7FFFFFFDE000-7FFFFFFE0000 rw-p 00000000 00:00 0 [stack]
```

**Line 61-80: Read Maps File Line by Line**
```asm
; =============================================
; READ MEMORY MAPS FILE INTO BUFFER
; =============================================
; We read the entire maps file into our buffer for parsing
; The file is typically 2-4KB, so one read should be sufficient

; First read() syscall to read file content
mov rax, 0              ; Set RAX to 0 - syscall number for read
                                ; read() reads data from a file descriptor

mov rdi, [rbp-0x10]     ; Set RDI to our file descriptor (5)
                                ; First argument: file descriptor to read from
                                ; This is the FD we stored earlier

mov rsi, 0x7FFD3000     ; Set RSI to our read buffer address (0x7FFD3000)
                                ; Second argument: buffer to read into
                                ; This is our 4KB read buffer allocated earlier

mov rdx, 4096           ; Set RDX to 4096 (bytes to read)
                                ; Third argument: number of bytes to read
                                ; We try to read the entire file in one go

syscall                 ; Execute the read syscall
                                ; Kernel reads data from /proc/self/maps into our buffer
                                ; Returns number of bytes read in RAX

mov [rbp-0x18], rax     ; Store bytes read count on stack at RBP-0x18
                                ; [0x7FFFFFFFDFD8] = bytes read (e.g., 1024)
                                ; This tells us how much data we actually got

; Now parse the first line in the buffer
mov rdi, 0x7FFD3000     ; Set RDI to start of read buffer
                                ; This is where the maps file content starts
                                ; RDI will be first argument to parse_maps_line

call parse_maps_line    ; Call our parsing function
                                ; This function extracts information from one line
                                ; and adds it to our module table
```

**The parse_maps_line function in detail:**
```c
// =============================================
// PARSE ONE LINE FROM /proc/self/maps
// =============================================
// Input: char* line - pointer to a null-terminated line from maps file
// Output: Adds module information to global module_table
// Example line: "7F8A1000-7F8B2000 r-xp 00000000 08:01 284761 /system/lib/libc.so"

void parse_maps_line(char* line) {
    // Step 1: Find the dash separating start and end addresses
    char* dash = strchr(line, '-');  // Find first '-' character in the line
    if (!dash) return;               // If no dash found, line is malformed
    
    *dash = '\0';                    // Replace '-' with null terminator
                                     // Now "7F8A1000" is a separate string
                                     // line points to "7F8A1000\0"
    
    // Step 2: Extract start address (hex string to integer)
    uint64_t start_addr = strtoul(line, NULL, 16);  // Convert "7F8A1000" to 0x7F8A1000
                                                     // Base 16 because it's hexadecimal
                                                     // NULL means we don't care about end pointer
    
    // Step 3: Move past the dash we null-terminated
    char* rest = dash + 1;           // rest now points to "7F8B2000 r-xp ..."
    
    // Step 4: Find space after end address  
    char* space = strchr(rest, ' '); // Find first space character
    if (!space) return;              // If no space found, line is malformed
    
    *space = '\0';                   // Replace space with null terminator
                                     // Now "7F8B2000" is a separate string
                                     // rest points to "7F8B2000\0"
    
    // Step 5: Extract end address
    uint64_t end_addr = strtoul(rest, NULL, 16);  // Convert "7F8B2000" to 0x7F8B2000
    
    // Step 6: Calculate memory region size
    uint64_t size = end_addr - start_addr;  // 0x7F8B2000 - 0x7F8A1000 = 0x11000 (69632 bytes)
    
    // Step 7: Move past the space to permissions field
    rest = space + 1;                // rest now points to "r-xp 00000000 ..."
    
    // Step 8: Extract permissions (next 4 characters)
    space = strchr(rest, ' ');       // Find space after permissions
    if (!space) return;
    
    *space = '\0';                   // Null terminate permissions
    char permissions[5];
    strncpy(permissions, rest, 4);   // Copy "r-xp" to permissions buffer
    permissions[4] = '\0';           // Ensure null termination
    
    // Step 9: Parse permissions into bitmask
    uint8_t perm_bits = 0;
    if (permissions[0] == 'r') perm_bits |= 0x4;  // Readable
    if (permissions[1] == 'w') perm_bits |= 0x2;  // Writable  
    if (permissions[2] == 'x') perm_bits |= 0x1;  // Executable
    if (permissions[3] == 'p') perm_bits |= 0x8;  // Private (vs shared)
    
    // Continue parsing offset, device, inode, and pathname...
    // This process repeats for each field in the line
    
    // Step 10: Add to module table if it's a file-backed mapping (has pathname)
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
; This table will be used later to locate game functions

; Store libc.so information in module_table[0]
mov rax, 0x7FFD1100          ; module_table[0] base address
                                  ; 0x7FFD1000 (scan buffer) + 0x100 (first entry)
                                  ; Each module entry is 256 bytes
                                  ; Total table size: 256 * 50 modules = 12.8KB

mov rdx, 0x7F8A1000          ; libc.so base address from parsing
mov [rax], rdx               ; Store base_address at offset 0
                                  ; module_table[0].base_address = 0x7F8A1000
                                  ; This is the start address of libc in memory

mov rdx, 0x7F8B2000          ; libc.so end address  
mov [rax+8], rdx             ; Store end_address at offset 8
                                  ; module_table[0].end_address = 0x7F8B2000
                                  ; This is the end address of libc in memory

mov rdx, 0x11000             ; libc.so size (0x7F8B2000 - 0x7F8A1000)
mov [rax+16], rdx            ; Store size at offset 16
                                  ; module_table[0].size = 0x11000 (69,632 bytes)
                                  ; This tells us how big the library is

mov byte [rax+24], 0x7       ; Store permissions: r-xp = 0x7
                                  ; Read (4) + Execute (1) + Private (2) = 7
                                  ; module_table[0].permissions = 0x7
                                  ; 'r-xp' means: readable, executable, private mapping

mov dword [rax+28], 0        ; Store offset: 0x00000000
                                  ; module_table[0].offset = 0
                                  ; This is the offset in the file where mapping starts
                                  ; 0 means it starts from beginning of the file

mov byte [rax+32], 8         ; Store device major number: 8
                                  ; module_table[0].device_major = 8
                                  ; Major number 8 usually means SCSI disk devices

mov byte [rax+33], 1         ; Store device minor number: 1  
                                  ; module_table[0].device_minor = 1
                                  ; Minor number identifies specific device

mov dword [rax+34], 284761   ; Store inode number: 284761
                                  ; module_table[0].inode = 284761
                                  ; Inode is the file's identifier in the filesystem

; Copy module name "/system/lib/libc.so" to name field
lea rdi, [rax+40]            ; Load address of name field (offset 40)
                                  ; RDI = 0x7FFD1128 (name field start)
                                  ; Name field is 200 bytes long (enough for long paths)

mov rsi, 0x4013500           ; Load address of string "/system/lib/libc.so"
                                  ; This string is stored in our code section
                                  ; It was placed there at compile time

call strcpy                  ; Copy string from [RSI] to [RDI]
                                  ; Copies "/system/lib/libc.so" to module_table[0].name
                                  ; strcpy copies until null terminator is found
                                  ; Destination: 0x7FFD1128
                                  ; Source: 0x4013500

; Now repeat for libgame.so in module_table[1]
mov rax, 0x7FFD1200          ; module_table[1] base address
                                  ; 0x7FFD1100 + 0x100 = 0x7FFD1200 (second entry)

mov rdx, 0x40000000          ; libgame.so base address
mov [rax], rdx               ; module_table[1].base_address = 0x40000000

mov rdx, 0x40100000          ; libgame.so end address
mov [rax+8], rdx             ; module_table[1].end_address = 0x40100000

mov rdx, 0x100000            ; libgame.so size (1MB)
mov [rax+16], rdx            ; module_table[1].size = 0x100000

mov byte [rax+24], 0x7       ; permissions: r-xp
mov dword [rax+28], 0        ; offset: 0

; Copy name "libgame.so"
lea rdi, [rax+40]            ; name field at 0x7FFD1228
mov rsi, 0x4013600           ; "libgame.so" string address
call strcpy

; Continue for all 47 modules found...
; This includes:
; - libBrawlStars.so (main game logic)
; - libunity.so (Unity engine)
; - libmono.so (Mono runtime)
; - Various system libraries
; - [heap] and [stack] regions (anonymous mappings)

; After processing all modules, we have complete memory map:
; module_table[0] = libc.so @ 0x7F8A1000
; module_table[1] = libgame.so @ 0x40000000  
; module_table[2] = libBrawlStars.so @ 0x50000000
; ...
; module_table[46] = [stack] @ 0x7FFFFFFDE000

; Set module count in global variable
mov rax, 0x7FFD5000          ; Global variable section
mov dword [rax+0x20], 47     ; Store module_count = 47
                                  ; [0x7FFD5020] = 47
                                  ; Other functions can check how many modules we found
```

## PHASE 2: ANDROID GUI SYSTEM

### Step 2.1: Android Context Acquisition

**Line 101-120: Java Environment Setup**
```java
// =============================================
// HOOK INTO ANDROID JAVA ENVIRONMENT USING FRIDA
// =============================================
// Frida allows us to inject JavaScript into the Java VM
// This gives us access to Android APIs and the game's Java classes

Java.perform(function() {
    // Java.perform() ensures we're running in the context of the Java VM
    // This function is called when Frida injects into the process
    // All Java operations must happen inside this callback
    
    // Get the ActivityThread class reference
    // ActivityThread is the main thread of an Android application
    // It manages the application's activities and services
    var ActivityThread = Java.use("android.app.ActivityThread");
    // Java.use() gets a JavaScript wrapper for a Java class
    // This allows us to call static methods and access fields
    
    // Get the current activity thread instance
    // currentActivityThread() is a static method that returns the singleton instance
    var currentThread = ActivityThread.currentActivityThread();
    // currentThread now contains the ActivityThread object for our process
    // This is the main thread that manages the Brawl Stars app
    
    // Get the activities map from the activity thread
    // mActivities is a HashMap containing all active activities
    var activities = currentThread.mActivities.value;
    // .value gets the actual Java object from the Frida wrapper
    // activities is now a Java HashMap<IBinder, ActivityClientRecord>
    
    // Get the first activity record from the map
    // We assume Brawl Stars has at least one active activity
    var activityRecord = null;
    var iterator = activities.values().iterator();
    // Get an iterator to traverse all values in the HashMap
    // activities.values() returns a Collection of ActivityClientRecord objects
    
    if (iterator.hasNext()) {
        activityRecord = iterator.next();
        // Get the first activity record from the iterator
        // This should be the main Brawl Stars activity
    }
    
    // Extract the actual Activity object from the activity record
    // ActivityClientRecord.activity field contains the Activity instance
    var activity = activityRecord.activity.value;
    // activity now contains the main Brawl Stars Activity object
    // We can use this to create views and access window manager
    
    // Store activity reference in global variable for later use
    _0x5d878e = activity;
    // This global variable will be used by other functions to access the activity
    // Without having to traverse the ActivityThread again
});
```

**Line 121-140: Window Manager Acquisition**
```java
// =============================================
// GET WINDOW MANAGER AND DISPLAY INFORMATION
// =============================================
// The WindowManager allows us to create overlay windows
// Display metrics give us screen size and density information

// Get the window manager from the activity
// getWindowManager() returns the WindowManager for this activity's window
var wm = activity.getWindowManager();
// wm is now the WindowManager that manages the game's window
// We'll use this to add our overlay view later

// Get display metrics for screen dimensions
// We need to know the screen size to position our overlay correctly
var display = wm.getDefaultDisplay();
// display contains information about the physical display
// This includes size, rotation, refresh rate, etc.

var metrics = Java.use("android.util.DisplayMetrics").$new();
// Create a new DisplayMetrics object
// DisplayMetrics holds information about screen density and scaling
// $new() creates a new instance of the Java class

display.getMetrics(metrics);
// Populate the metrics object with information from the display
// After this call, metrics contains:
// - widthPixels: screen width in pixels
// - heightPixels: screen height in pixels  
// - density: scaling factor (1.0 = mdpi, 1.5 = hdpi, 2.0 = xhdpi, etc.)
// - densityDpi: density in dpi (160 = mdpi, 240 = hdpi, 320 = xhdpi, etc.)

// Store screen information in global variables for later use
_0x32b402.screen_width = metrics.widthPixels;    // e.g., 1080 pixels
_0x32b402.screen_height = metrics.heightPixels;  // e.g., 2340 pixels (including status bar)
_0x32b402.screen_density = metrics.density;      // e.g., 3.0 (xxxhdpi device)
_0x32b402.density_dpi = metrics.densityDpi;      // e.g., 480 dpi

// Calculate usable screen height (excluding status bar and navigation bar)
// We need this to position our overlay without overlapping system UI
var resourceId = activity.getResources().getIdentifier("status_bar_height", "dimen", "android");
if (resourceId > 0) {
    var statusBarHeight = activity.getResources().getDimensionPixelSize(resourceId);
    _0x32b402.usable_height = metrics.heightPixels - statusBarHeight;
} else {
    _0x32b402.usable_height = metrics.heightPixels - 100; // Fallback estimate
}

// Log screen information for debugging
console.log("Screen: " + metrics.widthPixels + "x" + metrics.heightPixels + 
           ", Density: " + metrics.density + ", DPI: " + metrics.densityDpi);
// This helps verify we're getting correct screen information
// Example output: "Screen: 1080x2340, Density: 3.0, DPI: 480"
```

### Step 2.2: Overlay Window Creation

**Line 141-160: Window Layout Parameters Construction**
```java
// =============================================
// CREATE WINDOW LAYOUT PARAMETERS FOR OVERLAY
// =============================================
// WindowManager.LayoutParams defines how our overlay window behaves
// This includes size, position, type, flags, and appearance

// Import required Android classes
var WindowManager = Java.use("android.view.WindowManager");
var LayoutParams = Java.use("android.view.WindowManager$LayoutParams");
var Gravity = Java.use("android.view.Gravity");
var PixelFormat = Java.use("android.graphics.PixelFormat");

// Create a new LayoutParams instance
// We use $new() to instantiate the Java class from JavaScript
var params = LayoutParams.$new();
// params is now a WindowManager.LayoutParams object with default values
// We need to configure it for our overlay window

// Configure window dimensions
params.width = LayoutParams.MATCH_PARENT;    // Width: -1 (fill entire screen width)
                                                    // MATCH_PARENT constant = -1
                                                    // Our overlay will span the full screen width

params.height = LayoutParams.WRAP_CONTENT;   // Height: -2 (wrap to content height)
                                                    // WRAP_CONTENT constant = -2  
                                                    // Height will adjust based on menu content

// Set window type - this is critical for overlay behavior
params.type = LayoutParams.TYPE_APPLICATION_OVERLAY;  // Type: 2038
                                                    // TYPE_APPLICATION_OVERLAY = 2038 (API 26+)
                                                    // This type allows overlay over other apps
                                                    // Requires SYSTEM_ALERT_WINDOW permission

// Set window flags - control window behavior and interaction
params.flags = LayoutParams.FLAG_NOT_FOCUSABLE |      // Flag: 0x00000008
               LayoutParams.FLAG_NOT_TOUCH_MODAL |    // Flag: 0x00000020  
               LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH; // Flag: 0x00040000
                                                    // FLAG_NOT_FOCUSABLE: window can't receive input focus
                                                    // FLAG_NOT_TOUCH_MODAL: touches outside go to windows behind
                                                    // FLAG_WATCH_OUTSIDE_TOUCH: receive touch events outside window
                                                    // Combined: 0x00000008 | 0x00000020 | 0x00040000 = 0x00042028

// Set pixel format for transparency
params.format = PixelFormat.TRANSLUCENT;     // Format: -3
                                                    // TRANSLUCENT constant = -3
                                                    // Allows transparent and semi-transparent pixels
                                                    // Essential for our semi-transparent overlay background

// Set window gravity (positioning)
params.gravity = Gravity.TOP | Gravity.START; // Gravity: 0x00000033
                                                    // TOP constant = 0x00000030
                                                    // START constant = 0x00000003  
                                                    // Combined: 0x00000030 | 0x00000003 = 0x00000033
                                                    // Positions window at top-left corner

// Set initial window position
params.x = 0;                               // X offset: 0 pixels from left
params.y = 100;                             // Y offset: 100 pixels from top
                                                    // This positions our overlay 100px below top edge
                                                    // Avoids overlapping with status bar

// Set window transparency (alpha)
params.alpha = 0.95f;                       // Alpha: 0.95 (95% opaque)
                                                    // Makes overlay slightly transparent
                                                    // Allows seeing game content behind menu

// Set window animations (none for stealth)
params.windowAnimations = 0;                // Animation style: 0 (no animation)
                                                    // Window appears/disappears instantly
                                                    // Makes overlay less noticeable

// Store layout parameters in global variable
_0x2e7801 = params;                         // Save for later use when adding view
```

**Line 161-180: Window Manager Addition**
```java
// =============================================
// CREATE MAIN OVERLAY LAYOUT AND ADD TO WINDOW
// =============================================
// Now we create the actual overlay view and add it to the window manager
// This makes our cheat menu visible over the game

// Create the main overlay layout
// LinearLayout arranges views in a single direction (vertical or horizontal)
var LinearLayout = Java.use("android.widget.LinearLayout");
var mainLayout = LinearLayout.$new(activity);
// mainLayout is now a new LinearLayout instance
// We'll add all our cheat menu items to this layout

// Configure main layout orientation and appearance
mainLayout.setOrientation(LinearLayout.VERTICAL);  // Orientation: 1 (vertical)
                                                    // VERTICAL constant = 1
                                                    // Child views will be stacked vertically
                                                    // This creates a list-like menu structure

mainLayout.setBackgroundColor(0x80000000);  // Background: semi-transparent black
                                                    // Color format: 0xAARRGGBB
                                                    // 0x80 = 50% alpha (transparent)
                                                    // 0x000000 = black color
                                                    // Combined: semi-transparent black background

// Set layout parameters for the main layout
var ViewGroup = Java.use("android.view.ViewGroup");
var layoutParams = ViewGroup.LayoutParams.$new(
    ViewGroup.LayoutParams.MATCH_PARENT,    // Width: -1 (match parent)
    ViewGroup.LayoutParams.WRAP_CONTENT     // Height: -2 (wrap content)
);
mainLayout.setLayoutParams(layoutParams);
// This ensures the layout fills the width but only uses needed height

// Add the overlay to window manager
// This is the critical call that makes our overlay visible
wm.addView(mainLayout, params);
// wm.addView() adds our view to the window with the specified parameters
// The view becomes visible immediately over the game
// Game continues running normally behind our transparent overlay

// Store references for later access
_0x39f4b2 = mainLayout;  // Store main layout reference
                                // Other functions can add views to this layout
                                // Without having to recreate it

_0x2e7801 = params;      // Store layout parameters reference  
                                // We might need to modify parameters later
                                // For example, to move or resize the overlay

// Set initial visibility state
_0x32b402.overlay_visible = true;  // Track that overlay is now visible
_0x32b402.overlay_created = true;  // Mark that overlay creation completed

// Log successful overlay creation
console.log("Overlay created successfully");
console.log("Position: " + params.x + ", " + params.y);
console.log("Size: " + params.width + "x" + params.height);
// Example output: "Overlay created successfully"
//                 "Position: 0, 100" 
//                 "Size: -1x-2"
```

### Step 2.3: Main Menu Layout Construction

**Line 181-200: Title View Creation**
```java
// =============================================
// CREATE TITLE VIEW FOR CHEAT MENU
// =============================================
// The title appears at the top of the menu and shows the cheat version

// Create title TextView
// TextView displays text in the overlay
var TextView = Java.use("android.widget.TextView");
var Typeface = Java.use("android.graphics.Typeface");

var titleView = TextView.$new(activity);
// titleView is now a new TextView instance
// We'll configure it to display our menu title

// Configure title text content
titleView.setText("BSRE Dev Version");
// Set the text that appears in the title
// "BSRE" is probably the cheat name/author initials
// "Dev Version" indicates this is a development build

// Configure title text color (green for visibility)
titleView.setTextColor(0xFF00FF00);  // Color: bright green
                                            // Color format: 0xAARRGGBB
                                            // 0xFF = fully opaque (no transparency)
                                            // 0x00FF00 = green (full green, no red/blue)
                                            // Bright green stands out against game background

// Configure title text size
titleView.setTextSize(18.0);         // Text size: 18 scaled pixels
                                            // Android uses "scaled pixels" (sp) for text
                                            // This accounts for user's font size preferences
                                            // 18sp is large enough to be easily readable

// Configure title text alignment
titleView.setGravity(Gravity.CENTER); // Gravity: 0x11 (center)
                                            // CENTER constant = 0x11
                                            // Centers text horizontally and vertically
                                            // Makes title look professional and centered

// Configure title font style
titleView.setTypeface(Typeface.DEFAULT_BOLD); // Font: default bold
                                            // DEFAULT_BOLD is the system's bold typeface
                                            // Makes title stand out from other text

// Configure title padding (space around text)
titleView.setPadding(0, 20, 0, 20);  // Padding: left=0, top=20, right=0, bottom=20
                                            // Adds 20 pixels of space above and below text
                                            // Makes title visually separated from other elements

// Create layout parameters for title view
// LayoutParams control how the title is positioned within the parent layout
var LayoutParams = Java.use("android.widget.LinearLayout$LayoutParams");
var titleParams = LayoutParams.$new(
    LayoutParams.MATCH_PARENT,  // Width: -1 (fill available width)
    LayoutParams.WRAP_CONTENT   // Height: -2 (wrap to text height)
);
titleParams.gravity = Gravity.CENTER_HORIZONTAL; // Horizontal alignment: center
                                            // Centers the title within the available width

// Add title view to main layout
// This makes the title visible in our overlay
mainLayout.addView(titleView, titleParams);
// addView() adds the title as the first child of the main layout
// It will appear at the top of our cheat menu

// Store title reference for potential later use
_0x39f4b2_title = titleView;  // Save title view reference
                                     // We might want to update title text later
                                     // For example, to show enabled/disabled status

// Log title creation
console.log("Title view created: 'BSRE Dev Version'");
```

**Line 201-250: Cheat Option Creation Loop**

This creates each cheat option (dodge, autoshoot, xray, aimbot, spin modifier):

```java
// =============================================
// CREATE CHEAT OPTIONS WITH TOGGLE SWITCHES
// =============================================
// This loop creates one menu item for each cheat feature
// Each item has a toggle switch and descriptive label

// Array of cheat options to create
// These strings will appear as labels in the menu
var cheatOptions = ["enable dodge", "autoshoot", "xray", "aimbot", "spin modifier"];
// Each string describes a cheat feature:
// - "enable dodge": Automatic dodging of enemy projectiles
// - "autoshoot": Automatic aiming and shooting
// - "xray": See through walls and highlight enemies  
// - "aimbot": Perfect aiming assistance
// - "spin modifier": Character spinning (might be for evasion or memes)

// Store cheat option states in global array
_0x32b402.cheat_states = {};  // Object to track enabled/disabled states
                                     // Key: cheat name, Value: boolean (true=enabled)

// Loop through each cheat option
for (var i = 0; i < cheatOptions.length; i++) {
    var optionName = cheatOptions[i];
    // optionName is the current cheat option string
    // e.g., "enable dodge", "autoshoot", etc.
    
    // Initialize cheat state to disabled (false)
    _0x32b402.cheat_states[optionName] = false;
    // All cheats start disabled for safety
    // User must explicitly enable each one
    
    // =============================================
    // CREATE HORIZONTAL CONTAINER FOR THIS OPTION
    // =============================================
    // Each cheat option gets its own horizontal layout
    // This contains the toggle switch on left and label on right
    
    var optionLayout = LinearLayout.$new(activity);
    // Create new LinearLayout for this specific option
    
    optionLayout.setOrientation(LinearLayout.HORIZONTAL); // Orientation: 0 (horizontal)
                                                                     // HORIZONTAL constant = 0
                                                                     // Child views arranged left to right
    
    optionLayout.setBackgroundColor(0x40000000);  // Background: dark semi-transparent
                                                                     // 0x40 = 25% alpha (more transparent than main layout)
                                                                     // 0x000000 = black
                                                                     // Creates subtle separation between options
    
    optionLayout.setPadding(32, 16, 32, 16);      // Padding: left=32, top=16, right=32, bottom=16
                                                                     // Adds space inside the option container
                                                                     // 32dp left/right padding, 16dp top/bottom
                                                                     // dp = density-independent pixels (scales with screen density)
    
    // =============================================
    // CREATE TOGGLE SWITCH FOR ENABLE/DISABLE
    // =============================================
    // Switch widget allows user to toggle cheat on/off
    
    var Switch = Java.use("android.widget.Switch");
    var toggle = Switch.$new(activity);
    // Create new Switch instance
    
    toggle.setChecked(false);        // Initial state: unchecked (disabled)
                                            // All cheats start disabled for safety
    
    toggle.setId(0x7F0A0001 + i);   // Set unique view ID
                                            // Android requires unique IDs for views
                                            // 0x7F0A0001 is a safe range for generated IDs
                                            // Increment for each toggle: 0x7F0A0001, 0x7F0A0002, etc.
    
    // =============================================
    // CREATE LABEL FOR CHEAT OPTION
    // =============================================
    // TextView displays the cheat name to the user
    
    var label = TextView.$new(activity);
    // Create new TextView for the label
    
    label.setText(optionName);           // Set text to cheat option name
                                                // e.g., "enable dodge", "autoshoot"
    
    label.setTextColor(0xFFFFFFFF);      // Text color: white
                                                // 0xFFFFFFFF = fully opaque white
                                                // Good contrast against dark background
    
    label.setTextSize(16.0);             // Text size: 16 scaled pixels
                                                // Slightly smaller than title (18sp)
                                                // But still easily readable
    
    label.setTypeface(Typeface.DEFAULT); // Font: default (not bold)
                                                // Differentiates from bold title
    
    // =============================================
    // SET UP TOGGLE EVENT LISTENER
    // =============================================
    // This function is called when user toggles a switch
    // It enables/disables the corresponding cheat feature
    
    var OnCheckedChangeListener = Java.use("android.widget.CompoundButton$OnCheckedChangeListener");
    var listener = OnCheckedChangeListener.$new();
    // Create listener for switch state changes
    
    // Override the onCheckedChanged method
    // This is called automatically when switch is toggled
    listener.onCheckedChanged.implementation = function(buttonView, isChecked) {
        // buttonView: the Switch that was toggled
        // isChecked: new state (true=enabled, false=disabled)
        
        // Handle the cheat toggle
        handleCheatToggle(optionName, isChecked);
        // This function enables/disables the actual cheat functionality
        // It's defined elsewhere in the code
        
        // Update global state
        _0x32b402.cheat_states[optionName] = isChecked;
        
        // Log the toggle action for debugging
        console.log("Cheat '" + optionName + "' " + (isChecked ? "ENABLED" : "DISABLED"));
    };
    
    // Attach the listener to the toggle switch
    toggle.setOnCheckedChangeListener(listener);
    // Now when user taps the switch, our function will be called
    
    // =============================================
    // ASSEMBLE THE OPTION LAYOUT
    // =============================================
    // Add toggle and label to the horizontal layout
    
    // Create layout parameters for toggle (right-aligned)
    var toggleParams = LayoutParams.$new(
        LayoutParams.WRAP_CONTENT,  // Width: wrap to switch size
        LayoutParams.WRAP_CONTENT   // Height: wrap to switch size
    );
    toggleParams.gravity = Gravity.CENTER_VERTICAL; // Vertical alignment: center
    toggleParams.setMargins(0, 0, 16, 0);  // Margins: right=16dp
                                                   // Adds space between switch and label
    
    // Create layout parameters for label (takes remaining space)
    var labelParams = LayoutParams.$new(
        0,                          // Width: 0 (use weight system)
        LayoutParams.WRAP_CONTENT,  // Height: wrap to text height
        1.0f                        // Weight: 1.0 (take all remaining space)
    );
    labelParams.gravity = Gravity.CENTER_VERTICAL; // Vertical alignment: center
    
    // Add views to option layout
    optionLayout.addView(toggle, toggleParams);  // Add switch first (left side)
    optionLayout.addView(label, labelParams);    // Add label second (right side)
    
    // Add completed option layout to main layout
    mainLayout.addView(optionLayout);
    // This option now appears in the cheat menu
    
    // Store references for potential later access
    _0x39f4b2_options[optionName] = {
        layout: optionLayout,
        toggle: toggle,
        label: label
    };
}

// Log completion of menu creation
console.log("Cheat menu created with " + cheatOptions.length + " options");
// Example output: "Cheat menu created with 5 options"
```

# Example original code:
```js
Java.perform(function() {
    console.log("[+] Brawl Stars Cheat Initializing...");
    
    // =============================================
    // GLOBAL STATE AND CONFIGURATION
    // =============================================
    
    const CheatState = {
        // Cheat features
        aimbotEnabled: false,
        autoshootEnabled: false, 
        dodgeEnabled: false,
        xrayEnabled: false,
        spinEnabled: false,
        antiAfkEnabled: false,
        
        // Game addresses (resolved dynamically)
        gameBase: null,
        playerArray: null,
        projectileArray: null,
        aimStickX: null,
        aimStickY: null,
        moveStickX: null,
        moveStickY: null,
        
        // Player data
        localPlayer: {
            index: -1,
            x: 0,
            y: 0,
            z: 0,
            team: 0,
            health: 0,
            maxHealth: 0
        },
        
        // GUI references
        overlayView: null,
        windowManager: null,
        activity: null,
        
        // Timing
        lastAimbotUpdate: 0,
        lastDodgeUpdate: 0,
        lastSpinUpdate: 0
    };

    // =============================================
    // MEMORY SCANNING AND ADDRESS RESOLUTION
    // =============================================
    
    function initializeMemoryScanner() {
        console.log("[+] Scanning for game modules...");
        
        // Get base address of libBrawlStars.so
        Process.enumerateModules().forEach(function(module) {
            if (module.name.includes("libBrawlStars") || module.name.includes("libgame")) {
                CheatState.gameBase = module.base;
                console.log("[+] Found game base: " + CheatState.gameBase);
            }
        });
        
        if (!CheatState.gameBase) {
            console.log("[-] Failed to find game base address");
            return false;
        }
        
        // Resolve critical function addresses
        resolveGameAddresses();
        return true;
    }
    
    function resolveGameAddresses() {
        // These would be found through pattern scanning in real implementation
        CheatState.playerArray = CheatState.gameBase.add(0x19D1E1C);
        CheatState.projectileArray = CheatState.gameBase.add(0x19D6998);
        CheatState.aimStickX = CheatState.gameBase.add(0x19D250C);
        CheatState.aimStickY = CheatState.gameBase.add(0x19D2510);
        CheatState.moveStickX = CheatState.gameBase.add(0x19D2514);
        CheatState.moveStickY = CheatState.gameBase.add(0x19D2518);
        
        console.log("[+] Resolved game addresses:");
        console.log("    Player Array: " + CheatState.playerArray);
        console.log("    Projectile Array: " + CheatState.projectileArray);
    }

    // =============================================
    // ANDROID OVERLAY GUI SYSTEM
    // =============================================
    
    function createOverlayGUI() {
        try {
            // Get current activity
            const ActivityThread = Java.use("android.app.ActivityThread");
            const currentThread = ActivityThread.currentActivityThread();
            const activities = currentThread.mActivities.value;
            
            let activityRecord = null;
            const iterator = activities.values().iterator();
            if (iterator.hasNext()) {
                activityRecord = iterator.next();
            }
            
            CheatState.activity = activityRecord.activity.value;
            
            // Create window parameters
            const WindowManager = Java.use("android.view.WindowManager");
            const LayoutParams = Java.use("android.view.WindowManager$LayoutParams");
            const Gravity = Java.use("android.view.Gravity");
            
            const params = LayoutParams.$new();
            params.width = LayoutParams.MATCH_PARENT;
            params.height = LayoutParams.WRAP_CONTENT;
            params.type = LayoutParams.TYPE_APPLICATION_OVERLAY;
            params.flags = LayoutParams.FLAG_NOT_FOCUSABLE | 
                          LayoutParams.FLAG_NOT_TOUCH_MODAL |
                          LayoutParams.FLAG_WATCH_OUTSIDE_TOUCH;
            params.gravity = Gravity.TOP | Gravity.START;
            params.x = 0;
            params.y = 100;
            
            // Create main layout
            const LinearLayout = Java.use("android.widget.LinearLayout");
            const mainLayout = LinearLayout.$new(CheatState.activity);
            mainLayout.setOrientation(LinearLayout.VERTICAL);
            mainLayout.setBackgroundColor(0x80000000);
            
            // Create title
            const TextView = Java.use("android.widget.TextView");
            const titleView = TextView.$new(CheatState.activity);
            titleView.setText("BSRE Cheat Menu v2.1");
            titleView.setTextColor(0xFF00FF00);
            titleView.setTextSize(18);
            titleView.setGravity(Gravity.CENTER);
            mainLayout.addView(titleView);
            
            // Create cheat toggles
            createCheatToggles(mainLayout);
            
            // Add to window manager
            CheatState.windowManager = CheatState.activity.getWindowManager();
            CheatState.windowManager.addView(mainLayout, params);
            CheatState.overlayView = mainLayout;
            
            console.log("[+] Overlay GUI created successfully");
            
        } catch (error) {
            console.log("[-] Failed to create overlay: " + error);
        }
    }
    
    function createCheatToggles(parentLayout) {
        const cheatOptions = [
            { name: "Aimbot", state: "aimbotEnabled" },
            { name: "Auto Shoot", state: "autoshootEnabled" },
            { name: "Dodge", state: "dodgeEnabled" },
            { name: "X-Ray", state: "xrayEnabled" },
            { name: "Spin Bot", state: "spinEnabled" },
            { name: "Anti-AFK", state: "antiAfkEnabled" }
        ];
        
        const LinearLayout = Java.use("android.widget.LinearLayout");
        const Switch = Java.use("android.widget.Switch");
        const TextView = Java.use("android.widget.TextView");
        const Gravity = Java.use("android.view.Gravity");
        
        cheatOptions.forEach(function(option, index) {
            // Create horizontal container
            const optionLayout = LinearLayout.$new(CheatState.activity);
            optionLayout.setOrientation(LinearLayout.HORIZONTAL);
            optionLayout.setBackgroundColor(0x40000000);
            optionLayout.setPadding(50, 20, 50, 20);
            
            // Create toggle switch
            const toggle = Switch.$new(CheatState.activity);
            toggle.setChecked(false);
            
            // Create label
            const label = TextView.$new(CheatState.activity);
            label.setText(option.name);
            label.setTextColor(0xFFFFFFFF);
            label.setTextSize(16);
            
            // Set up click listener
            toggle.setOnCheckedChangeListener(Java.registerClass({
                name: 'com.example.ToggleListener' + index,
                implements: [Java.use("android.widget.CompoundButton$OnCheckedChangeListener")],
                methods: {
                    onCheckedChanged: function(buttonView, isChecked) {
                        CheatState[option.state] = isChecked;
                        console.log("[+] " + option.name + " " + (isChecked ? "ENABLED" : "DISABLED"));
                        
                        if (option.state === "xrayEnabled") {
                            toggleXRay(isChecked);
                        }
                    }
                }
            }).$new());
            
            // Add views to layout
            optionLayout.addView(toggle);
            optionLayout.addView(label);
            parentLayout.addView(optionLayout);
        });
    }

    // =============================================
    // MEMORY HOOKING SYSTEM
    // =============================================
    
    function installMemoryHooks() {
        console.log("[+] Installing memory hooks...");
        
        // Hook auto-shoot function
        const autoShootAddress = CheatState.gameBase.add(0x0046CB38);
        Interceptor.attach(autoShootAddress, {
            onEnter: function(args) {
                if (CheatState.autoshootEnabled) {
                    // Let our auto-shoot system handle shooting
                    this.skip = true;
                }
            }
        });
        
        // Hook player visibility function
        const hidePlayerAddress = CheatState.gameBase.add(0x004749C4);
        Interceptor.attach(hidePlayerAddress, {
            onEnter: function(args) {
                if (CheatState.xrayEnabled) {
                    // Force return false to always show players
                    this.returnValue = 0;
                }
            }
        });
        
        // Hook team check function
        const teamCheckAddress = CheatState.gameBase.add(0x006752A8);
        Interceptor.attach(teamCheckAddress, {
            onEnter: function(args) {
                // Bypass team checks for targeting
                this.returnValue = 1;
            }
        });
        
        console.log("[+] Memory hooks installed");
    }
    
    function applyMemoryPatches() {
        // Patch ammo check to always return true (infinite ammo)
        const ammoCheckAddress = CheatState.gameBase.add(0x037C048);
        Memory.patchCode(ammoCheckAddress, 4, function(code) {
            const writer = new X86Writer(code, { pc: ammoCheckAddress });
            writer.putMovRegU32('eax', 0x1);  // Always return true
            writer.putRet();
            writer.flush();
        });
        
        console.log("[+] Memory patches applied");
    }

    // =============================================
    // AIMBOT SYSTEM
    // =============================================
    
    function updateAimbot() {
        if (!CheatState.aimbotEnabled && !CheatState.autoshootEnabled) return;
        
        const currentTime = Date.now();
        if (currentTime - CheatState.lastAimbotUpdate < 16) return; // ~60 FPS
        CheatState.lastAimbotUpdate = currentTime;
        
        // Update player data
        updatePlayerData();
        
        // Find best target
        const target = findBestTarget();
        if (!target) return;
        
        // Calculate aim angles
        const aimData = calculateAimAngles(CheatState.localPlayer, target);
        
        // Write aim coordinates to game memory
        if (CheatState.aimbotEnabled) {
            Memory.writeFloat(CheatState.aimStickX, aimData.aimX);
            Memory.writeFloat(CheatState.aimStickY, aimData.aimY);
        }
        
        // Auto shoot if enabled
        if (CheatState.autoshootEnabled && aimData.inRange) {
            triggerAutoShoot();
        }
    }
    
    function updatePlayerData() {
        try {
            // Get local player index
            const playerIndex = getOwnPlayerIndex();
            if (playerIndex === -1) return;
            
            CheatState.localPlayer.index = playerIndex;
            
            // Calculate player object pointer
            const playerPtr = CheatState.playerArray.add(playerIndex * 0x268);
            
            // Read player data
            CheatState.localPlayer.x = Memory.readFloat(playerPtr.add(0x40));
            CheatState.localPlayer.y = Memory.readFloat(playerPtr.add(0x44));
            CheatState.localPlayer.z = Memory.readFloat(playerPtr.add(0x48));
            CheatState.localPlayer.team = Memory.readU8(playerPtr.add(0x34));
            CheatState.localPlayer.health = Memory.readS32(playerPtr.add(0x50));
            CheatState.localPlayer.maxHealth = Memory.readS32(playerPtr.add(0x54));
            
        } catch (error) {
            console.log("[-] Error updating player data: " + error);
        }
    }
    
    function findBestTarget() {
        let bestTarget = null;
        let bestScore = 0;
        
        for (let i = 0; i < 10; i++) {
            if (i === CheatState.localPlayer.index) continue;
            
            const enemyPtr = CheatState.playerArray.add(i * 0x268);
            const enemyTeam = Memory.readU8(enemyPtr.add(0x34));
            
            // Skip friendly players and invalid entries
            if (enemyTeam === CheatState.localPlayer.team || enemyTeam === 0xFF) continue;
            
            const enemyHealth = Memory.readS32(enemyPtr.add(0x50));
            if (enemyHealth <= 0) continue; // Skip dead players
            
            const enemyX = Memory.readFloat(enemyPtr.add(0x40));
            const enemyY = Memory.readFloat(enemyPtr.add(0x44));
            
            // Calculate distance
            const dx = enemyX - CheatState.localPlayer.x;
            const dy = enemyY - CheatState.localPlayer.y;
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
    
    function calculateAimAngles(localPlayer, target) {
        // Calculate basic aim vector
        let aimX = target.x - localPlayer.x;
        let aimY = target.y - localPlayer.y;
        
        // Normalize vector
        const length = Math.sqrt(aimX * aimX + aimY * aimY);
        aimX /= length;
        aimY /= length;
        
        // Add human-like imperfection
        if (CheatState.aimbotEnabled) {
            const error = (Math.random() - 0.5) * 0.1; // ±5% error
            aimX += error;
            aimY += error;
            
            // Re-normalize after adding error
            const newLength = Math.sqrt(aimX * aimX + aimY * aimY);
            aimX /= newLength;
            aimY /= newLength;
        }
        
        const inRange = target.distance < 10.0; // Within 10 units
        
        return {
            aimX: aimX,
            aimY: aimY,
            inRange: inRange
        };
    }
    
    function triggerAutoShoot() {
        // This would call the game's shoot function
        // In practice, this would simulate button press or call internal function
        try {
            const shootFunction = CheatState.gameBase.add(0x00477D1C);
            Interceptor.attach(shootFunction, {
                onEnter: function(args) {
                    // Force shoot
                }
            });
        } catch (error) {
            // Shoot simulation failed
        }
    }

    // =============================================
    // DODGE SYSTEM
    // =============================================
    
    function updateDodgeSystem() {
        if (!CheatState.dodgeEnabled) return;
        
        const currentTime = Date.now();
        if (currentTime - CheatState.lastDodgeUpdate < 8) return; // 120 FPS for faster reaction
        CheatState.lastDodgeUpdate = currentTime;
        
        // Scan for incoming projectiles
        const dangerousProjectile = findDangerousProjectile();
        if (!dangerousProjectile) return;
        
        // Calculate dodge vector
        const dodgeVector = calculateDodgeVector(dangerousProjectile);
        
        // Apply dodge movement
        Memory.writeFloat(CheatState.moveStickX, dodgeVector.x);
        Memory.writeFloat(CheatState.moveStickY, dodgeVector.y);
    }
    
    function findDangerousProjectile() {
        for (let i = 0; i < 50; i++) {
            const projPtr = CheatState.projectileArray.add(i * 0x60);
            const isActive = Memory.readU8(projPtr);
            
            if (isActive === 1) {
                const projX = Memory.readFloat(projPtr.add(0x10));
                const projY = Memory.readFloat(projPtr.add(0x14));
                const projVX = Memory.readFloat(projPtr.add(0x20));
                const projVY = Memory.readFloat(projPtr.add(0x24));
                const radius = Memory.readFloat(projPtr.add(0x30));
                const owner = Memory.readU8(projPtr.add(0x38));
                
                // Skip own projectiles
                if (owner === CheatState.localPlayer.index) continue;
                
                // Calculate collision time
                const collisionTime = calculateCollisionTime(
                    CheatState.localPlayer.x, CheatState.localPlayer.y,
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
    
    function calculateCollisionTime(playerX, playerY, projX, projY, projVX, projVY, radius) {
        // Simple collision prediction
        const dx = projX - playerX;
        const dy = projY - playerY;
        const distance = Math.sqrt(dx * dx + dy * dy);
        
        const relativeSpeed = Math.sqrt(projVX * projVX + projVY * projVY);
        const timeToCollision = distance / relativeSpeed;
        
        return timeToCollision;
    }
    
    function calculateDodgeVector(projectile) {
        // Calculate direction away from projectile
        const dx = CheatState.localPlayer.x - projectile.x;
        const dy = CheatState.localPlayer.y - projectile.y;
        
        // Normalize to get direction
        const length = Math.sqrt(dx * dx + dy * dy);
        const dodgeX = dx / length;
        const dodgeY = dy / length;
        
        // Add some randomness to avoid pattern detection
        const randomX = (Math.random() - 0.5) * 0.2;
        const randomY = (Math.random() - 0.5) * 0.2;
        
        return {
            x: dodgeX + randomX,
            y: dodgeY + randomY
        };
    }

    // =============================================
    // X-RAY SYSTEM
    // =============================================
    
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
            if (i === CheatState.localPlayer.index) continue;
            
            const enemyPtr = CheatState.playerArray.add(i * 0x268);
            const enemyTeam = Memory.readU8(enemyPtr.add(0x34));
            
            if (enemyTeam !== CheatState.localPlayer.team && enemyTeam !== 0xFF) {
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

    // =============================================
    // SPIN BOT SYSTEM
    // =============================================
    
    function updateSpinBot() {
        if (!CheatState.spinEnabled) return;
        
        const currentTime = Date.now();
        if (currentTime - CheatState.lastSpinUpdate < 50) return; // 20 FPS for spinning
        CheatState.lastSpinUpdate = currentTime;
        
        // Calculate spin angle (360 degrees over 2 seconds)
        const spinSpeed = 360 / 2000; // degrees per millisecond
        const elapsed = currentTime % 2000; // Loop every 2 seconds
        const angle = (elapsed * spinSpeed) * (Math.PI / 180); // Convert to radians
        
        // Calculate movement vector based on angle
        const moveX = Math.cos(angle) * 0.7;
        const moveY = Math.sin(angle) * 0.7;
        
        // Apply spin movement
        Memory.writeFloat(CheatState.moveStickX, moveX);
        Memory.writeFloat(CheatState.moveStickY, moveY);
    }

    // =============================================
    // ANTI-AFK SYSTEM
    // =============================================
    
    function updateAntiAFK() {
        if (!CheatState.antiAfkEnabled) return;
        
        // Every 30 seconds, simulate a small movement to prevent AFK detection
        const currentTime = Date.now();
        if (currentTime % 30000 < 16) { // Every ~30 seconds
            const randomX = (Math.random() - 0.5) * 0.1;
            const randomY = (Math.random() - 0.5) * 0.1;
            
            Memory.writeFloat(CheatState.moveStickX, randomX);
            Memory.writeFloat(CheatState.moveStickY, randomY);
        }
    }

    // =============================================
    // GAME FUNCTION WRAPPERS
    // =============================================
    
    function getOwnPlayerIndex() {
        try {
            // This would call the actual game function
            // For now, return a simulated value
            return 0;
        } catch (error) {
            return -1;
        }
    }

    // =============================================
    // MAIN LOOP AND INITIALIZATION
    // =============================================
    
    function mainLoop() {
        try {
            updateAimbot();
            updateDodgeSystem();
            updateSpinBot();
            updateAntiAFK();
        } catch (error) {
            console.log("[-] Error in main loop: " + error);
        }
        
        // Schedule next frame
        setTimeout(mainLoop, 1);
    }
    
    function initializeCheat() {
        console.log("[+] Starting Brawl Stars cheat...");
        
        // Step 1: Initialize memory scanner
        if (!initializeMemoryScanner()) {
            console.log("[-] Failed to initialize memory scanner");
            return;
        }
        
        // Step 2: Create overlay GUI
        createOverlayGUI();
        
        // Step 3: Install memory hooks
        installMemoryHooks();
        
        // Step 4: Apply memory patches
        applyMemoryPatches();
        
        // Step 5: Start main loop
        console.log("[+] Cheat initialized successfully - starting main loop");
        mainLoop();
    }
    
    // Start the cheat
    initializeCheat();
});
```
