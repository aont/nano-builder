# Policy for Fixing Japanese Input Issues in nano

# Proposed Approach

## Recommended Conclusion

For this repository, the following two-stage approach is the most practical.

1. **Short term: fix nano's input wrapper**

   * Do not unconditionally treat `KEY_CODE_YES` as a special key.
   * Treat it as a special key only if ncurses recognizes it as an actual key.
   * Otherwise, interpret it as a Unicode character originating from the Windows console and convert it to UTF-8.
   * Handle UTF-16 surrogate pairs and invalid Unicode code points.
   * Ensure that nano initializes its runtime locale as UTF-8.

2. **Long term: fix ncurses itself**

   * Modify the Windows console driver so that Unicode code points are not passed directly into the byte-oriented input path.
   * However, maintaining a custom ncurses build solely for this repository would impose significant maintenance overhead, so a nano-side workaround is the more practical solution.

---

# 1. Problems with the Current Patch

The current patch performs the following logic:

```c
status = wget_wch(frame, &wide_input);

if (status == ERR)
    return ERR;

if (status == KEY_CODE_YES)
    return (int)wide_input;
```

As a result, when ncurses mistakenly returns Japanese characters as `KEY_CODE_YES`, UTF-8 conversion is bypassed entirely.

In the Windows wide-character implementation of ncurses, `ReadConsoleInputW()` provides the `UnicodeChar` value, which is stored in the input buffer as an integer.

Internally, ncurses returns `KEY_CODE_YES` whenever the input value is greater than or equal to `KEY_MIN`, which is defined as 257.

Therefore, the primary issue is this line:

```c
if (status == KEY_CODE_YES)
    return (int)wide_input;
```

However, simply removing this branch would cause arrow keys, function keys, Delete, and other special keys to be processed as ordinary characters.

---

# 2. Recommended nano-side Fix

## 2.1 Use `keyname()` to Determine Whether It Is a Real Special Key

Conceptually, the recommended logic is as follows:

```c
status = wget_wch(frame, &wide_input);

if (status == ERR)
    return ERR;

if (status == KEY_CODE_YES) {
    /*
     * If ncurses recognizes this as an actual special key,
     * return it unchanged (KEY_LEFT, KEY_DC, etc.).
     */
    if (keyname((int)wide_input) != NULL)
        return (int)wide_input;

    /*
     * Otherwise treat it as a Win32 UnicodeChar that was
     * incorrectly classified as a KEY_* code.
     */
}

/* Process wide_input as a Unicode character. */
return queue_utf8_scalar(wide_input);
```

In ncurses, `keyname()` returns a name for recognized special keys, while returning `NULL` for values that are neither printable characters nor known keys. User-defined extended keys may also have names assigned to them. ([Invisible Island][1])

This produces the following behavior:

| Input        | `wget_wch()` Status | `keyname()`                  | Processing              |
| ------------ | ------------------: | ---------------------------- | ----------------------- |
| Left Arrow   |      `KEY_CODE_YES` | `"KEY_LEFT"`                 | Return as a special key |
| Delete       |      `KEY_CODE_YES` | `"KEY_DC"`                   | Return as a special key |
| F1           |      `KEY_CODE_YES` | `"KEY_F(1)"` (or equivalent) | Return as a special key |
| "あ" (U+3042) |      `KEY_CODE_YES` | `NULL`                       | Convert to UTF-8        |
| "日" (U+65E5) |      `KEY_CODE_YES` | `NULL`                       | Convert to UTF-8        |

### Do Not Rely Solely on `KEY_MAX`

A check such as:

```c
if (status == KEY_CODE_YES && wide_input <= KEY_MAX)
    return (int)wide_input;
```

may appear to work for Japanese input.

However, ncurses can dynamically assign extended key codes beyond `KEY_MAX`. ([Invisible Island][1])

Therefore, checking `keyname()` is more appropriate than comparing against `KEY_MAX`.

---

## 2.2 Separate UTF-8 Encoding into Its Own Function

The current implementation performs UTF-8 encoding directly inside `wgetch_as_utf8()`. Separating responsibilities makes the code easier to verify and test.

```c
static bool is_unicode_scalar(uint32_t value);
static int encode_utf8(uint32_t value, unsigned char output[4]);
static int queue_utf8_scalar(uint32_t value);
static bool is_curses_keycode(wint_t value);
static int wgetch_as_utf8(WINDOW *frame);
```

