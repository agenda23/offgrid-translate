# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single self-contained static HTML file (`index.html`) — an on-device English↔Japanese translation tool built on Chrome's built-in Translator API (`window.Translator`, Chrome 138+). No build system, no dependencies, no package.json. All HTML, CSS, and JS live in this one file.

Translation happens entirely in the browser via the on-device model; the app sends nothing to a server.

## Running / testing

There is no build or test tooling. To work on this, just open `index.html` directly in Chrome 138+ (desktop only — the Translator API is unavailable on mobile Chrome and other browsers). Reload the tab after editing to see changes.

There's no bundler, linter, or package manager configured — don't introduce one unless asked.

## Architecture

Everything is in `index.html`, organized top to bottom as: `<style>` → markup → `<script>`.

- **Translator lifecycle** (`getTranslator`): translator instances are cached per direction in `translatorCache` keyed by `"source-target"`. Creating one checks `Translator.availability()` first and surfaces a `monitor`-driven download-progress UI (`progressBar`/`progressFill`) when the on-device model needs downloading.
- **Chunking** (`chunkWithOffsets`): input text is split into paragraph-sized chunks (falling back to sentence-grouping for long paragraphs, capped at `MAX_LEN = 800` chars) because the Translator API works on bounded text. Each chunk retains its `{start, end}` offset into the original input string — this is what makes bidirectional highlighting possible.
- **Streaming translation** (`translateAll`): chunks are translated sequentially (not in parallel), with output rendered incrementally after each chunk completes so long inputs show progress rather than blocking. Uses a simple abort-token pattern (`currentAbort`/`myAbort.aborted`) to cancel stale in-flight translations when input changes mid-translation — check this pattern before changing the translation flow.
- **Bidirectional selection highlighting**: `currentChunks` (the chunk list from the last completed translation) links input and output. Selecting text in the input textarea highlights overlapping output `<span class="chunk">` elements (`handleInputSelection`); selecting output text sets the corresponding textarea selection range (`handleOutputSelection`). Both are wired through a single `document`-level `selectionchange` listener that branches on `document.activeElement`.
- **Auto-translate**: input events debounce (`DEBOUNCE_MS = 700`) into `translateAll`, toggleable via the `#autoToggle` checkbox.
- **Language swap** (`swapBtn`): flips `direction.source`/`direction.target` and swaps the input/output text contents in place; does not clear the translator cache (both directions get cached independently).
- **Theming**: CSS custom properties in `:root` with a `prefers-color-scheme: dark` media query and a `[data-theme="dark"]` attribute override for explicit theme selection (though nothing currently sets `data-theme`).
