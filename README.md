# TEXTEDIT

An 8086 text editor for MS-DOS with a true piece-table architecture.

Written in 16-bit x86 assembly for the [agent86](https://github.com/anthropics/agent86) assembler. Produces a standard .COM binary that runs on any DOS-compatible system (MS-DOS, FreeDOS, DOSBox, etc.).

## Features

- Edit text files up to 32 MB
- True piece table — original file is never modified in memory; edits go to a separate add buffer
- Demand-paged file access — only 256 KB of RAM used regardless of file size (8 x 32 KB LRU-cached chunks)
- Checkpoint table for O(1)+O(256) line seeking with lazy rebuild on structural changes
- Fast-path insert — consecutive typing skips cursor resolution entirely, runs in O(1) per keystroke
- Conditional metadata rebuild — only rescans when line count or piece indices actually change
- Direct VRAM rendering (CGA/EGA/VGA 80×25 text mode)
- Word wrap mode (`--wrap` flag)
- CRLF-aware editing (Enter inserts CR+LF, backspace/delete handle CR+LF pairs)

## Usage

```
TEXTEDIT <filename>
TEXTEDIT --wrap <filename>
```

### Keys

| Key | Action |
|-----|--------|
| Printable chars | Insert at cursor |
| Enter | Insert newline (CR+LF) |
| Backspace | Delete character before cursor |
| Delete | Delete character at cursor |
| Arrow keys | Move cursor |
| PgDn/PgUp | Scroll one page (24 lines) |
| Home/End | Jump to start/end of file |
| Ctrl+S | Save |
| Esc | Quit (prompts if unsaved changes) |

## Building

TEXTEDIT is built with **agent86**, a two-pass 8086 assembler and JIT emulator targeting .COM binaries.

```
agent86 TEXTEDIT.ASM
```

This produces `TEXTEDIT.COM` (~17 KB), ready to run on any DOS system.

### Testing with agent86's built-in emulator

```
agent86 TEXTEDIT.ASM --build_run --screen CGA80 --args "myfile.txt"
```

For large files, compile separately then run with a higher instruction limit:

```
agent86 TEXTEDIT.ASM
agent86 TEXTEDIT.COM --run 500000000 --screen CGA80 --args "bigfile.txt"
```

## Architecture

### Piece Table

TEXTEDIT uses a classic piece table with two text sources:

- **Original buffer** — the file's contents on disk, accessed read-only through demand-paged 32 KB chunks
- **Add buffer** — a 64 KB append-only buffer for all inserted text

The piece table (up to 4096 entries) describes the document as a sequence of spans, each referencing a contiguous run of bytes in one of the two sources. Inserts split an existing piece and add a new one; deletes shrink or remove pieces. The original file data is never copied or modified.

### Demand Paging

File data is loaded on demand through an 8-slot LRU cache (256 KB total). When a piece referencing the original file is accessed, the corresponding 32 KB chunk is loaded from disk (or served from cache). This allows editing files far larger than available RAM.

### Checkpoint Table

Every 256 visual lines, a checkpoint records the current piece index and byte offset. `seek_to_line` jumps to the nearest checkpoint in O(1), then walks forward at most 256 lines. A `meta_dirty` flag triggers a lazy full rescan only when structural changes (newline edits, piece splits/removals) invalidate the table.

### Fast-Path Insert

Consecutive typing at the same position extends the current piece in place — no cursor resolution, no piece-table insertion, no checkpoint walk. Only the first character at a new position takes the slow path.

### Memory Map

| Region | Size | Purpose |
|--------|------|---------|
| COM segment | ~17 KB code + ~10 KB BSS | Program, piece table, checkpoint table |
| Cache slots | 8 × 32 KB = 256 KB | Demand-paged file data |
| Add buffer | 64 KB | Inserted text |
| Stack | ~19 KB | Below 32 KB COM boundary |

### Limits

| Parameter | Value |
|-----------|-------|
| Max file size | 32 MB (1024 chunks × 32 KB) |
| Max visual lines | 524,288 (2048 checkpoints × 256 interval) |
| Max pieces | 4096 |
| Add buffer | 64 KB |
| RAM for file data | 256 KB fixed (8 cached chunks) |
| Video mode | 80×25 CGA/EGA/VGA text |

## License

Public domain.
