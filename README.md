# Huffman Decoder FPGA

This repository contains two synthesizable 32-symbol Huffman decoder architectures for FPGA implementation:

1. A barrel-shifter based decoder in `barrel_shifter_source_codes/`
2. A bit-pointer / sliding-window based decoder in `bit_pointer_source_codes/`

Both designs decode a 96-bit input packet into up to 32 output symbols of 5 bits each, with a valid mask indicating which symbol slots were produced. The two variants share the same top-level contract and the same packet drop rule:

- Input packet width: 96 bits
- Input bit count: 7 bits
- Output symbols: 32 lanes x 5 bits = 160 bits
- Output valid mask: 32 bits
- Drop rule: if the low 32 bits of the incoming packet are all zero, the packet is dropped

The two architectures differ in how each decode stage advances through the bitstream.

---

## High-Level View

Both decoders are built as a 32-stage pipeline. Each stage attempts one Huffman lookup, optionally emits one symbol, and then advances the packet state for the next stage.

```text
										+----------------------------+
in_valid ---------->| ingress decision / drop    |
in_bits[95:0] ----->| low-32-bit zero check      |
in_bit_count[6:0] -->| accept or drop packet      |
										+-------------+--------------+
																	|
																	v
									 +--------------------------------------+
									 | 32 repeated decode stages            |
									 | stage 0 -> stage 1 -> ... -> stage31 |
									 +------------------+-------------------+
																			|
																			v
														 out_valid, out_syms,
														 out_sym_valid
```

Each stage produces a local symbol decision and a valid flag. A separate delay chain aligns those per-stage results so the final output bus presents one complete 32-lane packet view.

---

## Common Behavior

Both implementations expose the same top-level ports:

- `clk`, `rst_n`
- `in_valid`, `in_bits[95:0]`, `in_bit_count[6:0]`
- `out_valid`, `out_syms[159:0]`, `out_sym_valid[31:0]`
- `in_decision_valid`, `in_drop`, `in_accept`

### Ingress decision

The ingress rule is purely combinational:

```text
if in_valid and in_bits[31:0] == 0:
		drop
else if in_valid:
		accept
```

The registered debug outputs `in_decision_valid`, `in_drop`, and `in_accept` are delayed by one clock so the testbenches can correlate the decision with the sampled input transaction.

### Output packing

Each stage contributes one `{valid, symbol}` pair. The output side repacks those pairs as:

- `out_sym_valid[j]` = valid bit for lane `j`
- `out_syms[j*5 +: 5]` = symbol value for lane `j`

The design uses `lane_delay_ff` to align the stage-local decisions across the pipeline depth.

```text
stage 0 result ---- delay 31 ---->
stage 1 result ---- delay 30 ---->
...
stage31 result ---- delay  0 ----> final lane map
```

---

## Architecture 1: Barrel Shifter Decoder

Files:

- [barrel_shifter_source_codes/huff_decoder32_local_rwoh.v](barrel_shifter_source_codes/huff_decoder32_local_rwoh.v)
- [barrel_shifter_source_codes/huff_dec_stage_local_rwoh.v](barrel_shifter_source_codes/huff_dec_stage_local_rwoh.v)
- [barrel_shifter_source_codes/huff_dec1_rom_look9.v](barrel_shifter_source_codes/huff_dec1_rom_look9.v)
- [barrel_shifter_source_codes/lane_delay_ff.v](barrel_shifter_source_codes/lane_delay_ff.v)

### Core idea

This architecture stores the packet as three 32-bit words:

- `word0` = bits `[31:0]`
- `word1` = bits `[63:32]`
- `word2` = bits `[95:64]`

Each stage reads the low 9 bits of `word0`, uses a ROM lookup to determine whether a symbol is valid and how many bits to consume, then advances the 96-bit view with a combinational barrel shift by 0 to 9 bits.

### Sketch

```text
in_bits[95:0]
	 |
	 +--> word0 / word1 / word2
					 |
					 v
	 +-----------------------+
	 | 9-bit ROM lookup      |
	 | look9 = word0[8:0]    |
	 | dec_valid / sym / len |
	 +-----------------------+
					 |
					 v
	 +-----------------------+
	 | barrel shift by len   |
	 | 0..9 bit mux network  |
	 +-----------------------+
					 |
					 v
 next stage sees advanced word0/1/2
```

### Stage-by-stage behavior

#### Stage entry

Each stage receives:

- `in_frame_valid`
- `in_active`
- `in_word0`, `in_word1`, `in_word2`
- `in_remaining`

The stage first checks whether the entire local state is zero:

- if all of `in_word0`, `in_word1`, `in_word2`, and `in_remaining` are zero, the frame is killed

That prevents useless propagation once the packet is exhausted.

#### Decode lookup

The 9-bit lookup key is:

```text
look9 = in_word0[8:0]
```

The ROM module [huff_dec1_rom_look9.v](barrel_shifter_source_codes/huff_dec1_rom_look9.v) uses the available bit count to select one of 10 length classes:

