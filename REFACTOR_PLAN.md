# auto-save.nvim Fork — Refactor Plan

## Goal

Fix the bugs and design flaws that currently force you to write fragile callback hacks in your LazyVim config. After this refactor, your use case (60-second auto-save, no formatting while in Insert mode, no double notifications) should be achievable with **a single boolean option**.

---

## Current User Config (work-around heavy)

```lua
local _fmt_state = {}
return {
  "suimong/auto-save.nvim",
  opts = {
    debounce_delay = 60000,
    callbacks = {
      before_saving = function()
        if vim.fn.mode() == "i" then
          local buf = vim.api.nvim_get_current_buf()
          _fmt_state[buf] = { value = vim.b[buf].autoformat }
          vim.b[buf].autoformat = false
        end
      end,
      after_saving = function()
        local buf = vim.api.nvim_get_current_buf()
        local state = _fmt_state[buf]
        if state then
          _fmt_state[buf] = nil
          vim.b[buf].autoformat = state.value
        end
      end,
    },
  },
}
```

**Problems with the workaround:**
1. `vim.fn.mode()` is checked at **deferred execution time** (after 60 s), not event time — by then the user may have left Insert mode or switched buffers.
2. `vim.api.nvim_get_current_buf()` inside a callback returns the **current** buffer at callback time, which may not be the buffer that triggered the save.
3. Setting `vim.b[buf].autoformat = false` is a race condition — `BufWritePre` may already have read the value before the callback runs.
4. The callback API receives **zero arguments**, so there is no reliable way to know *which* buffer or *which* mode triggered the save.

---

## Target User Config (after refactor)

```lua
return {
  dir = "/home/yjx/Projects/_contrib/auto-save.nvim",
  event = "BufReadPost",
  opts = {
    execution_message   = { cleaning_interval = 200 },
    debounce_delay      = 60000,
    trigger_events      = { "InsertLeave", "TextChanged" },
    condition = function(buf)
      return vim.bo[buf].modifiable and vim.bo[buf].filetype ~= "oil"
    end,
    noautocmd_in_insert = true,   -- ← one line replaces the entire callback hack
  },
}
```

---

## 1. Concrete Bugs

### Bug 1.1 — Duplicate notification block
**File:** `lua/auto-save/init.lua`  
**Lines:** ~85–124  
The `api.nvim_echo` + `fn.timer_start` block is copy-pasted **twice** in `M.save()`. This causes the exact symptom you observed: two identical auto-save notifications with the same timestamp.

**Fix:** Delete the second copy.

### Bug 1.2 — `write_all_buffers` ignores the `buf` argument
**File:** `lua/auto-save/init.lua`  
When `opts.write_all_buffers = true`, the code runs `cmd("silent! wall")` which saves **all** buffers, yet the `buf` parameter (and any buffer-specific logic) is completely discarded. The `callback("after_saving")` still fires once, which is misleading.

**Fix:** Either remove `write_all_buffers` (it contradicts the plugin’s per-buffer design) or make `M.save` loop over modified buffers and invoke the save logic per buffer.

### Bug 1.3 — `vim.api.nvim_buf_get_option` is deprecated (Neovim ≥ 0.10)
**File:** `lua/auto-save/init.lua`, line ~69  
`api.nvim_buf_get_option(buf, "modified")` triggers a deprecation warning on recent Neovim.

**Fix:** Replace with `vim.bo[buf].modified` or `api.nvim_get_option_value("modified", { buf = buf })`.

---

## 2. Design Flaws

### Flaw 2.1 — "Debounce" is actually a throttle
**File:** `lua/auto-save/init.lua`, function `debounce()`  
Current behaviour:

```
t=0   TextChanged → queue timer for t=60s, mark buffer as queued
t=10  TextChanged → ignored (buffer still queued)
t=60  Timer fires → save happens
```

This is a **throttle** (first event wins, subsequent events are dropped). A true **debounce** resets the timer on every event, so the save happens *after the user stops typing*:

