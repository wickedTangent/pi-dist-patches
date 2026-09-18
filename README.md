# Pi Dist Patches

Manual patches for `@earendil-works/pi-coding-agent` dist files.

These changes live in `node_modules/dist/` so they get wiped on every `pi update`. This repo documents them so we can re-apply quickly.

---

## Tokens/Sec Footer

Shows streaming throughput (e.g. `3.2t/s`) as first item in footer stats line. Matches llama.cpp metric: `total_tokens / generation_time`.

### File: `dist/modes/interactive/components/footer.js`

#### 1. Add field and method to FooterComponent class

In the FooterComponent class:

```js
// In constructor:
this._tokensPerSec = undefined;

// Add method after setAutoCompactEnabled():
setTokensPerSec(value) {
    this._tokensPerSec = value;
}
```

#### 2. Insert t/s as first item in statsParts

Find the `statsParts` block near the top of `render()`:

```js
const statsParts = [];
// Tokens/sec (user customization)
if (this._tokensPerSec !== undefined && this._tokensPerSec > 0) {
    statsParts.push(`${this._tokensPerSec.toFixed(1)}t/s`);
}
if (totalInput)
    statsParts.push(`↑${formatTokens(totalInput)}`);
```

### File: `dist/modes/interactive/interactive-mode.js`

#### 1. Add tracking field

In the InteractiveMode class, add:

```js
// First timestamp for tokens/sec (set on first message_update)
_streamStartTime = 0;
```

#### 2. Record start time on first message_update

In the `case "message_update":` block, before the toolCall loop:

```js
if (this._streamStartTime === 0) {
    this._streamStartTime = Date.now();
}
```

#### 3. Calculate and display on message_end

In the `case "message_end":` block, before `footer.invalidate()`:

```js
// Calculate tokens/sec: total_tokens / generation_window
if (this._streamStartTime > 0 && this.streamingMessage?.usage?.output > 0) {
    const genMs = Date.now() - this._streamStartTime;
    if (genMs > 0) {
        const tokensPerSec = (this.streamingMessage.usage.output / genMs) * 1000;
        if (tokensPerSec > 0) {
            this.footer.setTokensPerSec(tokensPerSec);
        }
    }
}
this._streamStartTime = 0;
```

#### 4. Force redraw after message_end

After `footer.invalidate()`:

```js
this.ui.requestRender(true);
```

---

## How to Re-apply After `pi update`

After `pi update`, apply these edits to the `dist/` files. The surrounding code is unique enough to identify exact insertion points.

---

## Notes

- Displays as first item in footer: `43.5t/s ↑2.4k ↓1.1k`
- Metric: `outputTokens / (message_end_time - first_message_update_time) * 1000`
- Matches llama.cpp slot timing
- Resets when a message completes (`message_end`)