### Validate Unicode Scalar Values

The current implementation blindly encodes even invalid values such as:

* UTF-16 surrogate range (`U+D800`–`U+DFFF`)
* Values above the Unicode maximum
* Invalid surrogate combinations

At minimum, validation should look like this:

```c
static bool is_unicode_scalar(uint32_t value)
{
    return value <= 0x10FFFF &&
           !(value >= 0xD800 && value <= 0xDFFF);
}
```

When invalid input is encountered, choose one consistent policy:

* Return `ERR` and discard the input.
* Replace it with `U+FFFD`.
* Log the event in debug builds.

For terminal input, discarding invalid input is generally safer than inserting unintended characters.

---

## 2.3 Handle UTF-16 Surrogate Pairs

Since Windows `WCHAR` is 16 bits wide, emoji and some supplementary ideographs may arrive as UTF-16 surrogate pairs.

Example:

```text
U+1F600 😀
    ↓ UTF-16
D83D DE00
    ↓ UTF-8
F0 9F 98 80
```

The current implementation encodes each surrogate independently, producing invalid UTF-8.

The implementation should maintain state similar to:

```c
static wint_t pending_wide_input;
static int pending_wide_status;
static bool has_pending_wide_input;
```

Processing would conceptually follow this pattern:

```c
if (is_high_surrogate(wide_input)) {
    read_next_wide_input();

    if (is_low_surrogate(next_input))
        scalar = combine_surrogate_pair(wide_input, next_input);
    else {
        preserve_next_input_for_later();
        return ERR;
    }
} else if (is_low_surrogate(wide_input)) {
    return ERR;
} else {
    scalar = wide_input;
}
```

This is not essential for ordinary Japanese text, since Hiragana, Katakana, and most commonly used Kanji are within the BMP. However, proper UTF-8 support should include surrogate-pair handling.

---

# 3. Limitations of the `keyname()` Approach

Even with this nano-side fix, some ambiguities cannot be completely resolved.

Within ncurses, Unicode characters and special keys share the same integer namespace. For example, `U+0101` has the numeric value 257, which is equal to `KEY_MIN`.

At that point, nano cannot distinguish between:

```text
Unicode character U+0101
Special key code 257
```

Additionally, `wget_wch()` assumes that `_nc_wgetch()` supplies either a byte stream or `KEY_*` codes. Once `KEY_CODE_YES` is returned, it never enters the multibyte decoding path.

Therefore, the nano-side fix should be viewed as follows:

* Practically effective for Japanese, Korean, Chinese, and similar languages.
* Preserves arrow keys and function keys.
* Protects many extended keys via `keyname()`.
* Cannot mathematically distinguish every Unicode code point from every possible key code.

---

# 4. Correct Long-Term Fix

## Fix the Windows Console Driver in ncurses

The root cause belongs in the ncurses Windows console driver, around `_nc_console_read()`.

Conceptually, the current implementation does:

```c
*buf = (int)inp_rec.Event.KeyEvent.uChar.UnicodeChar;
```

Later, `_nc_wgetch()` interprets any value greater than or equal to `KEY_MIN` as a special key.

Instead, the logic should distinguish ordinary characters from special keys explicitly:

```text
KEY_EVENT
 ├─ UnicodeChar == 0
 │    └─ Convert VirtualKeyCode to KEY_LEFT, etc.
 │
 └─ UnicodeChar != 0
      ├─ Combine UTF-16 surrogate pairs
      ├─ Convert to the current LC_CTYPE multibyte encoding
      └─ Feed the resulting bytes into the ncurses FIFO
```

This allows `_nc_wgetch()` to return UTF-8 bytes one at a time, enabling `wget_wch()` to decode them exactly as originally designed. The `wget_wch()` source comments also state that the lower layer is expected to supply a stream of single-byte characters plus `KEY_*` codes.

## If This Repository Ships Its Own ncurses

The current package dynamically depends on the MSYS2 ncurses library.

The current MSYS2 ncurses package is version 6.6, built with wide-character and terminal-driver support enabled. The published patch set does not appear to include a fix for this Windows Unicode input issue.

Possible approaches include:

| Approach                                  | Drawback                                                                 |
| ----------------------------------------- | ------------------------------------------------------------------------ |
| Replace the system ncurses package        | Conflicts with official MSYS2 packages                                   |
| Statically link ncurses                   | Increased build, licensing, and security maintenance burden              |
| Bundle a renamed ncurses DLL              | Requires managing configure scripts, import libraries, and DLL discovery |
| Submit a fix to MSYS2 or upstream ncurses | May take considerable time before adoption                               |

Therefore, **the most practical approach for this repository is to implement the nano-side workaround while separately reporting the root issue upstream.**

---

# 5. UTF-8 Locale Initialization Is Also Required

The current build uses `--enable-utf8`.

However, at runtime nano performs this strict check:

```c
if (setlocale(LC_ALL, "") &&
        strcmp(nl_langinfo(CODESET), "UTF-8") == 0)
    using_utf8 = TRUE;
```

In other words, compile-time `--enable-utf8` alone is insufficient. Runtime `CODESET` must literally equal `"UTF-8"` before UTF-8 support is enabled.

Even if the input wrapper produces UTF-8, Japanese text will still be processed as individual bytes when `using_utf8 == FALSE`.

## Recommended Initialization

On Windows UCRT64 only, the initialization should conceptually look like:

```c
setlocale(LC_ALL, "");

#if defined _WIN32 && !defined __CYGWIN__ && !defined __MSYS__
if (!codeset_is_utf8(nl_langinfo(CODESET))) {
    if (setlocale(LC_CTYPE, ".UTF8") == NULL) {
        /* Record that UTF-8 mode could not be enabled. */
    }
}
#endif

if (codeset_is_utf8(nl_langinfo(CODESET)))
    using_utf8 = TRUE;
```

On the UCRT, Windows 10 version 1803 and later support UTF-8 code pages by specifying `.UTF8` or `.UTF-8` with `setlocale()`. ([Microsoft Learn][2])

### Prefer `LC_CTYPE` over Resetting `LC_ALL`

The primary goal is to enable UTF-8 character encoding and multibyte processing.

Therefore, changing only `LC_CTYPE` is preferable to resetting `LC_ALL`, since it avoids altering language settings, date formats, and localization behavior.

The UTF-8 detection logic should also accept common variants such as:

```text
UTF-8
UTF8
utf-8
utf8
```

---

# 6. Recommended Patch Structure

Split `nano-9.2-ucrt64.patch` into the following logical changes.

## Patch A: Locale Initialization

Target:

```text
src/nano.c
```

Contents:

* Set `LC_CTYPE` to `.UTF8` on Windows UCRT64.
* Normalize UTF-8 codeset detection.
* Define behavior when UTF-8 initialization fails.

## Patch B: Reclassify Special Keys and Unicode Characters

Target:

```text
src/winio.c
```

Contents:

* Do not unconditionally trust `KEY_CODE_YES`.
* Treat values with a non-`NULL` `keyname()` as special keys.
* Treat values without a key name as Unicode candidates.
* Restrict this behavior to native Windows builds.

## Patch C: Harden the UTF-8 Encoder

Target:

```text
src/winio.c
```

Contents:

* Validate Unicode scalar values.
* Combine surrogate pairs.
* Reject invalid surrogate sequences.
* Improve UTF-8 output buffer management.
* Reset pending state on errors.

---

# 7. Example Implementation

The following illustrates the intended structure rather than a complete patch.

```c
static bool is_known_curses_key(wint_t value)
{
    if (value < KEY_MIN)
        return false;

    return keyname((int)value) != NULL;
}

static bool is_high_surrogate(uint32_t value)
{
    return value >= 0xD800 && value <= 0xDBFF;
}

static bool is_low_surrogate(uint32_t value)
{
    return value >= 0xDC00 && value <= 0xDFFF;
}

static bool is_unicode_scalar(uint32_t value)
{
    return value <= 0x10FFFF &&
            !is_high_surrogate(value) &&
            !is_low_surrogate(value);
}

static int wgetch_as_utf8(WINDOW *frame)
{
    wint_t wide_input;
    int status;

    if (have_pending_utf8_bytes())
        return take_pending_utf8_byte();

    status = read_next_wide_token(frame, &wide_input);

    if (status == ERR)
        return ERR;

    if (status == KEY_CODE_YES && is_known_curses_key(wide_input))
        return (int)wide_input;

    /*
     * Even when KEY_CODE_YES is returned, values without a known
     * key name are treated as Win32 UnicodeChar input.
     */

    uint32_t scalar;

    if (is_high_surrogate(wide_input)) {
        if (!read_and_combine_low_surrogate(frame, wide_input, &scalar))
            return ERR;
    } else if (!is_unicode_scalar(wide_input)) {
        return ERR;
    } else {
        scalar = (uint32_t)wide_input;
    }

    return encode_and_queue_utf8(scalar);
}
```

