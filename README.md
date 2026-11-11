# The ASUS gaming laptop ACPI firmware DPC stall

> **Status update (December 2025):** The original periodic ACPI/DPC stall described in this write-up is **fixed**. ASUS shipped a UEFI update that rewrites the ACPI `ECLV` routine (the sleeps and the self-rearm are gone) and also includes other ACPI power-management optimizations.

## What changed

ASUS updated the firmware's ACPI EC/GPE event handling. The main change is in the `ECLV` method. It no longer sleeps inside the event-processing loop, and it no longer re-arms itself when it hits its internal "time budget".

### AML diff (old vs new)

```diff
 Method (ECLV, 0, NotSerialized)
 {
-    // Main loop - continues while events exist OR sleep events are pending
-    // AND we haven't exceeded our time budget (TI3S < 0x78)
-    While (((CKEV() != Zero) || (SLEC != Zero)) && (TI3S < 0x78))
+    While ((CKEV () != Zero))
     {
         Local1 = One
-        While (Local1 != Zero)
+        While ((Local1 != Zero))
         {
-            Local1 = GEVT()    // Get next event from queue
-            LEVN (Local1)      // Process the event
-            TIMC += 0x19       // Increment time counter by 25
-            
-            If ((SLEC != Zero) && (Local1 == Zero))
-            {
-                If (TIMC == 0x19)
-                {
-                    Sleep (0x64)
-                    TIMC = 0x64
-                    TI3S += 0x04
-                }
-                Else
-                {
-                    Sleep (0x19)
-                    TI3S++
-                }
-            }
+            Local1 = GEVT ()
+            LEVN (Local1)
         }
     }
-    
-    If (TI3S >= 0x78)
-    {
-        TI3S = Zero
-        If (EEV0 == Zero)
-        {
-            EEV0 = 0xFF
-        }
-    }
 }
```

### Updated LatencyMon results (G533ZW, UEFI 327)

<img width="1738" height="636" alt="image" src="https://github.com/user-attachments/assets/767bde14-84c7-41ea-a33f-75a71076cfb9" />


## The problem

You own a high-end ASUS ROG laptop, maybe a Strix, Scar or Zephyrus. The specs are good: an RTX 30/40 series GPU, a top Intel CPU and plenty of RAM. But it stutters on basic tasks like watching a YouTube video, audio crackles and pops on Discord calls, and the mouse cursor freezes for a split second.

You've probably tried the usual fixes:
- Updating every driver, multiple times.
- Doing a clean reinstall of Windows.
- Disabling every power-saving option.
- Manually tweaking processor interrupt affinities.
- Following long multi-step guides from Reddit threads.
- Even installing Linux, where the problem is still there.

If none of that worked, it's because the problem was never in the operating system or a driver. It was in the laptop's firmware (UEFI), and ASUS has since shipped a UEFI update that fixes the original periodic ACPI/DPC stall described below.

## Initial symptoms and measurement

The first thing to run for symptoms like these is LatencyMon. On an affected ASUS Zephyrus M16 it reports:

```
CONCLUSION
Your system appears to be having trouble handling real-time audio and other tasks. 
You are likely to experience buffer underruns appearing as drop outs, clicks or pops.

HIGHEST MEASURED INTERRUPT TO PROCESS LATENCY
Highest measured interrupt to process latency (μs):   65,816.60
Average measured interrupt to process latency (μs):   23.29

HIGHEST REPORTED ISR ROUTINE EXECUTION TIME
Highest ISR routine execution time (μs):              536.80
Driver with highest ISR routine execution time:       ACPI.sys

HIGHEST REPORTED DPC ROUTINE EXECUTION TIME  
Highest DPC routine execution time (μs):              5,998.83
Driver with highest DPC routine execution time:       ACPI.sys
```

`ACPI.sys` has both the highest ISR and the highest DPC execution time. The per-CPU numbers narrow it down further:

