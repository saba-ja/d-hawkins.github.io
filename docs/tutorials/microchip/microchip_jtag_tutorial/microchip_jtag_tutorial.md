---
author:
- D. W. Hawkins (dwh@caltech.edu)
bibliography:
- sections/refs.bib
date: 2025-03-04
title: Microchip FPGA JTAG Tutorial
---

# Introduction {#sec:intro}

This tutorial demonstrates how to use the Microchip FPGA UJTAG
component [@Microchip_AC227_2015]. The Microchip UJTAG component
provides a communications path over the JTAG[^1] interface to a user
design implemented in the programmable logic fabric of an FPGA[^2].
Similar JTAG-to-fabric interfaces from other FPGA vendors are; Altera
Virtual JTAG [@Altera_VJTAG_2021] and Xilinx BSCAN [@Xilinx_UG908_2024].
Figure [1](#fig:jtag_tap){reference-type="ref" reference="fig:jtag_tap"}
shows the JTAG (IEEE 1149.1) Test Access Port (TAP) Controller. JTAG TAP
state transitions are controlled by the JTAG clock (TCK) and test mode
select (TMS) control.

This tutorial includes:

-   Hardware implementation of the UJTAG component

-   UJTAG software interfacing using FlashPro Express and STAPL

-   UJTAG software interfacing using FTDI D2XX under Windows and Linux

The FTDI D2XX [@FTDI_D2XX_PG_2023] example applications were tested
under Windows 10 and Ubuntu 24.04[^3].

<figure id="fig:jtag_tap">
<div class="center">
<embed src="../figures/jtag_tap.pdf" />
</div>
<figcaption>JTAG (IEEE 1149.1) Test Access Port (TAP) Controller <span
class="citation" data-cites="IEEE_STD_1149_1_2013"></span>.</figcaption>
</figure>

# Resources {#sec:resources}

Resources related to this tutorial are:

-   Libero SoC Download:

    -   [https://www.microchip.com/en-us/products/fpgas-and-plds/
        fpga-and-soc-design-tools/fpga/libero-software-later-versions](https://www.microchip.com/en-us/products/fpgas-and-plds/fpga-and-soc-design-tools/fpga/libero-software-later-versions)

    -   Libero SoC 2024.1 was used during the development of this
        tutorial.

-   Polarfire SoC Discovery Kit (MPFS-DISCO-KIT):

    -   <https://www.microchip.com/en-us/development-tool/mpfs-disco-kit>

-   FTDI D2XX software tutorial

    -   <https://github.com/d-hawkins/ftdi_d2xx_tutorial>

-   Altera JTAG tutorials

    -   [altera_jtag_to_avalon_mm_tutorial.pdf](https://github.com/d-hawkins/altera_jtag_to_avalon_mm_tutorial/blob/main/doc/altera_jtag_to_avalon_mm_tutorial.pdf)

    -   [altera_jtag_to_avalon_analysis.pdf](https://github.com/d-hawkins/altera_jtag_to_avalon_analysis/blob/main/doc/altera_jtag_to_avalon_analysis.pdf)

    -   [altera_virtual_jtag_analysis.pdf](https://github.com/d-hawkins/altera_virtual_jtag_analysis/blob/main/doc/altera_virtual_jtag_analysis.pdf)

-   Xilinx JTAG tutorial (in progress)

The bibliography contains additional references.

# UJTAG {#sec:ujtag}

## Overview

The Microchip UJTAG component is described in
AC227 [@Microchip_AC227_2015] and in the
[ProASIC3](https://www.microchip.com/en-us/products/fpgas-and-plds/fpgas/proasic-3-fpgas)
Fabric Users Guide [@Microchip_PA3_UG_2012]. The UJTAG component is also
described in the Macro Library Users Guide for each device.
Figure [2](#fig:polarfire_macro_ujtag){reference-type="ref"
reference="fig:polarfire_macro_ujtag"} shows the Microchip UJTAG
component from the PolarFire Macro Library in the Libero SoC online
documentation for v2024.2. UJTAG is also supported in
[IGLOO2](https://www.microchip.com/en-us/products/fpgas-and-plds/fpgas/igloo-2-fpgas),
[SmartFusion2](https://www.microchip.com/en-us/products/fpgas-and-plds/system-on-chip-fpgas/smartfusion-2-fpgas),
[Polarfire](https://www.microchip.com/en-us/products/fpgas-and-plds/fpgas/polarfire-fpgas/polarfire-mid-range-fpgas),
and [PolarFire
SoC](https://www.microchip.com/en-us/products/fpgas-and-plds/system-on-chip-fpgas/polarfire-soc-fpgas)
devices. The example designs in this tutorial target evaluation boards
for the ProASIC3, SmartFusion2, and PolarFire SoC devices.

The UJTAG component provides user logic access to the JTAG clock, serial
data in and out, the 8-bit instruction register, and four TAP state
indicators. The serial data in and TAP state indicators change on the
falling-edge of the JTAG clock. A user interface to the UJTAG component
typically consists of;

-   A shift-register used for serial data in and out (shifted
    LSB-to-MSB)

-   URSTB used as asynchronous reset

-   UDRCK used as the clock

-   UIREG\[7:0\] used as a select (or multiplex) control

-   UDRCAP used to load the shift-register

-   UDRSH used to enable the shift-register

-   UDRUPD used to capture the shift-register into a parallel register

-   UTDO driven by the shift-register LSB

A user interface to the UJTAG component does not have to use all UJTAG
signals. For example, an SPI-like interface can be constructed using;

-   SPI clock is UDRCK

-   SPI select generated from a UIREG pattern (or bit) and UDRSH

-   SPI MOSI is UTDI

-   SPI MISO is UTDO

The SPI analogy works well for hardware designs containing only a single
device in the JTAG chain. The serial data stream is slightly different
for hardware designs containing multiple devices daisy-chained in the
JTAG chain. When communicating with a single device on a multi-device
JTAG chain, the other devices are placed in BYPASS mode, and each
bypassed device introduces a zero valued bit into the serial data
stream. These extra bits can be handled by detecting the number of
devices in BYPASS mode before, and after, the selected device.

The UJTAG UTDO input must be driven on the rising-edge of UDRCK, as the
UJTAG contains a falling-edge register on the path to the JTAG TDO
output.
Figure [8](#fig:jtag_to_register_questasim_waveforms){reference-type="ref"
reference="fig:jtag_to_register_questasim_waveforms"} shows the
Questasim simulation waveforms of the JTAG-to-Register design. The
serial data on TDI is 0x55 = 0101_0101b, while the serial data out on
TDO is 0x11 = 0001_0001b. The UTDO data, highlighted in cyan, is clocked
on the rising-edge of DRCK within the fabric logic. The TDO changes are
observed to occur on the falling-edge of TCK. The operation of UTDO is
inconsistent with AC227, where Figure 4 on p3 indicates that UTDO =
TDO [@Microchip_AC227_2015], and is inconsistent with the ProASIC3
fabric users guide statement on p381 that *UTDI, UTDO, and UDRCK are
directly connected to the JTAG TDI, TDO, and TCK ports,
respectively.* [@Microchip_PA3_UG_2012]. The UJTAG simulation models for
the ProASIC3 (Libero IDE 9.2 SP4 `proasic3e.v` library) and for the
SmartFusion2 (Libero SoC 2024.1 `smartfusion2.v` library) both exhibit
the same UTDO-to-TDO pipelining through a falling-edge register (see the
testbenches in `ip/ujtag`).

Figures [3](#fig:sf2_starter_ujtag_ila_tdo_rise){reference-type="ref"
reference="fig:sf2_starter_ujtag_ila_tdo_rise"}
and [4](#fig:sf2_starter_ujtag_ila_tdo_fall){reference-type="ref"
reference="fig:sf2_starter_ujtag_ila_tdo_fall"} show logic analyzer
traces captured from the SmartFusion2 Starter Kit for UJTAG designs that
drove UTDO on the rising-edge and falling-edge of UDRCK respectively.
The SmartFusion 2 Starter Kit was used, as it contains both an FPGA JTAG
header and an ARM debug header, so the JTAG signals could be probed on
the ARM header. The JTAG communications were implemented by an FTDI D2XX
applications. The logic analyzer traces were captured using the Digilent
Arty board and Xilinx Vivado.

-   [**TDO driven on UDRCK rising-edge**]{style="color: OliveGreen"}

    The TDO bits in
    Figure [3](#fig:sf2_starter_ujtag_ila_tdo_rise){reference-type="ref"
    reference="fig:sf2_starter_ujtag_ila_tdo_rise"} match the expected
    values.

-   [**TDO driven on UDRCK falling-edge**]{style="color: red"}

    The TDO bits in
    Figure [4](#fig:sf2_starter_ujtag_ila_tdo_fall){reference-type="ref"
    reference="fig:sf2_starter_ujtag_ila_tdo_fall"} do not match the
    expected values: the expected bit values are delayed by 1-bit, with
    the 2 LSBs repeated.

::: center
[**Conclusion:** UTDO must be clocked on UDRCK
rising-edge.]{style="color: magenta"}
:::

<figure id="fig:polarfire_macro_ujtag">
<div class="minipage">
<div class="center">
<p><embed src="../figures/polarfire_macro_ujtag.pdf"
style="width:100.0%" /><br />
(a) Component</p>
</div>
</div>
<div class="minipage">
<div class="center">
<table>
<thead>
<tr>
<th style="text-align: left;">Pin</th>
<th style="text-align: left;">Function</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: left;"></td>
<td style="text-align: left;"></td>
</tr>
<tr>
<td style="text-align: left;"></td>
<td style="text-align: left;"><strong>JTAG</strong></td>
</tr>
<tr>
<td style="text-align: left;">TRSTB</td>
<td style="text-align: left;">Reset (active low)</td>
</tr>
<tr>
<td style="text-align: left;">TCK</td>
<td style="text-align: left;">Clock</td>
</tr>
<tr>
<td style="text-align: left;">TMS</td>
<td style="text-align: left;">Test mode select</td>
</tr>
<tr>
<td style="text-align: left;">TDI</td>
<td style="text-align: left;">Serial data in</td>
</tr>
<tr>
<td style="text-align: left;">TDO</td>
<td style="text-align: left;">Serial data out</td>
</tr>
<tr>
<td style="text-align: left;"></td>
<td style="text-align: left;"></td>
</tr>
<tr>
<td style="text-align: left;"></td>
<td style="text-align: left;"><strong>TAP State</strong></td>
</tr>
<tr>
<td style="text-align: left;">URSTB</td>
<td style="text-align: left;">Test-Logic-Reset</td>
</tr>
<tr>
<td style="text-align: left;">UDRCAP</td>
<td style="text-align: left;">Capture-DR</td>
</tr>
<tr>
<td style="text-align: left;">UDRSH</td>
<td style="text-align: left;">Shift-DR</td>
</tr>
<tr>
<td style="text-align: left;">UDRUPD</td>
<td style="text-align: left;">Update-DR</td>
</tr>
<tr>
<td style="text-align: left;"></td>
<td style="text-align: left;"></td>
</tr>
<tr>
<td style="text-align: left;"></td>
<td style="text-align: left;"><strong>User</strong></td>
</tr>
<tr>
<td style="text-align: left;">UIREG[7:0]</td>
<td style="text-align: left;">Instruction</td>
</tr>
<tr>
<td style="text-align: left;">UDRCK</td>
<td style="text-align: left;">Clock</td>
</tr>
<tr>
<td style="text-align: left;">UTDI</td>
<td style="text-align: left;">Serial data in</td>
</tr>
<tr>
<td style="text-align: left;">UTDO</td>
<td style="text-align: left;">Serial data out</td>
</tr>
<tr>
<td style="text-align: left;"></td>
<td style="text-align: left;"></td>
</tr>
</tbody>
</table>
<p><br />
(b) Pin functions</p>
</div>
</div>
<figcaption>Microchip UJTAG component.</figcaption>
</figure>

<figure id="fig:sf2_starter_ujtag_ila_tdo_rise">
<div class="center">
<p><img src="../figures/sf2_starter_ujtag_ila_tdo_rise_0x09_0x08.png"
style="width:95.0%" alt="image" /><br />
(a) Write 0x09 = 0000_<span style="color: blue">1001</span>b. Read 0x08
= 0000_<span style="color: blue">1000</span>b</p>
</div>
<div class="center">
<p><img src="../figures/sf2_starter_ujtag_ila_tdo_rise_0x0A_0x09.png"
style="width:95.0%" alt="image" /><br />
(b) Write 0x0A = 0000_<span style="color: blue">1010</span>b. Read 0x09
= 0000_<span style="color: blue">1001</span>b</p>
</div>
<div class="center">
<p><img src="../figures/sf2_starter_ujtag_ila_tdo_rise_0x0B_0x0A.png"
style="width:95.0%" alt="image" /><br />
(c) Write 0x0B = 0000_<span style="color: blue">1011</span>b. Read 0x0A
= 0000_<span style="color: blue">1010</span>b</p>
</div>
<div class="center">
<p><img src="../figures/sf2_starter_ujtag_ila_tdo_rise_0x0C_0x0B.png"
style="width:95.0%" alt="image" /><br />
(d) Write 0x0C = 0000_<span style="color: blue">1100</span>b. Read 0x0B
= 0000_<span style="color: blue">1011</span>b</p>
</div>
<figcaption>JTAG logic analyzer traces for UTDO driven on UDRCK
rising-edge.</figcaption>
</figure>

<figure id="fig:sf2_starter_ujtag_ila_tdo_fall">
<div class="center">
<p><img src="../figures/sf2_starter_ujtag_ila_tdo_fall_0x09_0x10.png"
style="width:95.0%" alt="image" /><br />
(a) Write 0x09 = 0000_<span style="color: blue">1001</span>b. Read 0x10
= 000<span style="color: blue">1_000</span><span
style="color: red">0</span>b</p>
</div>
<div class="center">
<p><img src="../figures/sf2_starter_ujtag_ila_tdo_fall_0x0A_0x13.png"
style="width:95.0%" alt="image" /><br />
(b) Write 0x0A = 0000_<span style="color: blue">1010</span>b. Read 0x13
= 000<span style="color: blue">1_001</span><span
style="color: red">1</span>b</p>
</div>
<div class="center">
<p><img src="../figures/sf2_starter_ujtag_ila_tdo_fall_0x0B_0x14.png"
style="width:95.0%" alt="image" /><br />
(c) Write 0x0B = 0000_<span style="color: blue">1011</span>b. Read 0x14
= 000<span style="color: blue">1_010</span><span
style="color: red">0</span>b</p>
</div>
<div class="center">
<p><img src="../figures/sf2_starter_ujtag_ila_tdo_fall_0x0C_0x17.png"
style="width:95.0%" alt="image" /><br />
(d) Write 0x0C = 0000_<span style="color: blue">1100</span>b. Read 0x17
= 000<span style="color: blue">1_011</span><span
style="color: red">1</span>b</p>
</div>
<figcaption>JTAG logic analyzer traces for UTDO driven on UDRCK
falling-edge.</figcaption>
</figure>

## UJTAG Testbench

The simplest usage of the UJTAG component is a serial-to-parallel
register with write (host-to-FPGA) and read (FPGA-to-host) support. This
sort of interface is commonly found in SPI devices. For example, the
Texas Instruments LMX2615-SP clock synthesizer uses a 24-bit SPI
transaction consisting of;

-   1-bit read(1)/write(0) indicator

-   7-bit address (128 possible registers)

-   16-bit data

Another example is the Texas Instruments LMK04832 clock synthesizer
which also uses a 24-bit SPI transaction, but with a slightly different
bit functionality;

-   1-bit read(1)/write(0) indicator

-   15-bit address (32768 possible registers)

-   8-bit data

The UJTAG component could be used to create this interface, but for the
purpose of this tutorial, just the serial-to-parallel register is
implemented, without interpretation of the read/write register bits.

The UJTAG testbench is located in the repository directory `ip/ujtag`.
The UJTAG testbench is run as follows;

1.  Start Questasim (2023.4 was used for this tutorial)

2.  Change directory to the project, eg.,

        vsim> cd {C:\github\microchip_jtag_tutorial\ip\ujtag}

3.  Source the simulation script, eg.,

        vsim> source scripts/questasim.tcl

    The script ends with a list of the testbench procedures

        # Testbench Procedures
        # --------------------
        #
        #  ujtag_pa3_tb - run the UJTAG ProASIC3 testbench
        #  ujtag_sf2_tb - run the UJTAG Smartfusion2 (and newer) testbench
        #

4.  Run the ProASIC3 UJTAG testbench

        vsim> ujtag_pa3_tb

5.  Run the SmartFusion2 UJTAG testbench

        vsim> ujtag_sf2_tb

There are two UJTAG testbenches as the ProASIC3 (and earlier) devices
used a slightly different port naming convention than the SmartFusion2
(and newer) devices. A UJTAG wrapper component is used to update the
ProASIC3 port mapping to match that of the SmartFusion2 port mapping,
and the testbench instantiates the wrapper. See the source code for
additional details.

The UJTAG testbench implements the JTAG-to-Register component within the
testbench. The next section describes the JTAG-to-Register implemented
as a separate SystemVerilog module.

# JTAG-to-Register {#sec:jtag_to_register}

## Architecture

<figure id="fig:jtag_to_register_diagram">
<div class="center">
<embed src="../figures/ujtag_to_register_diagram.pdf" />
</div>
<figcaption>JTAG-to-Register block diagram.</figcaption>
</figure>

The JTAG-to-Register component implements a single write (control) and
read (status) register. The register bit-width is controlled by a
SystemVerilog parameter, WIDTH.
Figure [5](#fig:jtag_to_register_diagram){reference-type="ref"
reference="fig:jtag_to_register_diagram"} shows a block diagram of the
architecture. The JTAG-to-Register component uses a single-bit input
select bit.
Figure [5](#fig:jtag_to_register_diagram){reference-type="ref"
reference="fig:jtag_to_register_diagram"} shows the select bit will
assert for any valid user instruction (0x10 to 0x7F). User designs could
use the IR to select (and multiplex UTDO) different blocks of user
logic.

Figure [6](#fig:jtag_to_register_timing_ir){reference-type="ref"
reference="fig:jtag_to_register_timing_ir"} shows the instruction
shift-register timing. The TDO data shown in the figure is the value
read from the PolarFire SoC Discovery Kit.
Figure [6](#fig:jtag_to_register_timing_ir){reference-type="ref"
reference="fig:jtag_to_register_timing_ir"} shows the data
shift-register timing. The figure shows TDI serializing 0x55 and TDO
serializing 0x11. These timing diagrams are reproduced by the
JTAG-to-Register testbench, and are discussed in the Questasim waveform
section.

::: landscape
<figure id="fig:jtag_to_register_timing_ir">
<div class="center">
<embed src="../figures/ujtag_to_register_timing_ir.pdf"
style="width:210mm" />
</div>
<figcaption>JTAG-to-Register JTAG instruction shift-register
timing.</figcaption>
</figure>
:::

::: landscape
<figure id="fig:jtag_to_register_timing_dr">
<div class="center">
<embed src="../figures/ujtag_to_register_timing_dr.pdf"
style="width:210mm" />
</div>
<figcaption>JTAG-to-Register JTAG data shift-register
timing.</figcaption>
</figure>
:::

## Testbench

The JTAG-to-Register component is general-purpose, so is located in the
repository intellectual property (IP) directory `ip/jtag_to_register`.
The JTAG-to-Register testbench is run as follows;

1.  Start Questasim (2023.4 was used for this tutorial)

2.  Change directory to the project, eg.,

        vsim> cd {C:\github\microchip_jtag_tutorial\ip\jtag_to_register}

3.  Source the simulation script, eg.,

        vsim> source scripts/questasim.tcl

    The script ends with a list of the testbench procedures

        # Testbench Procedures
        # --------------------
        #
        #  jtag_to_register_tb - run the JTAG-to-Register testbench
        #

4.  Run the testbench

        vsim> jtag_to_register_tb

The testbench uses the SmartFusion2 UJTAG component.

## Waveforms

<figure id="fig:jtag_to_register_questasim_waveforms">
<div class="center">
<img src="../figures/jtag_to_register_questasim_waveforms.png" />
</div>
<figcaption>JTAG-to-Register testbench Questasim waveforms.</figcaption>
</figure>

Figure [8](#fig:jtag_to_register_questasim_waveforms){reference-type="ref"
reference="fig:jtag_to_register_questasim_waveforms"} shows the
Questasim waveform view captured from the JTAG-to-Register testbench,
with the JTAG-to-Register component configured for 8-bits control/status
width. The testbench interfaces to the JTAG signals and implements the
following sequence;

-   TAP reset

-   Load the 8-bit instruction register with 0x20

-   Load the 8-bit data register with 0x11 = 0001_0001b

-   Load the 8-bit data register with 0x55 = 0101_0101b

-   Load the 8-bit data register with 0x99 = 1001_1001b

-   TAP reset

Figure [8](#fig:jtag_to_register_questasim_waveforms){reference-type="ref"
reference="fig:jtag_to_register_questasim_waveforms"} shows the data
register load with 0x55. Items of interest in the waveform are;

-   TDI contains the LSB-to-MSB serial data 0x55 = 0101_0101b

-   TDO contains the LSB-to-MSB serial data 0x11 = 0001_0001b

-   UTDO updates on the rising-edge of TCK

-   TDO updates on the falling-edge of TCK

-   status is captureed during the CAPTURE TAP state

-   control is updated during the UPDATE TAP state

## Software Interfacing with STAPL

<figure id="fig:pfs_disco_and_arty">
<div class="center">
<p><img src="../figures/pfs_disco_and_arty.png" alt="image" /><br />
</p>
</div>
<figcaption>PolarFire SoC Discovery Kit and the Digilent Arty
Board.</figcaption>
</figure>

STAPL (Standard Test and Programming Language) is an Altera-developed
standard for JTAG interfacing, standardized in JEDEC standard
JESD-71 [@JEDEC_JESD71_1999]. Two STAPL files are located in the
repository:

-   `stp/read_idcode.stp`

    Read and print a Microchip JTAG device IDCODE.

    This script resets the TAP, loads the instruction register with the
    8-bit read IDCODE instruction, and then reads a single 32-bit
    IDCODE.

-   `stp/jtag_to_register.stp`

    Incrementing LED count.

    This script resets the TAP, loads the instruction register with the
    8-bit user instruction 0x20, and then loops from 1 to 15, loading
    the data register with the loop index, and printing the value read
    (which matches the previously loaded loop index).

The first version of the JTAG-to-Register STAPL was based on the example
in AC227 Appendix A [@Microchip_AC227_2015]. The JEDEC STAPL
specification was then consulted to understand the syntax for: loops,
integer-to-binary conversion, binary-to-integer conversion, and printing
messages.

Figure [9](#fig:pfs_disco_and_arty){reference-type="ref"
reference="fig:pfs_disco_and_arty"} shows the hardware setup used to
capture logic analyzer traces. The PolarFire SoC Discovery Kit Raspberry
Pi connector drove UJTAG signals to a PMod connector on the Digilent
Arty board, and the Xilinx Vivado [Integrated Logic Analyzer
(ILA)](https://www.xilinx.com/products/intellectual-property/ila.html)
was used to capture traces. FlashPro Express was used to execute the
JTAG-to-Register STAPL file.
Figure [10](#fig:jtag_to_register_stapl){reference-type="ref"
reference="fig:jtag_to_register_stapl"} shows logic analyzer traces for
TDI serializing 0x02, 0x03, and 0x04, and TDO serializing 0x01, 0x02,
and 0x03.

## Software Interfacing with FTDI D2XX

A JTAG-to-Register application was written in C++, compiled with
Microsoft Visual Studio 2022 (Community Edition), and linked with the
FTDI D2XX DLL (the application is part of the FTDI D2XX tutorial). The
application used the FTDI Multi-Protocol Synchronous Serial Engine
(MPSSE) mode to implement JTAG transactions.
Figure [11](#fig:jtag_to_register_d2xx){reference-type="ref"
reference="fig:jtag_to_register_d2xx"} shows logic analyzer traces for
TDI serializing 0x02, 0x03, and 0x04, and TDO serializing 0x01, 0x02,
and 0x03. The logic analyzer traces are very similar to those captured
from the FlashPro Express STAPL transactions in
Figure [10](#fig:jtag_to_register_stapl){reference-type="ref"
reference="fig:jtag_to_register_stapl"}, with one minor difference in
that the STAPL captures show additional TCK pulses while in
Run-Test-Idle.

<figure id="fig:jtag_to_register_stapl">
<div class="center">
<p><img src="../figures/jtag_to_register_stapl_0x02_0x01.png"
style="width:95.0%" alt="image" /><br />
(a) Write 0x02 = 0000_0010b. Read 0x01 = 0000_0001b</p>
</div>
<div class="center">
<p><img src="../figures/jtag_to_register_stapl_0x03_0x02.png"
style="width:95.0%" alt="image" /><br />
(b) Write 0x03 = 0000_0011b. Read 0x02 = 0000_0010b</p>
</div>
<div class="center">
<p><img src="../figures/jtag_to_register_stapl_0x04_0x03.png"
style="width:95.0%" alt="image" /><br />
(c) Write 0x04 = 0000_0100b. Read 0x03 = 0000_0011b</p>
</div>
<figcaption>JTAG-to-Register logic analyzer traces for STAPL
interfacing.</figcaption>
</figure>

<figure id="fig:jtag_to_register_d2xx">
<div class="center">
<p><img src="../figures/jtag_to_register_d2xx_0x02_0x01.png"
style="width:95.0%" alt="image" /><br />
(a) Write 0x02 = 0000_0010b. Read 0x01 = 0000_0001b</p>
</div>
<div class="center">
<p><img src="../figures/jtag_to_register_d2xx_0x03_0x02.png"
style="width:95.0%" alt="image" /><br />
(b) Write 0x03 = 0000_0011b. Read 0x02 = 0000_0010b</p>
</div>
<div class="center">
<p><img src="../figures/jtag_to_register_d2xx_0x04_0x03.png"
style="width:95.0%" alt="image" /><br />
(c) Write 0x04 = 0000_0100b. Read 0x03 = 0000_0011b</p>
</div>
<figcaption>JTAG-to-Register logic analyzer traces for FTDI D2XX
interfacing.</figcaption>
</figure>

## Hardware Implementations

Table [1](#tab:jtag_to_register_hardware){reference-type="ref"
reference="tab:jtag_to_register_hardware"} shows the JTAG-to-Register
design was implemented on the following hardware:

-   Microchip Polarfire SoC Discovery Kit
    ([MPFS-DISCO-KIT](https://www.microchip.com/en-us/development-tool/mpfs-disco-kit))

-   Microchip ProASIC3 Starter Kit
    ([A3PE-STARTER-KIT-2](https://www.microchip.com/en-us/development-tool/a3pe-starter-kit-2))

-   Microchip SmartFusion2 Security Evaluation Kit
    ([M2S090TS-EVAL-KIT](https://www.microchip.com/en-us/development-tool/m2s090ts-eval-kit))

-   Emcraft SmartFusion2 Starter Kit
    ([SF2-STARTER-KIT](https://emcraft.com/products/153))

    The tutorial used the ES version of this kit: SF2-STARTER-KIT-ES

-   Avnet SmartFusion2 Kickstart Kit (no longer available)

The UJTAG component is supported by other Microchip FPGAs, eg.,
[IGLOO2](https://www.microchip.com/en-us/development-tool/m2gl-eval-kit).

:::: center
::: {#tab:jtag_to_register_hardware}
  Directory                 Board                     Control
  ------------------------- ------------------------- ---------
                                                      
  `designs/pfs_disco`       PolarFire SoC Discovery   4 LEDs
  `designs/pa3_starter`     ProASIC3 Starter          7 LEDs
  `designs/sf2_security`    SmartFusion2 Security     7 LEDs
  `designs/sf2_starter`     SmartFusion2 Starter      1 LED
  `designs/sf2_kickstart`   SmartFusion2 Kickstart    3 LEDs
                                                      

  : JTAG-to-Register hardware implementations.
:::
::::

# Epilogue {#sec:epilogue}

This tutorial (and associated design files) demonstrates the use of the
Microchip UJTAG component: simulation, synthesis, and hardware
implementation. The UJTAG component and the JTAG-to-Register example
form the basis for more complex JTAG interface components:

-   **JTAG-to-AXI-Stream Bridge**

    The JTAG-to-AXI-Stream bridge interfaces JTAG serial data streams
    into byte streams compiliant with the ARM [AMBA AXI-Stream Protocol
    Specification](https://developer.arm.com/documentation/ihi0051/latest).
    The JTAG TDO bit-stream is converted to an AXI-Stream Transmitter
    byte-stream, while the JTAG TDI bit-stream is used to receive data
    from an AXI-Stream Receiver byte-stream.

-   **JTAG-to-AXI Bridge**

    The JTAG-to-AXI bridge interfaces JTAG serial data streams into
    transactions compliant with the ARM [AMBA AXI Protocol
    Specification](https://developer.arm.com/documentation/ihi0022/latest).
    The JTAG-to-AXI bridge combines the JTAG-to-AXI-Stream Bridge with
    an AXI-Stream-to-AXI Manager.

These components can be found on the same github as this tutorial.

# Revision History

::: center
      Date     Description
  ------------ ----------------------------------------------------
               
   01/15/2025  Created the document.
   01/19/2025  Completed the UJTAG and JTAG-to-Register sections.
   01/20/2025  First release on github.
               
:::

[^1]: Joint Test Action Group.

[^2]: Field Programmable Gate Array.

[^3]: Ubuntu was tested using an Oracle VirtualBox Virtual Machine (VM)
    running under Windows 10.