```
t=0   TextChanged → start 60 s timer
t=10  TextChanged → reset timer to t=70
t=20  TextChanged → reset timer to t=80
t=80  Timer fires → save happens
```

With a 60-second *throttle*, the save can fire while the user is still actively typing, which increases the chance that it hits Insert mode unexpectedly.

**Fix:** Rewrite `debounce` using `vim.uv.new_timer()` (or `vim.loop.new_timer()`). Store timers in a `timers[buf]` table; on each event, `stop()` / `close()` the existing timer and start a new one. Use `vim.schedule_wrap()` to call `M.save` from the timer callback.

### Flaw 2.2 — Callbacks receive no context
**File:** `lua/auto-save/utils/data.lua`, function `do_callback()`  
Current signature: `do_callback(name)` → calls `cnf.callbacks[name]()` with **zero** arguments.

Because of Flaw 2.1, the callback executes asynchronously. By the time it runs, `vim.fn.mode()` and `vim.api.nvim_get_current_buf()` may reflect a completely different state than when the triggering event fired.

**Fix:** Change `do_callback` to forward a context table:

```lua
function M.do_callback(callback_name, ctx)
  if type(cnf.callbacks[callback_name]) == "function" then
    cnf.callbacks[callback_name](ctx)
  end
end
```

`ctx` should contain at minimum:
- `ctx.buf`   — the buffer that triggered the save
- `ctx.mode`  — the mode at event time (captured when the autocmd fires)
- `ctx.event` — the triggering event name (e.g. `"TextChanged"`, `"InsertLeave"`)

Then the user’s workaround could at least be written correctly:

```lua
callbacks = {
  before_saving = function(ctx)
    if ctx.mode:match("^[iR]") then
      vim.b[ctx.buf].autoformat = false
    end
  end,
  after_saving = function(ctx)
    -- restore logic...
  end,
}
```

### Flaw 2.3 — No built-in way to suppress autocmds during Insert-mode saves
**Root cause:** The only hook that runs *inside* the actual write is `before_saving`, which runs **before** `nvim_buf_call(... cmd("silent! write"))`. Even if you set `vim.b.autoformat = false` there, other plugins (editorconfig trailing-whitespace trim, conform.nvim, etc.) may have already cached the state or use different variables.

The only robust way to guarantee *no* side effects during a background Insert-mode save is to run the write itself with `:noautocmd` (or temporarily set `eventignore = "BufWritePre"`).

**Fix:** Add a new option:

```lua
opts = {
  noautocmd_in_insert = false,  -- default: off (backwards compatible)
}
```

When `true`, `M.save` detects Insert/Replace mode and runs:

```lua
api.nvim_buf_call(buf, function()
  cmd("noautocmd silent! write")
end)
```

This skips **all** `BufWritePre`/`BufWritePost` autocmds, which is exactly what you want for a mid-typing background save. It also neatly sidesteps the formatter problem without any state-management hacks.

*Alternative API (more flexible):*  
Instead of a boolean, accept an `insert_mode_save` callback:

```lua
opts = {
  insert_mode_save = function(buf)
    vim.api.nvim_buf_call(buf, function()
      vim.cmd("noautocmd silent! write")
    end)
  end,
}
```

If `insert_mode_save` is nil, fall back to the normal path. This gives power users full control while keeping the common case trivial.

### Flaw 2.4 — `M.on()` is not idempotent
**File:** `lua/auto-save/init.lua`, function `M.on()`  
Each call to `M.on()` creates fresh autocmds in the `"AutoSave"` group. If `M.setup()` is called twice (e.g. by lazy.nvim reloading), duplicate autocmds are registered. `M.off()` clears the group, but there is no guard preventing double `M.on()`.

**Fix:** Track an internal flag (e.g. `M._autocmd_ids`) or check `autosave_running` at the top of `M.on()` and bail out early if already running.

