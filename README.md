# gnuradio-qpsk-simulation

A GNU Radio flowgraph that simulates a QPSK packet radio link entirely in software, using a ZeroMQ pub/sub loopback in place of real RF hardware (e.g. USRPs).

## Overview

The flowgraph reads data from a file, encapsulates it into packets, modulates it as differentially-encoded QPSK, and pushes the resulting complex baseband samples through a ZMQ PUB socket. A ZMQ SUB socket immediately consumes that same stream, acting as a simulated channel, and feeds it into a full receive chain (symbol timing recovery, Costas loop carrier recovery, an adaptive linear equalizer, and packet framing/CRC validation) before writing the recovered payload back out to a file. This makes it possible to test and tune an entire QPSK TX/RX chain — including access-code correlation and CRC-checked packet framing — without any hardware.

## Signal Chain

**Transmit:**
`File Source` → `Tagged Stream Mux` (header + payload) → `Constellation Modulator` (differential QPSK, RRC pulse shaping) → `ZMQ PUB Sink`

**Receive:**
`ZMQ SUB Source` → `Symbol Sync` (PFB matched filter timing recovery) → `Costas Loop` (carrier phase recovery) → `Linear Equalizer` (CMA adaptive algorithm) → `Constellation Decoder` → `Differential Decoder` → `Map`/`Unpack Bits` → `Correlate Access Code` (packet sync) → `Repack Bits` → `Stream CRC32` (validates payload) → `File Sink`

Packets are framed using GNU Radio's `header_format_default` (protocol formatter + CRC), with `digital_correlate_access_code_xx_ts` locating a 32-bit access code in the demodulated bitstream on the RX side to recover packet boundaries.

## Files

- `qpsk_simulation.grc` — GNU Radio Companion flowgraph.
- `qpsk_simulation.py` — Generated Python flowgraph (run directly with `python3 qpsk_simulation.py`, or regenerate from the `.grc` in GRC).
- `test.txt` — Sample input file transmitted through the pipeline.
- `output.tmp` — Recovered payload written by the receive chain (only populated once a packet passes CRC validation).

## Running

1. Open `qpsk_simulation.grc` in GNU Radio Companion (tested on GNU Radio 3.10) and Generate/Run, or run the generated Python directly:
   ```
   python3 qpsk_simulation.py
   ```
2. Constellation and time-domain scopes are available in the GUI tabs to inspect each stage of the chain (modulator output, symbol sync, Costas loop, equalizer, decoder).
3. Check `output.tmp` for the recovered contents of `test.txt` once packets start passing CRC validation.

## Notes

- The receive-side adaptive equalizer (CMA) and Costas loop need a settling period to converge before packets will pass CRC — expect the first several packets after startup to fail until the loops lock.
- `File Source` is set to repeat indefinitely so the chain has enough data to converge; a `Head` block can be added to bound test runs, but should allow enough repetitions for convergence.
