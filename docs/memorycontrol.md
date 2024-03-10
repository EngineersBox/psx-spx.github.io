#   Memory Control
The Memory Control registers are initialized by the BIOS, and, normally
software doesn't need to change that settings. Some registers are useful for
expansion hardware (allowing to increase the memory size and bus width).<br/>

#### 1F801000h - Expansion 1 Base Address (usually 1F000000h)
#### 1F801004h - Expansion 2 Base Address (usually 1F802000h)
```
  0-23   Base Address   (Read/Write)
  24-31  Fixed          (Read only, always 1Fh)
```
For Expansion 1, the address is forcefully aligned to the selected expansion
size (see below), ie. if the size is bigger than 1 byte, then the lower bit(s)
of the base address are ignored.<br/>
For Expansion 2, trying to use ANY other value than 1F802000h seems to disable
the Expansion 2 region, rather than mapping it to the specified address (ie.
Port 1F801004h doesn't seem to work).<br/>
For Expansion 3, the address seems to be fixed (1FA00000h).<br/>

#### 1F801008h - Expansion 1 Delay/Size (usually 0013243Fh) (512Kbytes, 8bit bus)
#### 1F80100Ch - Expansion 3 Delay/Size (usually 00003022h) (1 byte)
#### 1F801010h - BIOS ROM Delay/Size (usually 0013243Fh) (512Kbytes, 8bit bus)
#### 1F801014h - SPU Delay/Size (200931E1h) (use 220931E1h for SPU-RAM reads)
#### 1F801018h - CDROM Delay/Size (00020843h or 00020943h)
#### 1F80101Ch - Expansion 2 Delay/Size (usually 00070777h) (128 bytes, 8bit bus)
```
  0-3   Write Delay        (00h..0Fh=01h..10h Cycles)
  4-7   Read Delay         (00h..0Fh=01h..10h Cycles)
  8     Recovery Period    (0=No, 1=Yes, uses COM0 timings)
  9     Hold Period        (0=No, 1=Yes, uses COM1 timings)
  10    Floating Period    (0=No, 1=Yes, uses COM2 timings)
  11    Pre-strobe Period  (0=No, 1=Yes, uses COM3 timings)
  12    Data Bus-width     (0=8bits, 1=16bits)
  13    Auto Increment     (0=No, 1=Yes)
  14-15 Unknown (R/W)
  16-20 Memory Window Size (1 SHL N bytes) (0..1Fh = 1 byte ... 2 gigabytes)
  21-23 Unknown (always zero)
  24-27 DMA timing override
  28    Address error flag. Write 1 to it to clear it.
  29    DMA timing select  (0=use normal timings, 1=use bits 24-27)
  30    Wide DMA           (0=use bit 12, 1=override to full 32 bits)
  31    Wait               (1=wait on external device before being ready)
```
When booting, all these registers are using the maximum cycle delays for both
reads and writes. Then, the BIOS will immediately select a faster read
access delay, resulting in a visible speed up after the first few instructions.
The effects aren't immediate however. The BIOS boots using the following instructions:

```mips
bfc00000    lui        $t0, 0x0013
bfc00004    ori        $t0, 0x243f
bfc00008    lui        $at, 0x1f80
bfc0000c    sw         $t0, 0x1010($at)
bfc00010    nop
bfc00014    li         $t0, 0x0b88
bfc00018    lui        $at, 0x1f80
bfc0001c    sw         $t0, 0x1060($at)
bfc00020    nop
```

When using a logic analyzer to monitor the boot sequence, the instruction at
bfc00014 is still read using the old timings since reset, and then the instruction
at bfc00018 is finally read using the sped up timings.

Reads and writes access times aren't symmetrical, and are each controlled with
their own values. By default, EXP1 will be set to 16 cycles when writing, which
is the slowest possible. If the programmer wants to write to a flash chip on
EXP1, or communicate with a computer, speeding up write access is recommended.

The fastest a port could go would be by setting the lowest 16 bits to zero, which
will result in 3 CPU cycles for a single byte access.

!CS always goes active at least one cycle before !WR or !RD go active. The various
timing changes are between all the events inside the data read/write waveform. The
whole formula for computing the total access time is fairly complex overall, and
difficult to properly describe.

 - The pre-strobe period will add delays between the moment the data bus is set,
and the moment !CS goes active.
 - The hold period will keep the data in the data bus for some more cycles after
!WR goes inactive, and before !CS goes inactive. The accessed device is supposed
to sample the data bus during this interval.
 - The floating period will keep the data bus floating for some more cycles after
!RD goes inactive, and before !CS goes inactive. The accessed device is supposed
to stop driving the data bus during this interval. The CPU will sample the data
bus somewhere before or exactly when !CS goes inactive.
 - The recovery period will add delays between two operations.

The data bus width will influence if the CPU does full 16 bits reads, or only
8 bits. When doing 32 bits operations, the CPU will issue 2 16-bits operations,
or 4 8-bits operations, keeping !CS active the whole time, and strobing !WR or
!RD accordingly. When doing these sequences, the address bus will also increment
automatically between each operation, if the auto-increment bit is active.

This means it is possible to slightly shorten the read time of 4 bytes off the
same address by disabling auto-increment, and reading a full word. The CPU will
then read 4 bytes off the same address, and place them all into each byte of
the loaded register.

The DMA timing override portion will replace the access timing when doing DMA,
only if the DMA override flag is set.

The Wide DMA flag will enable full 32 bits DMA operations on the bus, by reusing
the low 16-bits address signals as the high 16-bits data. This means that if
the CPU is doing Wide DMA reads, the low 16-bits of the address bus will become
inputs.

Trying to access addresses that exceed the selected size causes a bus exception.
Maximum size would be Expansion 1 = 17h (8MB), BIOS = 16h (4MB), Expansion 2 =
0Dh (8KB), Expansion 3 = 15h (2MB). Trying to select larger sizes would overlap
the internal I/O ports, and crash the PSX. The Size bits seem to be ignored for
SPU/CDROM. The SPU timings seem to be applied for both the 200h-byte SPU region
at 1F801C00h and for the 200h-byte unknown region at 1F801E00h.<br/>

#### 1F801020h - COM\_DELAY / COMMON\_DELAY (00031125h or 0000132Ch or 00001325h)
```
  0-3   COM0 - Recovery period cycles
  4-7   COM1 - Hold period cycles
  8-11  COM2 - Floating release cycles
  12-15 COM3 - Strobe active-going edge delay
  16-31 Unknown/unused (read: always 0000h)
```
This register contains clock cycle offsets that can be added to the Access Time
values in Port 1F801008h..1Ch. Works (somehow) like so:<br/>
```
  1ST=0, SEQ=0, MIN=0
  IF Use_COM0 THEN 1ST=1ST+COM0-1, SEQ=SEQ+COM0-1
  IF Use_COM2 THEN 1ST=1ST+COM2,   SEQ=SEQ+COM2
  IF Use_COM3 THEN MIN=COM3
  IF 1ST<6 THEN 1ST=1ST+1   ;(somewhat like so)
  1ST=1ST+AccessTime+2, SEQ=SEQ+AccessTime+2
  IF 1ST<(MIN+6) THEN 1ST=(MIN+6)
  IF SEQ<(MIN+2) THEN SEQ=(MIN+2)
```
The total access time is the sum of First Access, plus any Sequential
Access(es), eg. for a 32bit access with 8bit bus: Total=1ST+SEQ+SEQ+SEQ.<br/>
If the access is done from code in (uncached) RAM, then 0..4 cycles are added
to the Total value (the exact number seems to vary depending on the used COMx
values or so).<br/>

#### 1F801060h - RAM\_SIZE (R/W) (usually 00000B88h) (or 00000888h)
```
  0-2   Unknown (no effect)
  3     Crashes when zero (except PU-7 and EARLY-PU-8, which <do> set bit3=0)
  4-6   Unknown (no effect)
  7     Delay on simultaneous CODE+DATA fetch from RAM (0=None, 1=One Cycle)
  8     Unknown (no effect) (should be set for 8MB, cleared for 2MB)
  9-11  Define 8MB Memory Window (first 8MB of KUSEG,KSEG0,KSEG1)
  12-15 Unknown (no effect)
  16-31 Unknown (Garbage)
```
Possible values for Bit9-11 are:<br/>
```
  0 = 1MB Memory + 7MB Locked
  1 = 4MB Memory + 4MB Locked
  2 = 1MB Memory + 1MB HighZ + 6MB Locked
  3 = 4MB Memory + 4MB HighZ
  4 = 2MB Memory + 6MB Locked              ;<--- would be correct for PSX
  5 = 8MB Memory                           ;<--- default by BIOS init
  6 = 2MB Memory + 2MB HighZ + 4MB Locked     ;<-- HighZ = Second /RAS
  7 = 8MB Memory
```
The BIOS initializes this to setting 5 (8MB) (ie. the 2MB RAM repeated 4 times),
although the "correct" would be setting 4 (2MB, plus other 6MB Locked). The
remaining memory, after the first 8MB, and up to the Expansion/IO/BIOS region
seems to be always Locked.<br/>
The HighZ regions are FFh-filled, that even when grounding data lines on the
system bus (ie. it is NOT a mirror of the PIO expansion region).<br/>
Locked means that the CPU generates an exception when accessing that area.<br/>
Note: Wipeout uses a BIOS function that changes RAM\_SIZE to 00000888h (ie. with
corrected size of 2MB, and with the unknown Bit8 cleared). Gundam Battle
Assault 2 does actually use the "8MB" space (with stacktop in mirrored RAM at
807FFFxxh).<br/>
Clearing bit7 causes many games to hang during CDROM loading on both EARLY-PU-8
and LATE-PU-8 (but works on PU-18 through PM-41).<br/>

#### FFFE0130h BIU_CONFIG, Cache Control (R/W)
```
  0-2   Unknown (Read/Write)                                            (R/W)
  3     Scratchpad Enable 1 (0=Disable, 1=Enable when Bit7 is set, too) (R/W)
  4-5   Unknown (Read/Write)                                            (R/W)
  6     Unknown (read=always zero)                  (R) or (W) or unused..?
  7     Scratchpad Enable 2 (0=Disable, 1=Enable when Bit3 is set, too) (R/W)
  8     Unknown                                                         (R/W)
  9     Crash (0=Normal, 1=Crash if code-cache enabled)                 (R/W)
  10    Unknown (read=always zero)                  (R) or (W) or unused..?
  11    Code-Cache Enable (0=Disable, 1=Enable)                         (R/W)
  12-31 Unknown                                                         (R/W)
```
Used primarily to flush the i-cache (in combination with COP0), like so:<br/>
```
 Init Cache Step 1:
  [FFFE0130h]=00000804h, then set cop0_sr=00010000h, then
  zerofill each FOURTH word at [0000..0FFFh], then set cop0_sr=zero.
 Init Cache Step 2:
  [FFFE0130h]=00000800h, then set cop0_sr=00010000h, then
  zerofill ALL words at [0000h..0FFFh], then set cop0_sr=zero.
 Finish Initialization:
  read 8 times 32bit from [A0000000h], then set [FFFE0130h]=0001E988h
```
At least one game (TOCA World Touring Cars, SLES-02572) flushes the cache
without using code from uncached ram (KSEG1) instead of calling the BIOS
function described above. It follows this sequence:
- First, it reads the current value of the BIU register, and stores it back
with this modification: `BIU = (BIU & ~0x0483) | 0x0804`. With the normal
BIU value of 0x001e988, this results in writing the value 0x001e90c
- Then, it writes 0x00010000 to cop0's Status register. This order is
important, as it would otherwise not take the BIU change, and still write
to main ram instead of the i-cache region.
- At this point, it proceeds with the step 1 of the flushcache procedure
of clearing every fourth word.
- Finally, it restores both the original value of the BIU and Status
registers it had previously saved.

A usable version of this code
[is available](https://github.com/pcsx-redux/nugget/blob/main/common/hardware/flushcache.s).

Note: FFFE0130h is described in LSI's "L64360" datasheet, chapter 14 (and
probably also in their LR33300/LR33310 datasheet, if it were available in
internet).<br/>