```
CPU 0 Interrupt cycle time (s):                       208.470124
CPU 0 ISR highest execution time (μs):                536.804674
CPU 0 DPC highest execution time (μs):                5,998.834725
CPU 0 DPC total execution time (s):                   90.558238
```

CPU 0 takes almost all of it, with over 90 seconds of DPC time, while the other cores are barely affected. The work is confined to a single core.

The same test on a Scar 15 from 2022 shows the same result, high DPC latency coming from `ACPI.sys`.

<img width="974" height="511" alt="latencymon" src="https://github.com/user-attachments/assets/fdf6f26a-dda8-4561-82c7-349fc8c298ab" />

It's easy to blame a Windows driver, but `ACPI.sys` is not a typical driver. Its main job is to interpret ACPI Machine Language (AML), the code provided by the laptop's firmware (UEFI). If `ACPI.sys` is slow, it's because the firmware is giving it slow or broken AML to run. These slowdowns are often triggered by General Purpose Events (GPEs) and traffic from the Embedded Controller (EC).

## ETW tracing

### Setting up the ACPI trace

To see what `ACPI.sys` is doing during these latency spikes, we can use Event Tracing for Windows (ETW) to capture logs from the ACPI providers.

```powershell
# Find the relevant ACPI ETW providers
logman query providers | findstr /i acpi
# This returns two providers:
# Microsoft-Windows-Kernel-Acpi {C514638F-7723-485B-BCFC-96565D735D4A}
# Microsoft-ACPI-Provider {DAB01D4D-2D48-477D-B1C3-DAAD0CE6F06B}

# Start the trace session
logman start ACPITrace -p {DAB01D4D-2D48-477D-B1C3-DAAD0CE6F06B} 0xFFFFFFFF 5 -o C:\Temp\acpi.etl -ets
logman update ACPITrace -p {C514638F-7723-485B-BCFC-96565D735D4A} 0xFFFFFFFF 5 -ets

# Then once we're done we can stop the trace and check the etl file and save the data in csv format aswell.
logman stop ACPITrace -ets
tracerpt C:\Temp\acpi.etl -o C:\Temp\acpi_events.csv -of CSV
```

### The spikes are periodic

Opening the trace file in Windows Performance Analyzer shows that the spikes are periodic. They come every 30 to 60 seconds.

<img width="1673" height="516" alt="61c7abb1-d7aa-4b69-9a88-22cca7352f00" src="https://github.com/user-attachments/assets/2aac7320-3e06-4025-841c-86129f9d5b62" />

Random spikes would suggest a hardware fault or thermal throttling. A regular interval like this points to a timer or a scheduled event in the system's own logic.

The raw event data shows the same thing:
```csv
Clock-Time (100ns),        Event,                      Kernel(ms), CPU
134024027290917802,       _GPE._L02 started,          13.613820,  0
134024027290927629,       _SB...BAT0._STA started,    0.000000,   4
134024027290932512,       _GPE._L02 finished,         -,          6
```

The first event, `_GPE._L02`, is an interrupt handler that takes **13.6 milliseconds** to execute. That is far too long for a high-priority interrupt, and it is enough to break real-time work like audio.

Further into the trace there is something else odd. The system keeps trying to power the discrete GPU on and off, even when it's supposed to stay on.
```csv
Clock-Time,                Event,                    Duration
134024027315051227,       _SB.PC00.GFX0._PS0 start, 278μs     # GPU Power On
134024027315155404,       _SB.PC00.GFX0._DOS start, 894μs     # Display Output Switch
134024027330733719,       _SB.PC00.GFX0._PS3 start, 1364μs    # GPU Power Off
[~15 seconds later]
134024027607550064,       _SB.PC00.GFX0._PS0 start, 439μs     # Power On Again!
134024027607657368,       _SB.PC00.GFX0._DOS start, 1079μs    # Display Output Switch
134024027623134006,       _SB.PC00.GFX0._PS3 start, 394μs     # Power Off Again!
...
```
### Note on GPU power-method activity