- remaining bits less than 9 use the exact remaining length
- remaining bits 9 or more use class 9

The address is therefore:

```text
{len_sel, look9}
```

The ROM returns:

- `dec_valid`
- `dec_sym`
- `dec_len`

#### Advance logic

The stage precomputes all candidate shifts from 0 to 9 bits:

```text
shift by 0  -> pass through
shift by 1  -> words moved right by 1 bit, refill from the next word
...
shift by 9  -> words moved right by 9 bits
```

The selected shift is driven directly from `dec_len`. This is why the architecture is called a barrel shifter decoder: every stage contains a parallel shift network and a mux selects the required offset.

#### Sequential commit

On the rising clock edge:

- `out_frame_valid` becomes `in_frame_valid && !zero_state`
- `out_active` becomes `do_decode`
- `out_vld_local` becomes `do_decode`
- if the stage is enabled, the advanced word bundle and the decoded symbol are committed
- otherwise the outputs are cleared to zero

### Why this architecture is fast conceptually

This version keeps the decode state very simple:

- no byte pointer
- no bit offset register
- no sliding window state
- only direct bit shifts over the 96-bit packet view

The tradeoff is that each stage builds a fairly wide combinational mux network for the 0..9-bit barrel shift.

---

## Architecture 2: Bit Pointer / Sliding Window Decoder

Files:

- [bit_pointer_source_codes/huff_decoder32_local_win24_offlut_norem_5a.v](bit_pointer_source_codes/huff_decoder32_local_win24_offlut_norem_5a.v)
- [bit_pointer_source_codes/huff_dec_stage_local_win24_offlut_norem_5a.v](bit_pointer_source_codes/huff_dec_stage_local_win24_offlut_norem_5a.v)
- [bit_pointer_source_codes/huff_dec1_rom_off9_norem.v](bit_pointer_source_codes/huff_dec1_rom_off9_norem.v)
- [barrel_shifter_source_codes/lane_delay_ff.v](barrel_shifter_source_codes/lane_delay_ff.v)

### Core idea

This architecture keeps the whole 96-bit packet, but it does not physically barrel-shift the entire stream on every stage. Instead it tracks:

- a one-hot byte pointer `byte_ptr_oh[11:0]`
- a bit offset `bit_off[2:0]`
- a 24-bit local window `win24[23:0]`

The ROM uses the pair `{bit_off, look9}` to decide:

- whether the code is valid
- the symbol value
- the code length
- how many bytes to advance
- the next bit offset

### Sketch

```text
in_frame_bits[95:0]
	 |
	 +--> byte pointer (one-hot)
	 +--> bit offset 0..7
	 +--> 24-bit window
						|
						v
		 {bit_off, look9} -> ROM -> dec_valid / sym / len / byte step / next offset
						|
						v
	 update pointer, offset, window, remaining
```

### Stage-by-stage behavior

#### Stage entry

Each stage receives:

- `in_frame_bits`
- `in_byte_ptr_oh`
- `in_bit_off`
- `in_remaining`
- `in_win24`

The local zero-state check is intentionally defined as:

- zero if `in_frame_bits`, `in_remaining`, and `in_win24` are all zero

The pointer and bit offset are excluded from this zero-state test so that valid pointer bookkeeping does not incorrectly terminate the packet.

#### Local lookup window

The 9-bit lookup key is formed by selecting the appropriate slice of the 24-bit window:

```text
bit_off = 0 -> look9 = in_win24[8:0]
bit_off = 1 -> look9 = in_win24[9:1]
...
bit_off = 7 -> look9 = in_win24[15:7]
```

This stage-local alignment makes the decoder tolerant of bit positions that are not byte-aligned.

#### ROM decode

The ROM module [huff_dec1_rom_off9_norem.v](bit_pointer_source_codes/huff_dec1_rom_off9_norem.v) returns a packed result:

- `code_valid`
- `dec_sym`
- `dec_len`
- `adv_bytes`
- `next_bit_off`

The address is the concatenation of bit offset and 9-bit lookahead:

```text
{bit_off, look9}
```

#### Advance logic

The stage computes the next byte pointer by applying the ROM-reported byte advance:

- 0 bytes: keep the pointer
- 1 byte: shift the one-hot pointer left by 1
- 2 bytes: shift the one-hot pointer left by 2

The next bit offset comes straight from the ROM.

The local 24-bit window is updated by refilling from the original 96-bit frame using helper selections:

- `refill3` when the advance is 1 byte
- `refill4` when the advance is 2 bytes

The new window is assembled from the refill bytes plus the existing window tail.

#### Sequential commit

On the rising clock edge:

- `out_frame_valid` becomes `in_frame_valid && !zero_state`
- `out_active` and `out_vld_local` assert only when a valid decode occurs
- on decode, the stage commits the updated frame bits, byte pointer, bit offset, remaining count, window, and symbol
- if the stage is not enabled, it clears the state back to zeros or defaults

### Why this architecture is different

This version moves complexity away from a wide shifter and into:

- pointer tracking
- bit-offset tracking
- local-window refilling logic
- a richer ROM encoding

