# Plan: Webcam grayscale baseline (Option 1.4) + FPGA object detection extension

## Context
Option 1.4 (camera image processing on FPGA) is the baseline: webcam -> laptop -> UART -> FPGA grayscale -> laptop. The user wants to extend it to detect objects. This is feasible on a Basys 3 / Nexys A7 (Artix-7, no CPU) if "detection" means classic pixel-level methods: color-threshold blob detection or frame-difference motion detection with a bounding box. A neural-net detector is not realistic on this board (BRAM and no CPU), so it is out of scope. The repo currently has only README.md and the proposal, so everything is new.

## Feasibility answer
Yes, for color/motion blob detection on a 64x64 image: it needs only per-pixel compares plus min/max trackers (bounding box) and a pixel counter. A 64x64 frame is 12 KB RGB or 4 KB gray, which fits in BRAM. Real-time video is not needed: still frames over UART (115200 baud ~ 1 s per frame, up to 921600 baud for faster) are enough.

## Approach
Repo layout:
- `rtl/uart_rx.v`, `rtl/uart_tx.v` - 8N1 UART, parameterized baud
- `rtl/gray.v` - gray = (77R + 150G + 29B) >> 8 (fits in 16 bits, no overflow; note it in a comment)
- `rtl/threshold.v` - BW mode, threshold from switches
- `rtl/detect.v` - color-range (or |frame - previous| > T) mask, tracks min/max x,y and count
- `rtl/top.v` - frame FSM, mode select by switches, LEDs/7-seg show detected/count
- `constraints/basys3.xdc` - clk, UART, switches, LEDs
- `sim/*_tb.v` - testbenches per module with known RGB values
- `host/capture.py` - OpenCV capture + resize 64x64, pyserial send/receive with a frame header (0xAA 0x55 + mode byte), shows original / gray / BW / detection box side by side

Protocol: header, mode, 12288 RGB bytes in; response header + 4096 result bytes + bounding box (4 bytes) + count (2 bytes). Header resync addresses the synchronization challenge in the proposal.

## Milestones
1. UART loopback on board (verify COM port and baud).
2. Grayscale module: simulate, then compare returned pixels to a test image computed in Python.
3. Threshold mode via switches (two threshold settings for the demo).
4. Detection mode: color-range mask and bounding box; add motion mode if time allows.
5. Demo: test image + webcam snapshot; show gray, BW at two thresholds, detection box.

## Verification
- Simulation: run each testbench (Vivado xsim or iverilog) and check against known values (e.g. (255,0,0) -> 76, (255,255,255) -> 255).
- Hardware: Python script sends a known test image, then asserts FPGA output equals a NumPy reference (exact match for gray/BW; box coordinates match a NumPy mask).
- End to end: webcam snapshot of a colored object; confirm box sits on the object.

## Open items / proposal fixes
- Fix typos ("FGPA", "progressing") and state in the proposal that this is Option 1.4 extended, not Option 2.
- Confirm exact board model and Vivado version.