The trace also shows periodic GPU-related ACPI method activity (for example `_PS0`, `_PS3`, and `_DOS`) that lines up with `_GPE._L02`. Whether these GPU transitions are intended or correct depends on the model, configuration and UEFI version, so this write-up no longer treats them as a settled root cause. The rest of it sticks to the measured ACPI stall and its firmware fix.

### General Purpose Events

GPEs are how the firmware signals hardware events to the operating system. They are essentially hardware interrupts that trigger the execution of ACPI code. In the trace, `_GPE._L02` is the source of the latency.

The timing of these events is consistent:
```
_GPE._L02 Event Analysis from ROG Strix Trace:

Event 1 @ Clock 134024027290917802
  Duration: 13,613,820 ns (13.61ms)
  Triggered: Battery and AC adapter status checks

Event 2 @ Clock 134024027654496591  
  Duration: 13,647,255 ns (13.65ms)
  Triggered: Battery and AC adapter status checks
  
Event 3 @ Clock 134024028048493318
  Duration: 13,684,515 ns (13.68ms)  
  Triggered: Battery and AC adapter status checks

Interval between events: ~36-39 seconds
Consistency: The duration is stable and the interval is periodic.
```

### What each `_GPE._L02` triggers

Every time the long `_GPE._L02` event fires, it triggers the same sequence of ACPI method calls.

<img width="589" height="589" alt="64921999-7614-4706-a5ac-54c39c38fd0b" src="https://github.com/user-attachments/assets/01326c61-b7a2-4c12-a907-8433f43a6a72" />

The sequence:
1.  A hardware interrupt fires `_GPE._L02`.
2.  The handler runs methods to check battery status.
3.  Shortly after, the firmware attempts to change the GPU's power state.
4.  The system runs normally for about 30-60 seconds.
5.  The cycle repeats.

## Extracting and decompiling the firmware code

### Dumping the ACPI tables

To look at the code behind this behavior, we need to extract and decompile the ACPI tables that the UEFI provides to the operating system.

```bash
# Extract all ACPI tables into binary .dat files
acpidump -b

# Output includes:
# DSDT.dat - The main Differentiated System Description Table
# SSDT1.dat ... SSDT17.dat - Secondary System Description Tables

# Decompile the main table into human-readable ACPI Source Language (.dsl)
iasl -d dsdt.dat
```
The decompiled ASL is the code that the firmware gives to ACPI.sys, which runs it inside the Windows kernel. A logic flaw in this code shows up directly in how the system behaves.


### Finding the GPE handler

Searching the decompiled `DSDT.dsl` file, we find the definition of the GPE handler:

```asl
Scope (_GPE)
{
    Method (_L02, 0, NotSerialized)  // _Lxx: Level-Triggered GPE
    {
        \_SB.PC00.LPCB.ECLV ()
    }
}
```
When the `_L02` interrupt occurs, it calls a single method, `ECLV`. The "L" prefix in `_L02` means this is a **level-triggered** interrupt, so it keeps firing as long as the underlying hardware condition is active.

### The pre-fix `ECLV` implementation (historical)

In older UEFI versions, `ECLV()` looked like this. It is enough to explain the periodic stutters and latency spikes seen in ETW and LatencyMon. Newer UEFI builds replace it (see the diff at the top of this document).

```asl
Method (ECLV, 0, NotSerialized)  // Starting at line 099244
{
    // Main loop - continues while events exist OR sleep events are pending
    // AND we haven't exceeded our time budget (TI3S < 0x78)
    While (((CKEV() != Zero) || (SLEC != Zero)) && (TI3S < 0x78))
    {
        Local1 = One
        While (Local1 != Zero)
        {
            Local1 = GEVT()    // Get next event from queue
            LEVN (Local1)      // Process the event
            TIMC += 0x19       // Increment time counter by 25
            
            If ((SLEC != Zero) && (Local1 == Zero))
            {
                // No events but sleep events pending
                If (TIMC == 0x19)
                {
                    Sleep (0x64)    // Sleep for 100 milliseconds!!!
                    TIMC = 0x64     // Set time counter to 100
                    TI3S += 0x04    // Increment major counter by 4
                }
                Else
                {
                    Sleep (0x19)    // Sleep for 25 milliseconds!!!
                    TI3S++          // Increment major counter by 1
                }
            }
        }
    }
    
    If (TI3S >= 0x78)  // If we hit our time budget (120)
    {
        TI3S = Zero
        If (EEV0 == Zero)
        {
            EEV0 = 0xFF    // Force another event to be pending!
        }
    }
}
```

