# Spec: direct image transmission (`t=d`) for remote sessions

Status: proposal, not yet implemented.

## Problem

Inline images never appear when Neovim runs over SSH, even in a terminal that
fully supports the Kitty graphics protocol. This affects every image type the
plugin renders — local images, web images, video frames, Mermaid, and PlantUML.

The cause is the transmission medium. Every transmit site sends a **filesystem
path**, base64-encoded, and asks the terminal to open it:

- `image.lua:1184` — `transmit_image` (`t=t` for temp files, else `t=f`)
- `image.lua:1327` — animated frame transmit (`t=f`)
- `image.lua:1357` — `transmit_image_async` (`t=t`/`t=f`)
- `image.lua:1443` — batched animation frames (`t=f`)
- `image.lua:1639` — `put_image`'s Ghostty `a=T` re-transmit (`t=f`)

`t=f` and `t=t` both mean "a path on the terminal's own filesystem". Locally
that path resolves. Over SSH the image is written to the **remote** machine's
disk (`~/.cache/nvim/md-render/...`) while the terminal runs on the **local**
machine, so the path does not exist there and nothing renders. No error is
surfaced: the plugin has already cleared the placeholder text, so the user sees
a blank gap.

Confirmed on a live session: Neovim on the remote host produced valid PNGs in
its cache and emitted well-formed escape sequences; the local Ghostty had no
such files to open.

## Approach

Add `t=d` (direct) transmission: base64 the image **bytes** and send them inline
so they travel over the SSH stream, chunked per the protocol. Select the medium
at transmit time — keep `t=f`/`t=t` locally (cheap, no payload in the write
path) and use `t=d` when the session is remote.

### Protocol requirements

