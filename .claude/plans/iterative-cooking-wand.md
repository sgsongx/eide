# Fix: JLink loadfile doubled path in HexUploader

## Problem

When using the JLink download button, the loadfile command produces a doubled path:
```
J-Link>loadfile "d:\...\keil\d:\...\keil\build\zephyr\zephyr.hex"
```
Instead of:
```
J-Link>loadfile "d:\...\keil\build\zephyr\zephyr.hex"
```

## Root Cause

In `src/HexUploader.ts`, `parseProgramFiles()` prepends `'.'` to an already-absolute path:

```typescript
// Line 238-240 (JLink uploader)
const hexPath = [
    '.', this.project.getExecutablePathWithoutSuffix() + '.hex'
].join(File.sep);
```

`getExecutablePathWithoutSuffix()` (in `src/EIDEProject.ts:901`) returns an **absolute** path via `getOutputFolder().path` (which calls `ToAbsolutePath()`). The result:
- `['.', 'd:\...\build\zephyr\zephyr.hex'].join('\')` -> `'.\d:\...\build\zephyr\zephyr.hex'`
- `File.isAbsolute('.\d:\...')` returns `false` (Node considers `.\` relative)
- `toAbsolutePath()` prepends project root -> doubled path

## Fix

Remove the unnecessary `'.'` prefix in both locations since `getExecutablePathWithoutSuffix()` already returns an absolute path.

### Files to modify

1. **`src/HexUploader.ts`** — 2 locations:

**Line 238-240 (JLink uploader):**
```typescript
// Before:
const hexPath = [
    '.', this.project.getExecutablePathWithoutSuffix() + '.hex'
].join(File.sep);

// After:
const hexPath = this.project.getExecutablePathWithoutSuffix() + '.hex';
```

**Line 1168 (ProbeRS uploader):**
```typescript
// Before:
const elfPath = ['.', this.project.getExecutablePathWithoutSuffix() + '.elf'].join(File.sep);

// After:
const elfPath = this.project.getExecutablePathWithoutSuffix() + '.elf';
```

## Verification

After the fix, the JLink loadfile command should produce the correct single path. The probe-rs `--path` argument should also be correct.