### Flaw 2.5 — Global `g.auto_save_abort` is not per-buffer
**File:** `lua/auto-save/init.lua`, `perform_save()`  
`g.auto_save_abort` is a single global boolean. If two buffers trigger saves concurrently, one buffer’s `before_asserting_save` callback could set `abort = true` and accidentally cancel the other buffer’s save.

**Fix:** Make it per-buffer: `vim.b[buf].auto_save_abort` or pass an abort signal through the context object.

---

## 3. Recommended Implementation Order

| Step | Scope | Change | Backwards Compatible? |
|------|-------|--------|----------------------|
| 1 | `init.lua` | Remove duplicated `api.nvim_echo` block | Yes |
| 2 | `init.lua` | Replace `nvim_buf_get_option` with `vim.bo[buf].modified` | Yes |
| 3 | `init.lua` | Rewrite `debounce()` to use `vim.uv.new_timer()` (true debounce) | Behaviour change, but fixes the core timing issue |
| 4 | `init.lua` + `utils/data.lua` | Pass `ctx = { buf, mode, event }` to all callbacks | Yes (callbacks currently receive 0 args; Lua ignores extra args) |
| 5 | `init.lua` | Capture `mode` and `event` inside the autocmd callback and forward through `perform_save` → `save_func` → `M.save` | Yes |
| 6 | `init.lua` | Add `opts.insert_mode_save` (or `opts.noautocmd_in_insert`) | Yes (nil = old behaviour) |
| 7 | `init.lua` | Guard `M.on()` against double registration | Yes |
| 8 | `init.lua` | Replace global `g.auto_save_abort` with per-buffer flag | Minor behaviour change, but fixes a race |

---

## 4. Sketch of the New `M.save`

```lua
function M.save(ctx)
  ctx = ctx or {}
  local buf = ctx.buf or api.nvim_get_current_buf()

  do_callback("before_asserting_save", ctx)

  if cnf.opts.condition(buf) == false then
    return
  end

  if not vim.bo[buf].modified then
    return
  end

  do_callback("before_saving", ctx)

  if vim.b[buf].auto_save_abort then
    return
  end

  -- Insert-mode path
  local in_insert = ctx.mode and ctx.mode:match("^[iR]")
  if in_insert and cnf.opts.insert_mode_save then
    cnf.opts.insert_mode_save(buf)
  elseif cnf.opts.write_all_buffers then
    cmd("silent! wall")
  else
    api.nvim_buf_call(buf, function()
      cmd("silent! write")
    end)
  end

  do_callback("after_saving", ctx)

  -- single notification block (Bug 1.1 fixed)
  local msg = type(cnf.opts.execution_message.message) == "function"
      and cnf.opts.execution_message.message()
      or cnf.opts.execution_message.message

  api.nvim_echo({ { msg, AUTO_SAVE_COLOR } }, true, {})

  if cnf.opts.execution_message.cleaning_interval > 0 then
    fn.timer_start(cnf.opts.execution_message.cleaning_interval, function()
      cmd([[echon '']])
    end)
  end
end
```

---

## 5. Acceptance Criteria

After the refactor, open a Markdown file, enter Insert mode, type a line ending with a space, wait 60 seconds, and observe:

1. **Exactly one** `AutoSave: saved at …` message appears (not two).
2. The trailing space on the current line **is preserved** (no formatter trimmed it).
3. The cursor **remains in Insert mode** (no mode switch, no cursor jump).
4. Your `lua/plugins/auto-save.lua` contains **zero** callback hacks — just `insert_mode_save` or `noautocmd_in_insert`.

---

## 6. Nice-to-Haves (post-refactor)

- **Add a `ASToggle` command** that actually works correctly after the idempotency fix.
- **Add a minimal test suite** using `busted` or `mini.test` so regressions are caught.
- **Expose `M.debounced_save(buf)`** as a public API so users can manually trigger a debounced save from their own mappings.
