# Primary hardware references

Source links are **manufacturer primary references** gathered during architecture planning. Always inspect the latest relevant revision and package before freezing a symbol/footprint or values. A URL alone is not validation that the circuit meets project requirements.

| Topic | Manufacturer source | Extracted decision-relevant facts |
|---|---|---|
| TI ADS8900B product | https://www.ti.com/product/ADS8900B | 20-bit, 1 MSa/s; enhanced SPI / parity; VREF 2.5–5 V; 104.5 dB typical SNR; nominal 22 MHz interface at 1 MSa/s |
| TI ADS8900B data sheet | https://www.ti.com/lit/ds/symlink/ads8900b.pdf | Analog input range, digital I/O, reference, timing, typical reference driver and layout recommendations — REVIEW PIN-LEVEL BEFORE USE |
| ADI AD8421 data sheet | https://www.analog.com/media/en/technical-documentation/data-sheets/ad8421.pdf | 3.2 nV/sqrt(Hz) max at 1 kHz, 200 fA/sqrt(Hz) input current noise, 10 MHz BW (G=1), 2 MHz BW (G=100); supplies ±2.5 to ±18 V; note source-impedance behavior |
| ADI AD8421 product | https://www.analog.com/en/products/ad8421.html | Orderable packages and recommended use |
| TI THS4551 product | https://www.ti.com/product/THS4551 | 2.7–5.4 V supply; 3.3 nV/sqrt(Hz) typical differential input voltage noise; suitable SAR ADC driver; verify common mode and output headroom |
| ADI AD9833 product | https://www.analog.com/en/products/ad9833.html | 28-bit DDS, max 25 MHz MCLK, output formats, SPI setup; fin = MCLK × tuning_word / 2^28; frequency resolution depends on MCLK |
| ADI AD9833 data sheet | https://www.analog.com/media/en/technical-documentation/data-sheets/ad9833.pdf | Phase/register reset sequencing, output characteristics, reconstruction/filtering, clock |
| ADI AD9833 No-OS software | https://wiki.analog.com/resources/tools-software/uc-drivers/ad9833 | Existing reference driver for bring-up/cross-check |
| TI ISOUSB111 datasheet | https://www.ti.com/lit/ds/symlink/isousb111.pdf | Full-speed 12 Mbps USB; dual-side supply and isolation barrier requirements; system-level isolation NOT assured |
| TI STM32H723 reference | https://www.st.com/en/microcontrollers-microprocessors/stm32h723-733.html | MCU family capabilities; **verify chosen exact variant's latest datasheet, reference manual and errata** |
| TI REF5045 reference | https://www.ti.com/product/REF5045 | Precision 4.5 V reference; verify output current, temperature drift, 1/f and broadband noise, buffer requirements |

Correction: STM32H723 is an **STMicroelectronics**, not TI, part; the linked ST product page is authoritative. This document has not yet audited exact STM32H723 variant/errata. Likewise, prefer the latest approved ADI AD9833 URL if manufacturer redirects.

## Comparison benchmark (NOT a Rev A guarantee)

Zurich Instruments MFLI published data and product reference: https://www.zhinst.com/products/mfli-lock-in-amplifier

Broadly, MFLI-class 500 kHz/5 MHz and 60 MSa/s hardware is a separate FPGA/high-speed architecture. Do not translate product specifications into Rev A promises.

## Engineering references to add before fab

- Latest TI ADS8900B evaluation-board files, exact THS4551 + REF5045 application schematic, required input drive network and reference bypass.
- Latest AD8421 reference circuit and SOIC package outline; exact datasheet revision and stable document archive.
- ST STM32H723 exact MCU datasheet, reference manual, SPI/TIM/DMA/MPU/cache application notes, current silicon errata.
- Actual DDS/clock TCXO datasheets; USB isolator/isolation power/creepage notes.
- Chosen relay, comparator, protection diode and power regulator primary datasheets.
- KiCad library library provenance and licence notices, appropriate assembly-fab capability rules.

**License note:** manufacturer data sheets and reference diagrams are sources for engineering, not a blanket licence to copy arbitrary graphics or proprietary evaluation design files into the repository. Prefer short factual references and independent schematics.