That can be useful when the desired implementation style is to avoid large shift muxes in every stage.

---

## Side-by-Side Comparison

| Aspect | Barrel Shifter | Bit Pointer / Sliding Window |
|---|---|---|
| Packet state | 3 x 32-bit words | 96-bit frame + byte pointer + bit offset + 24-bit window |
| Lookup key | `word0[8:0]` | `{bit_off, look9}` |
| Advance method | 0..9-bit barrel shift | ROM-driven byte pointer and bit offset advance |
| ROM output | symbol, length | symbol, length, byte advance, next bit offset |
| State complexity | Simpler control state | More explicit pointer/window bookkeeping |
| Main hardware tradeoff | Wider shift muxes | More pointer/window logic |

---

## Pipeline Alignment

Both architectures use the same lane alignment strategy.

Each stage creates a local `{valid, symbol}` bundle, then sends it through `lane_delay_ff` with depth `31 - stage_index`.

```text
stage 0 result  -> 31-cycle delay
stage 1 result  -> 30-cycle delay
stage 2 result  -> 29-cycle delay
...
stage31 result  ->  0-cycle bypass
```

This makes all 32 lane results arrive in the same output cycle, even though the symbol decisions are created at different points in the pipeline.

The helper module [lane_delay_ff.v](barrel_shifter_source_codes/lane_delay_ff.v) is a simple register chain with an optional bypass when `DEPTH == 0`.

---

## Memory Initialization

The decode tables are loaded with `$readmemh`:

- [barrel_shifter_source_codes/huff_dec1_rom_look9.v](barrel_shifter_source_codes/huff_dec1_rom_look9.v) loads `huff_dec1.mem`
- [bit_pointer_source_codes/huff_dec1_rom_off9_norem.v](bit_pointer_source_codes/huff_dec1_rom_off9_norem.v) loads `huff_dec1_off9_norem.mem`

The script [mem_file_generation/gen_huff_dec1_mem.py](mem_file_generation/gen_huff_dec1_mem.py) is used to generate or regenerate memory content for the decoder ROMs.

---

## Simulation

Each architecture has its own testbench under its `simulation/` directory:

- [barrel_shifter_source_codes/simulation/tb_huff_decoder32_local_rwoh.v](barrel_shifter_source_codes/simulation/tb_huff_decoder32_local_rwoh.v)
- [bit_pointer_source_codes/simulation/tb_huff_decoder32_local_win24_offlut_norem_5a.v](bit_pointer_source_codes/simulation/tb_huff_decoder32_local_win24_offlut_norem_5a.v)

The testbenches:

- generate a 100 MHz-style simulation clock with `#5` toggling
- dump VCD waveforms
- compare the DUT output against a built-in golden decoder
- verify the ingress decision outputs

The testbench golden model matches the code table used by both implementations and checks that accepted packets decode into the expected symbol sequence and valid mask.

---

## Stage-by-Stage Decode Story

This is the practical decode sequence for both designs:

1. A packet arrives on `in_bits` with `in_valid = 1`.
2. The low 32 bits are checked.
3. If those 32 bits are all zero, the packet is dropped.
4. Otherwise the packet enters stage 0.
5. Stage 0 performs one lookup and, if valid, emits one symbol.
6. The stage advances the remaining bitstream state.
7. Stage 1 repeats the same process on the advanced state.
8. This continues through stage 31.
9. All stage-local outputs are delayed and aligned onto the final output buses.
10. The final `out_valid` indicates the frame has traversed all 32 stages.

In both architectures, the pipeline does not decode the whole packet in a single combinational block. Instead, each stage is responsible for one decode step plus one state update, which is easier to pipeline and close timing on FPGA targets.

---

## Choosing Between the Two

Use the barrel shifter version if you want:

- the simplest packet-state representation
- direct 96-bit shifting behavior
- a very straightforward stage implementation

Use the bit-pointer version if you want:

- explicit byte/bit tracking
- a sliding-window style datapath
- reduced dependence on wide per-stage shift networks

---

## Repository Layout

```text
README.md
barrel_shifter_source_codes/
	huff_dec_stage_local_rwoh.v
	huff_dec1_rom_look9.v
	huff_dec1.mem
	huff_decoder32_local_rwoh.v
	lane_delay_ff.v
	simulation/
		tb_huff_decoder32_local_rwoh.v
bit_pointer_source_codes/
	huff_dec_stage_local_win24_offlut_norem_5a.v
	huff_dec1_off9_norem.mem
	huff_dec1_rom_off9_norem.v
	huff_decoder32_local_win24_offlut_norem_5a.v
	lane_delay_ff.v
	simulation/
		tb_huff_decoder32_local_win24_offlut_norem_5a.v
mem_file_generation/
	gen_huff_dec1_mem.py
```

---

## Notes

- The repository contains two independent decoder implementations that share the same external behavior.
- Both implementations are designed for 32 symbol slots per packet.
- The code and testbenches are written to keep the ingress decision visible for debugging and verification.

