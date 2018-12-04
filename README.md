# fm8086
A design for a fast-as-realistically-possible multiprocessing 8086 machine, built with components available in early-mid 1982.
Plan is for an initial version based on 5MHz i8086 chips, but with an easy upgrade path to 8MHz i8086-2 parts (the latter *were* available in 1982, but were hard to come by in major retailers so may have been expensive in low volumes).  The system should be able to run MS-DOS, but is not necessarily IBM PC compatible (e.g. doesn't necessarily support IBM BIOS calls, graphics hardware may be entirely different to any supported on the PC, memory mapping details may differ, etc).  Note that as of 1982 MS-DOS was distributed to OEMs in source code form with the intent that it should be customized to the hardware, rather than the other way around.  This source code is [now available](https://github.com/Microsoft/MS-DOS), so getting an appropriate version to run on such a machine should be possible.

An exception for the 1982 availability requirement for components is for any additions to support operation with modern hardware, e.g. the use of SD cards for storage, PS/2 keyboard/mouse connections and VGA output is likely, all of which is anachronistic, but necessary in order to realistically make such a machine useful today.  The former two components are most easily implemented by using a modern microcontroller (eg. AVR).  We can assume that an original machine of this kind would have used floppy disk controllers, parallel ASCII keyboards and composite video output, which is admittedly of slightly higher complexity, but is clearly not a difficult task to have achieved at the time, so is not especially interesting to replicate.

(Project status: early planning)

Overview
========

Design idea is to have a synchronous bus, running with a cycle rate at least double that of the CPU.  A second clock with a 90 degree phase shift will allow for devices to perform fine timing.  Devices negotiate access to the bus one half-cycle in advance, and the current holder of the bus can lock it to allow transactions that take multiple cycles (in most cases, this will be all bus transactions).

Memory will be DRAM, with a controller that uses page mode; DRAM chips will be selected to allow for response times:
* First request in a transaction: 3 bus cycles
* Subsequent requests (assumed to be within the same page): 2 bus cycles
* R/M/W requests: 4 bus cycles

The bus will be 16 bits wide, and 4164-type chips will be used for initial versions (thus giving a minimum configuration of 128KiB).

CPU cards will contain a single processor and a small quantity of cache memory.  This will be a small static RAM (likely 2 chips x 8 bits wide) that is fast enough to respond within a single bus cycle (e.g. 70ns for a 10MHz bus/5MHz processor system, or 55ns for a 16MHz bus/8MHz processor system).  

On simple CPU cards (which should be suitable for small systems), cache control will be [direct-mapped](https://en.wikipedia.org/wiki/Cache_Placement_Policies#Direct_Mapped_Cache) using a bank of fast bipolar static RAM chips (e.g. 74S189 or Am27S03 for 16x4) to store part of the address.  Data will be cached in (at least) 32-bit chunks, so for a 20-bit address, A1..A0 will be implicit by location of the byte within a cache entry, A7..A2 (6 bits, requiring 4 banks of S189s to store the corresponding tags) will determine the cache location, leaving A19..A8 (12 bits, so 3 chips per bank of 4-bit RAMs) of tag to be stored in the registers (requiring 12 total ICs).

Larger systems will require better caching, however, so a number of variants are proposed:

* Separating out code and data -- code is more likely to be accessed sequentially, so can be cached in larger chunks.
* Using larger memory stores to hold the tags, so more locations can be cached (e.g. 74S200/Am27S00 256x1, giving 8 bits of cache location in a single bank)
* Using specific ICs that are designed for cache tag storage -- e.g. the Am2150, which held 8 bits of tag + 1 of parity indexed by 9 bits (512 words) of address and integrated a comparator that gave a match/no match signal in 20ns -- much faster than we could do this ourselves with discrete ICs, so for fast systems (e.g. 16MHz bus rate, which will require a decision to request bus access within 32ns of getting a valid address from the processor if we don't want to miss the first available slot) this is likely the only viable approach.  This single-chip solution can give us full coverage for 17 bits of address, so works by itself for systems using 8-byte cache lines, or 2 can be used together if 4-byte cache lines are required.  Unfortunately, the chip is hard to acquire today, so would have to be emulated in a CPLD.  It's also not entirely clear when these chips were introduced; the earliest AMD memory databook I can find online is from 1986, and does include it; their 1979 Am2900 bitslice processor series databook (which includes related devices including memory) doesn't include it, or indeed any other TTL memory as large as 4.5Kbits, so I imagine it was introduced some time after 1979 but before 1986, but exactly when it became available I'm not sure.
