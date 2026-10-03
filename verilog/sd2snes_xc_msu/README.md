# Xeno Crisis on sd2snes: the MSU-1 core (`sd2snes_xc_msu`)

This core plays the music from an MSU-1 pack (for example one made from the official soundtrack) instead of
decoding the game's Opus streams. **It runs on hardware.** The firmware picks it automatically when `<rom>.msu` is
next to the ROM and `/sd2snes/fpga_xc_msu.bi3` is on the card (`xc_msu_pack()` in `xc_load.c`); otherwise it loads
`fpga_xc.bi3`. Both use the same `xc_soc.bin`. It also gives the LPC1756 firmware (`firmware.im3`), which can't
decode Opus, a way to get music. Track numbers and lengths for pack makers: [`MSU1_PACK.md`](MSU1_PACK.md).

## FPGA

- A separate Quartus project (`sd2snes_xc_msu.qpf`; `make` builds `fpga_xc_msu.bi3`). Like `sd2snes_sgb_msu`, it
  takes the shared sources from `../sd2snes_xc` (soft CPU, `main.v`, `mcu_cmd.v`, IP), with `XC_MSU` defined.
- Its own files: `msu.v`, `xc_dac.v`, `xc_msubox.v`, `main.qsf` (the Opus core's, plus `XC_MSU` and the search path)
  and `main.sdc` (a copy of the Opus core's; keep them the same).
- `xc_msubox.v` replaces the Opus decode mailbox at the same soft CPU address: `CTRL` bit 3 reads 1, a write to
  `+0x10` is an MSU-1 register write (`$2000 + n`), `+0x14` reads the MSU-1 status byte.
- `msu.v` gets its register writes from the soft CPU only (the SNES doesn't see the MSU-1; the MCU clears
  `FEAT_MSU1`) and has no data buffer, since the game has no MSU-1 data (16 M9K saved).
- `xc_dac.v` is `dac.v` with linear interpolation instead of the 3-stage CIC (about 520 LUTs + 4 DSP multipliers
  instead of about 1,600 LUTs); same buffer, MCU interface, 44.1 kHz timing, volume ramp and I2S output.
- Net: about 500 LEs more than the Opus core and 6 M9K fewer.

## Soft CPU (`src/xc_soc/xc_mix.c`, MSU-1 mode)

- The game's music position still advances at the same pace (packets count as decoded, nothing goes to the MCU),
  so its end-of-track and loop logic are unchanged. The music is not mixed into the BRR stream; the sound effects are.
- When the game starts a track from the beginning, the mixer requests MSU-1 track n (the stream's number in the
  firmware's table). A track the game loops onto itself plays with MSU-1 repeat. An intro plays once, and its loop
  part starts as soon as the intro `.pcm` ends.
- Pause and resume map to the MSU-1 play bit; a reset stops the MSU-1; a missing `.pcm` is silent. The mixer waits
  for "audio busy" to clear before writing the control register.

## MCU

The usual `msu1_loop()`, with one change for Xeno Crisis: it checks the save RAM every second even while music
plays, regardless of the MSU-1 autosave setting, because this is the game's only save path while it runs. During
that CRC and the save, `xc_audio_service()` refills the MSU-1 audio buffer (`msu1_audio_service()`).

## Tests

- **MesenCE** routes the soft CPU's MSU-1 registers to its MSU-1 emulation when a `.msu` is next to the ROM
  (`XC_MSU=0` forces Opus mode). With a pack of test tones, the register writes and recorded audio were right for
  intros, repeats, intro → loop, pause/resume, missing tracks and reset.
- **`fpga/msutest/tb_msu.v`** checks `xc_msubox` + `msu.v` + `xc_dac.v`: register writes and status, "audio busy",
  44.1 kHz input rate, and the same output level and channel order as `dac.v`.

Everything else (soft CPU, memory layout, loader) is as in [`../sd2snes_xc/README.md`](../sd2snes_xc/README.md).
