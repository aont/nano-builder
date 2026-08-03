## Conclusion

The most likely cause is **a mismatch between GNU nano's input processing, which assumes UTF-8 byte sequences, and the Windows version of ncurses, which returns IME-committed characters as UTF-16 Unicode code points**.

For example, the Japanese character "あ" has the Unicode code point `U+3042`, whose integer value is `12354`. With the current stack, the input path is approximately:

```text
Windows IME
  ↓ Commit "あ"
ReadConsoleInputW()
  ↓ UnicodeChar = U+3042
ncurses wgetch()
  ↓ int 12354
nano
  ↓ Greater than 0xFF, so not treated as a normal character
Discarded as an unbound key
```

## Evidence from the Code

### 1. ncurses uses Windows wide-character input

The MSYS2 build of ncurses is compiled with:

```text
--enable-widec
--enable-term-driver
```

In the Windows terminal driver used for wide-character support, the input API and character field are defined as follows:

```c
#define read_keycode ReadConsoleInputW
#define KeyEventChar KeyEvent.uChar.UnicodeChar
```

This means that when the Japanese IME commits a character, ncurses receives it as a UTF-16 Unicode code point rather than as a UTF-8 byte sequence.

Furthermore, the Windows console input handler stores that value directly into an `int` buffer:

```c
*buf = (int) inp_rec.Event.KeyEventChar;
```

As a result, "あ" is passed to nano as `0x3042`, not as the UTF-8 bytes `0xE3 0x81 0x82`.

### 2. nano is designed to read bytes via `wgetch()`

nano's input layer uses the standard `wgetch()` API rather than the wide-character `wget_wch()`.

```c
input = wgetch(frame);
```

The values returned by successive calls are then interpreted as individual UTF-8 bytes and stored in nano's key buffer.

The range accepted as a normal character is explicitly limited to one byte:

```c
if (input < 0x20 || input > 0xFF || meta_key)
    unbound_key(input);
```

Therefore, `U+3042` (`0x3042`) is not recognized as a normal character and is instead treated as an unbound key.

This affects not only Japanese but, in principle, any character with a code point of `U+0100` or higher.

## Why `--enable-utf8` Does Not Fix It

This repository builds nano with:

```bash
--enable-utf8
--enable-nls
```

However, `--enable-utf8` only enables nano to process **UTF-8 multibyte sequences internally**. It does **not** convert Windows UTF-16 character values into UTF-8.

At runtime, nano also enables `using_utf8` only when the locale's codeset name is exactly `"UTF-8"`:

```c
if (setlocale(LC_ALL, "") &&
        strcmp(nl_langinfo(CODESET), "UTF-8") == 0)
    using_utf8 = TRUE;
```

Consequently, if Windows reports the locale using names such as `CP932`, `CP65001`, or `UTF8`, additional issues may arise. This is a secondary risk, however, and is independent of the primary `U+3042 > 0xFF` problem.

## Repository-Specific Situation

The current Windows compatibility patch modifies only:

* Directory rescanning in `browser.c`
* Timestamp handling in `files.c`

No changes have been made to the input handling code in `winio.c` or to the interface between nano and ncurses where character encoding is handled.

The CI validation currently consists primarily of:

```bash
/ucrt64/bin/nano.exe --version
```

IME input, non-ASCII input, and UTF-8 file editing are **not** tested. As a result, this issue would not be detected even if the build and startup succeed.

In addition, the workflow explicitly prohibits linking against `msys-2.0.dll`. That means the build intentionally uses the native Windows console input path rather than receiving UTF-8 byte streams through the MSYS PTY, making this issue much more likely to surface.

## Evaluation of Possible Fixes

### Recommended: Use `wget_wch()` in nano

The most robust solution for native Windows builds is:

1. Read input with `wget_wch()`.
2. Distinguish ordinary characters from special keys based on the return value:

   * `OK`: Unicode character
   * `KEY_CODE_YES`: Function key or other special key
3. Convert the Unicode character to UTF-8.
4. Push the resulting one to four UTF-8 bytes into nano's existing key buffer.

Conceptually:

```c
wint_t value;
int status = wget_wch(frame, &value);

if (status == OK) {
    /* Convert value to UTF-8 and store the resulting 1–4 bytes in key_buffer */
} else if (status == KEY_CODE_YES) {
    /* Pass through the existing special-key handling */
}
```

Simply accepting values greater than `0xFF` as normal characters is insufficient because nano's editing buffer stores UTF-8 byte sequences, while special key codes occupy the same integer namespace.

### Smaller Alternative: Convert Windows Character Values to UTF-8

Another approach is to convert non-special character values returned by the Windows driver into UTF-8 within `read_keys_from()` before passing them to nano's existing byte-oriented processing.

This requires fewer code changes but must correctly distinguish:

* Regular Unicode characters from ncurses special keys
* UTF-16 surrogate pairs
* Combining characters
* Multiple events generated during IME composition

Because `wgetch()` alone does not reliably distinguish ordinary characters from special keys, `wget_wch()` is the safer approach.

### Practical Workaround

If a native UCRT build is not required, using the MSYS runtime version of nano—which receives UTF-8 byte streams through the PTY—is much more consistent with nano's original input model.

Depending on the environment (Windows Terminal, ConPTY, MSYS2, etc.), ncurses may also use the terminfo backend instead of the Win32 console driver, allowing Japanese input to work. However, this behavior depends on the terminal, `TERM`, `LANG`, and the console code page, making it unsuitable as a consistent release configuration.

## How to Isolate the Issue

Compare the same binary under different environments using:

```bash
printf '%s\n' "$TERM"
printf '%s\n' "$LANG" "$LC_ALL"
locale
nano --version
```

On Windows, also check:

```powershell
chcp
$env:TERM
$env:LANG
$env:LC_ALL
```

The following symptoms can help identify the cause:

| Symptom                                          | Likely Cause                                                                                        |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------- |
| ASCII input works, but Japanese input is ignored | Unicode values greater than `0xFF` are being discarded                                              |
| Paste works, but IME input does not              | Problem in the Windows keyboard event path                                                          |
| Characters appear but are garbled                | UTF-8 locale or code-page configuration issue                                                       |
| Works in MSYS2/mintty but not in cmd.exe         | Difference in the ncurses driver path                                                               |
| Japanese input is ignored in every terminal      | Strong indication of a mismatch between nano's byte-oriented input model and wide-character ncurses |

## Assessment

**The evidence strongly supports this explanation.**

The issue is not a build failure or a missing DLL. Instead, it results from an architectural mismatch between the two APIs:

```text
Windows ncurses:
Returns Unicode characters

GNU nano:
Consumes UTF-8 bytes
```

If this issue is to be fixed, the primary target is **nano's Windows wide-character input handling in `src/winio.c`**, not the PKGBUILD. No files have been modified, committed, or pushed.