From the [kitty graphics protocol spec](https://sw.kovidgoyal.net/kitty/graphics-protocol/):

- Base64 payload is chunked into pieces **no larger than 4096 bytes**.
- Every chunk except the last must have a size that is **a multiple of 4**.
- `m=1` on all chunks except the final one; `m=0` on the last.
- The **first** chunk carries the full control data (`a=t,f=100,t=d,i=<id>,q=2`)
  plus `m=1`. **Continuation chunks must carry only `m` and optionally `q`** —
  not `i=`, not `f=`. (Animation frames are the exception: those must also
  repeat `a=f`.)
- All chunks for one image must be sent before any other graphics escape code.
- The terminal displays nothing until the full sequence arrives and validates.

### Detecting a remote session

```lua
--- True when the terminal is not on this machine, so filesystem-path
--- transmission (t=f/t=t) cannot work and the bytes must be sent inline.
local function is_remote_session()
  if vim.g.md_render_force_direct_transmission ~= nil then
    return vim.g.md_render_force_direct_transmission and true or false
  end
  return vim.env.SSH_CONNECTION ~= nil or vim.env.SSH_TTY ~= nil
end
```

`SSH_CONNECTION`/`SSH_TTY` cover the common case. The override exists because
detection cannot be exhaustive (containers, `distrobox`, nested muxers, remote
dev setups that scrub SSH vars), and because a user on a slow link may want to
force the local path back on. Memoize like the other capability checks, and
reset it in `M.reset_cache()`.

### Chunked writer

```lua
local DIRECT_CHUNK_SIZE = 4096 -- protocol max; multiple of 4

--- Transmit base64 image data inline, chunked per the protocol.
--- The first chunk carries the control data; continuation chunks carry
--- only m= (and q=), as the spec requires.
---@param control string  e.g. "a=t,f=100,t=d,i=42,q=2"
---@param b64 string      base64-encoded image bytes
local function write_direct_chunks(control, b64)
  local total = #b64
  if total == 0 then return end
  local pos = 1
  local first = true
  while pos <= total do
    local chunk = b64:sub(pos, pos + DIRECT_CHUNK_SIZE - 1)
    pos = pos + #chunk
    local more = pos <= total and 1 or 0
    if first then
      term_write(string.format("\x1b_G%s,m=%d;%s\x1b\\", control, more, chunk))
      first = false
    else
      term_write(string.format("\x1b_Gm=%d,q=2;%s\x1b\\", more, chunk))
    end
  end
end
```

Chunk size 4096 is already a multiple of 4, so every non-final chunk satisfies
the alignment rule automatically; only the final chunk may be unaligned, which
is permitted.

Because `term_write` participates in the existing batching layer
(`begin_batch`/`flush_batch`, `image.lua:26-68`), all chunks for one image land
contiguously in the same batch — which is exactly what "finish sending all
chunks before any other graphics escape code" requires. Do **not** interleave
placements between chunks.

### Reading the bytes

```lua
---@param path string
---@return string? b64
local function read_file_b64(path)
  local f = io.open(path, "rb")
  if not f then return nil end
  local data = f:read "*a"
  f:close()
  if not data or data == "" then return nil end
  return vim.base64.encode(data)
end
```

### Wiring into the transmit sites

Each site currently builds one `string.format` with `t=%s` and a base64 path.
The change is the same shape at all five:

```lua
if is_remote_session() then
  local b64 = read_file_b64(png_path)
  if not b64 then return nil end
  write_direct_chunks(string.format("a=t,f=100,t=d,i=%d,q=2", id), b64)
else
  -- existing t=f / t=t path, unchanged
end
```

Two sites need extra care:

- **`put_image`'s Ghostty branch (`image.lua:1639`)** re-transmits with `a=T`
  on every placement specifically because Ghostty re-reads the file. That trick
  is meaningless remotely — there is no file to re-read, and re-sending the full
  payload on each redraw would be very expensive. When remote, skip the `a=T`
  workaround and use the normal `a=t` (once) + `a=p` (per placement) path. This
  needs verifying against Ghostty over SSH: if `a=p` genuinely misbehaves there,
  fall back to re-sending, but gate it behind a size threshold.
- **Animation frames (`image.lua:1327`, `1443`)** transmit many images in a
  loop. Chunked direct transmission multiplies the bytes on the wire by the
  frame count. Recommend disabling animated GIF/video frames entirely when
  remote (render the first frame as a static image) rather than shipping tens of
  megabytes over SSH. Make that a documented behavior, not a silent one.

### Temp-file cleanup

`_temp_image_paths` (`image.lua:1164`) exists to keep temp files alive because
the terminal re-reads them. Direct transmission copies the bytes into the escape
stream, so the file is no longer needed after the chunks are written: when
remote, delete the temp file immediately after transmit instead of deferring to
`delete_image`/`delete_all`. Keep the deferred path for local sessions.

## Cost

Base64 inflates by 4/3. A typical PlantUML render (~8KB) becomes ~11KB, about 3
chunks — negligible. A large screenshot (2MB) becomes ~2.7MB across ~680 chunks,
which is noticeable on a slow link and is the main reason animation should be
disabled remotely.

Consider a size ceiling (`vim.g.md_render_max_remote_image_bytes`, default
perhaps 2MB) above which the image is skipped with a visible note rather than
stalling the UI. Silent truncation would be worse than not rendering.

## Testing

Escape-sequence assertions, following `tests/plantuml_kitty_test.lua`, which
already captures output by monkey-patching `vim.api.nvim_ui_send`:

- Small payload (< 4096 b64 bytes) emits exactly one chunk, with full control
  data and `m=0`.
- Multi-chunk payload: first chunk has the control data and `m=1`; middle chunks
  carry only `m=1`/`q=2`; final chunk has `m=0`; no continuation chunk repeats
  `i=` or `f=`.
- Every non-final chunk's payload length is ≤ 4096 and a multiple of 4.
- Concatenating all chunk payloads and base64-decoding reproduces the original
  file byte-for-byte.
- `SSH_CONNECTION` set selects `t=d`; unset selects `t=f`/`t=t`;
  `vim.g.md_render_force_direct_transmission` overrides both directions.
- No other graphics escape code appears between the first and last chunk.

Set/restore `SSH_CONNECTION`/`SSH_TTY` in setup and teardown, and call
`image.reset_cache()` after changing them so the memoized detection re-runs —
the same env-leak trap that made `image_test.lua` terminal-dependent.

Mutation-test the result: transposing `m=1`/`m=0`, repeating `i=` on a
continuation chunk, or using a chunk size that is not a multiple of 4 must all
fail the suite.

Real-terminal verification cannot be automated here; confirm manually over SSH
into Ghostty, Kitty, and WezTerm before merging.

## Out of scope

- Fixing terminal detection when `TERM_PROGRAM`/`GHOSTTY_RESOURCES_DIR` are
  absent and only `TERM=xterm-ghostty` identifies the terminal. That is a
  separate pre-existing bug in `supports_kitty()` (`image.lua:563`) that also
  blocks images on SSH sessions, and it should land on its own branch. Both are
  needed for images to work remotely.
- The `a=T` Ghostty workaround has no test coverage today (stubbing
  `is_ghostty()` to `false` breaks no tests). Worth adding alongside this, since
  this change alters that branch.