A production implementation should also include:

* Preserving lookahead input when it is not a low surrogate.
* Resetting pending state on `ERR`.
* Regression tests for `KEY_MOUSE` and `KEY_RESIZE`.
* Correct behavior during timeout and non-blocking reads.
* Clearing pending state when nano exits or flushes input.

---

# 8. Testing Strategy

The current CI verifies builds, package contents, `nano.exe --version`, dependent DLLs, and independence from the MSYS runtime, but it does not perform interactive input testing.

## 8.1 Unit Tests

Separate the UTF-8 encoder from curses input and verify at least the following cases:

| Input          | Expected Bytes |
| -------------- | -------------- |
| `U+007F`       | `7F`           |
| `U+0080`       | `C2 80`        |
| `U+07FF`       | `DF BF`        |
| `U+0800`       | `E0 A0 80`     |
| `U+3042` (あ)   | `E3 81 82`     |
| `U+65E5` (日)   | `E6 97 A5`     |
| `U+1F600` (😀) | `F0 9F 98 80`  |
| `U+D800`       | Error          |
| `U+DFFF`       | Error          |
| `U+110000`     | Error          |

Also verify that the following special keys pass through unchanged:

```text
KEY_LEFT
KEY_RIGHT
KEY_UP
KEY_DOWN
KEY_HOME
KEY_END
KEY_NPAGE
KEY_PPAGE
KEY_BACKSPACE
KEY_DC
KEY_IC
KEY_BTAB
KEY_MOUSE
KEY_RESIZE
KEY_F(1)–KEY_F(12)
```

## 8.2 Windows Console Input Tests

Create a small test program that records:

```text
status
wide_input
keyname(wide_input)
```

Test with:

```text
あ
日本語
カタカナ
ＡＢＣ
「」、。
髙
😀
```

Expected output example:

```text
Input: あ
status: KEY_CODE_YES
wide_input: 0x3042
keyname: NULL
Final UTF-8: E3 81 82
```

## 8.3 Interactive nano Testing

Verify at least the following:

1. IME can commit "日本語入力テスト".
2. Left and right arrow keys work correctly.
3. Backspace deletes one Japanese character.
4. Delete removes the next character.
5. The cursor never lands in the middle of a full-width character.
6. Saved files are UTF-8.
7. Existing UTF-8 files can be edited without corruption.
8. Japanese input works in the Search, Save As, and Go To Line prompts.
9. F1, Page Up, Page Down, Home, and End continue working.
10. No pending UTF-8 bytes remain after switching from IME input back to ASCII.

Testing should cover at least:

* Windows Terminal + PowerShell
* Windows Terminal + cmd.exe
* Legacy conhost
* UCRT64 shell
* `TERM=win32con`
* Standard VT/ConPTY paths

---

# 9. Changes That Should Not Be Adopted

## Treat Every `KEY_CODE_YES` as Unicode

```c
if (status == KEY_CODE_YES)
    encode_utf8(wide_input);
```

This breaks arrow keys and function keys.

## Treat Every Value Above 256 as Unicode

```c
if (wide_input > 255)
    encode_utf8(wide_input);
```

This is incorrect because ncurses special keys begin at 257.

## Treat Only Values Below `KEY_MAX` as Special Keys

This may work for Japanese input but can break dynamically allocated extended key codes. Using `keyname()` is preferable.

## Rely Only on `chcp 65001`

The conflict occurs after `ReadConsoleInputW()` has already produced a Unicode character and ncurses classifies it numerically. Changing the console code page alone cannot fix this.

## Force `TERM` to Another Value

Redirecting users to a VT path is merely a workaround and may introduce side effects involving key definitions, mouse handling, colors, and resize behavior. It is not an appropriate package-level fix.

## Replace ncurses with PDCurses

Replacing the