### Breaking down the pre-fix behavior

There are three problems in this short block of code.

**Issue 1: Sleeping in an interrupt-driven path**
```asl
Sleep (0x64)    // 100ms sleep
Sleep (0x19)    // 25ms sleep
```
GPE control methods need to finish quickly, but `ECLV` sleeps for 25 or 100 milliseconds at a time inside its event loop.

> Clarification: These Sleep() calls live in the ACPI GPE handling path for the GPE L02, these calls get executed at PASSIVE_LEVEL after the SCI/GPE is acknowledged so it's not a raw ISR (because i don't think windows will even allow that) but analyzing this further while the control method runs the GPE stays masked  and the ACPI/EC work is serialized. With the Sleep() calls inside that path and the self rearm it seems to have the effect of making ACPI.sys get tied up in long periodic bursts (often on CPU 0) which still have the same effect on the system.

**Issue 2: Time-sliced event processing**
The whole loop is designed to run for a long time, processing events in batches. It's effectively a poorly designed task scheduler running inside an interrupt handler, and it can tie up a CPU core for potentially seconds at a time.

**Issue 3: Self-rearming behavior**
```asl
If (EEV0 == Zero)
{
    EEV0 = 0xFF    // Forces all EC event bits on
}
```
Even if the Embedded Controller's event queue is empty, this code creates a new, artificial event. That makes another interrupt fire shortly after, which gives the periodic pattern of ACPI spikes seen in the traces.

## The event dispatch system

### How events route to actions

The LEVN() method takes an event and routes it:

```asl
Method (LEVN, 1, NotSerialized)
  {
      If ((Arg0 != Zero))
      {
          MBF0 = Arg0
          P80B = Arg0
          Local6 = Match (LEGA, MEQ, Arg0, MTR, Zero, Zero)
          If ((Local6 != Ones))
          {
              LGPA (Local6)
          }
      }
  }

```

### The `LGPA` dispatch table

The LGPA() method is a giant switch statement handling different events:

```asl
Method (LGPA, 1, Serialized)  // Line 098862
{
    Switch (ToInteger (Arg0))
    {
        Case (Zero)  // Most common case - power event
        {
            DGD2 ()       // GPU-related function
            ^EC0._QA0 ()  // EC query method
            PWCG ()       // Power change - this is our battery polling
        }
        
        Case (0x18)  // GPU-specific event
        {
            If (M6EF == One)
            {
                Local0 = 0xD2
            }
            Else
            {
                Local0 = 0xD1
            }
            NOD2 (Local0)  // Notify GPU driver
        }
        
        Case (0x1E)  // Another GPU event
        {
            Notify (^^PEG1.PEGP, 0xD5)  // Direct GPU notification
            ROCT = 0x55                  // Sets flag for follow-up
        }
       
    }
}
```
So when a GPE fires, the dispatch logic calls functions related to battery polling and GPU notifications.

## The battery polling function

The `PWCG()` method is called by multiple event types. It polls the battery and AC adapter status.

```asl
Method (PWCG, 0, NotSerialized)
{
    Notify (ADP0, Zero)      // Tell OS to check the AC adapter
    ^BAT0._BST ()            // Execute the Battery Status method
    Notify (BAT0, 0x80)      // Tell OS the battery status has changed
    ^BAT0._BIF ()            // Execute the Battery Information method  
    Notify (BAT0, 0x81)      // Tell OS the battery info has changed
}
```
Which we can see here:

<img width="1043" height="315" alt="image" src="https://github.com/user-attachments/assets/f6c62050-b470-49bd-ad55-35def0fff893" />

Each of these operations has to talk to the Embedded Controller, which adds more work inside the already-stalled interrupt handler.

### The GPU notification system

The `NOD2()` method sends notifications to the GPU driver.

```asl
Method (NOD2, 1, Serialized)
{
    If ((Arg0 != DNOT))
    {
        DNOT = Arg0
        Notify (^^PEG1.PEGP, Arg0)
    }

    If ((ROCT == 0x55))
    {
        ROCT = Zero
        Notify (^^PEG1.PEGP, 0xD1) // Hardware-Specific
    }
}
```
These notifications (`0xD1`, `0xD2`, etc.) are hardware-specific signals that tell the NVIDIA driver to re-evaluate its power state. In traces this can show up as GPU-related ACPI method activity (for example `_PSx` / `_DOS`) near the same periodic events.

## Cross-system analysis

Traces from more than one ASUS gaming laptop model show the same behavior.

#### Scar 15
- **Trace Duration:** 4.1 minutes
- **`_GPE._L02` Events:** 7 (every ~39 seconds)
- **Avg. GPE Duration:** 1.56ms 
- **GPU Power Cycles:** 8

#### Zephyrus M16
- **Trace Duration:** 19.9 minutes
- **`_GPE._L02` Events:** 3 (same periodic pattern)
- **Avg. GPE Duration:** 2.94ms
- **GPU Power Cycles:** 197 (far more frequent)
- **ASUS WMI Calls:** 2,370 (Armoury Crate amplifying the problem)

### What actually breaks

The firmware acts as the hardware abstraction layer between Windows and the physical hardware. When ACPI control methods execute, they run under the Windows ACPI driver with specific timing constraints and because of these timing constraints GPE control methods need to finish quickly because the firing GPE stays masked until the method returns so sleeping or polling inside a path like that can trigger real time-glitches and produce very high latency numbers, as our tests indicate.

Microsoft's [Hardware Lab Kit GlitchFree test](https://learn.microsoft.com/windows-hardware/test/hlk/testref/f0ed5aa8-ef49-4fc9-99b6-753c857e4e2d) checks for this by measuring audio/video glitches during HD playback. It fails systems with driver stalls longer than a few milliseconds, because delays like that break the real-time guarantees that smooth media playback needs.

These ASUS systems break those constraints. The firmware holds GPE._L02 masked for 13ms while sleeping in ECLV, and all ACPI/EC operations are serialized behind that delay. It polls battery state when it should use event-driven notifications. The result is severe periodic latency spikes on high-end hardware.

### What the affected systems have in common

These are different models, but all affected systems show the same flaws:
1.  `_GPE._L02` handlers take milliseconds to execute instead of microseconds.
2.  The GPEs trigger unnecessary battery polling.
3.  Some platforms show GPU-related method activity that lines up with the same periodic events.
4.  The whole thing is driven by a periodic, timer-like trigger.

## Summary of findings

### Root cause 1: Misunderstanding of interrupt context

On windows, the LXX / EXX run at PASSIVE_LEVEL via ACPI.sys but while a GPE control method runs **the firing GPE stays masked** and ACPI/EC work is **serialized**. ASUS's dispatch from GPE._L02 to ECLV loops, calls Sleep(25/100ms) and re-arms the EC stretching that masked window into tens of milliseconds (which would explain the 13ms CPU time in ETW (Kernel ms) delay for GPE Events) and producing a periodic ACPI.sys burst that causes the latency problems on the system. The correct behavior is to latch or clear the event, exit the method, and signal a driver with Notify for any heavy work; do not self-rearm or sleep in this path at all.

### Root cause 2: Flawed interrupt handling
The firmware re-arms the interrupt artificially. Instead of clearing the source and waiting for the next real hardware event, it creates an endless loop of GPEs, which turns a hardware notification system into a periodic timer.

### Root cause 3: Lack of platform awareness
Some of the GPU-related ACPI activity seen alongside `_GPE._L02` is still under investigation, and this write-up does not treat it as a settled root cause.

## Timeline of user reports

Users have been reporting the same symptoms (high ACPI.sys DPC latency, periodic stuttering, audio crackling) since at least 2021, across ASUS's entire gaming laptop lineup.

**August 2021: First major reports (AMD Advantage Edition)**
The earliest documented case on the official ASUS ROG forums is a **G15 Advantage Edition (G513QY, all-AMD)** owner reporting ["severe DPC latency from ACPI.sys"](https://rog-forum.asus.com/t5/rog-strix-series/g15-advantage-edition-g513qy-severe-dpc-latency-audio-dropouts/m-p/809512) with audio dropouts under any load. The thread, last edited in March 2024, shows the issue remained unresolved for years.

**August 2021: Parallel reports (NVIDIA-based models)**
Around the same time, **separate Reddit threads on NVIDIA-based ROG models** describe [identical ACPI.sys latency problems](https://www.reddit.com/r/ASUS/comments/odprtv/high_dpc_latency_from_acpisys_can_be_caused_by/). The GPU vendor is different but the firmware/ACPI failure pattern is the same.

**2021-2023: Spreading across models**  
Over this period the same reports show up across ASUS's gaming lineup:
- [ROG Strix models experience micro-stutters](https://www.reddit.com/r/techsupport/comments/mxtm86/i_need_help_high_acpisys_latency_and_microstutters/)
- [TUF Gaming series reports throttling for seconds at a time](https://www.reddit.com/r/Asustuf/comments/1m2e40v/my_laptop_throttling_for_few_seconds/)
- [G18 models show the same 45-second periodic stuttering](https://www.reddit.com/r/techsupport/comments/17rqfq5/new_laptop_started_stuttering_every_45_seconds/)

**2023-2024: Still there in new models**  
The newer generations have it too:
- [2023 Zephyrus M16 owners report persistent audio issues](https://www.reddit.com/r/ZephyrusM16/comments/1j33ld6/this_machine_has_been_nothing_but_problems_no/)
- [2023 G16 models still have audio pops/crackles](https://www.reddit.com/r/ZephyrusG14/comments/1l4jb13/audio_popscrackles_on_zephyrus_g16_2023/)
- [2024 Intel G16 models need workarounds for audio stuttering](https://www.reddit.com/r/ZephyrusG14/comments/1i2w9ah/resolving_audio_popsstuttering_on_2024_intel_g16/)

## Conclusion

### Status: fixed (UEFI update)

The periodic ACPI/DPC stall in this write-up was caused by what the firmware did inside `ECLV` (sleeping/polling and self-rearming in the EC/GPE event path). ASUS has since shipped a UEFI update that rewrites `ECLV` to process queued events without sleeping and without the artificial self-rearm. It also optimizes other ACPI power-management methods.

If you were here for the original "ACPI.sys latency spikes every ~30-60 seconds" symptom: update to the latest UEFI for your model. LatencyMon results from after the update are in the section at the top.

### Ongoing investigation

Other platform stability and power-management issues, separate from the original DPC stall, are still being investigated (including PCIe power-management configuration problems such as LTR L1.2 threshold mismatches). That work is outside the scope of this write-up.

> Update 1: ASUS has officially put out a statement: https://x.com/asus_rogna/status/1968404596658983013?s=46

> Update 2: Reply from ASUS RD received; repro info sent over

> Update 3: Asus sent me beta UEFI, currently testing. (https://x.com/ASUS_ROGNA/status/1971662996339646645/photo/1)

> Update 4: Updated UEFI shipped; `ECLV` rewritten and the periodic ACPI/DPC stall is resolved (see the diff and new LatencyMon screenshots above).
