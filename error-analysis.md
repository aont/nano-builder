# Investigation into the Cause of the Japanese Input Issue in nano

# Investigation Results

## Conclusion

The primary cause of Japanese input being completely unresponsive is a **bug in which Unicode code points returned by the Windows native console driver are misidentified as ncurses special key codes**.

This issue is highly likely to occur specifically when using the `win32con` input path—that is, when input is obtained through `ReadConsoleInputW()`.

Although the locale detection issue discussed earlier also exists, further analysis shows that there is a more fundamental problem occurring beforehand: **Japanese characters never reach nano as character input in the first place**.

| Factor                                                              | Classification  | Expected Symptom                                        | Confidence                      |
| ------------------------------------------------------------------- | --------------- | ------------------------------------------------------- | ------------------------------- |
| Collision between Unicode code points and ncurses special key codes | Primary cause   | Characters are not inserted even after IME confirmation | High                            |
| nano runtime UTF-8 detection                                        | Secondary cause | Garbled text, cursor movement/deletion inconsistencies  | High, but environment-dependent |
| No interactive input tests in CI                                    | Detection gap   | Build succeeds while the bug remains undetected         | Certain                         |

No modifications were made to the repository.

---

## 1. Exact Execution Path Where the Problem Occurs

This repository builds nano 9.2 for UCRT64 with `--enable-utf8` enabled. It also includes a custom patch that replaces the traditional `wgetch()` call with `wget_wch()` on Windows, intending to convert returned wide characters into UTF-8 byte sequences.

However, when using the Windows native console path, processing proceeds as follows.

### When entering "あ"

```text
IME confirms "あ"
    ↓
Unicode code point U+3042
    ↓
ReadConsoleInputW()
    ↓
KEY_EVENT_RECORD.UnicodeChar = 0x3042
    ↓
ncurses _nc_console_read()
    ↓
Input value 0x3042 is passed as an integer to _nc_wgetch()
```

The Windows console driver in the wide-character version of ncurses uses `ReadConsoleInputW()` and returns the `UnicodeChar` field from `KEY_EVENT_RECORD` directly as the input value. Consequently, the character "あ" enters ncurses as the integer `0x3042` (decimal `12354`).

Meanwhile, ncurses defines the beginning of its special key range as follows:

```text
KEY_CODE_YES = 0400 (decimal 256)
KEY_MIN      = 0401 (decimal 257)
```

Inside `_nc_wgetch()`, any input value greater than or equal to `KEY_MIN` is treated as a special key rather than an ordinary character, causing `KEY_CODE_YES` to be returned.

Therefore,

```text
"あ" = U+3042 = 12354
12354 >= KEY_MIN (257)
```

As a result, **the ordinary Unicode character "あ" is incorrectly classified as a special key.**

---

## 2. The Repository's UTF-8 Conversion Never Executes

Conceptually, the custom wrapper contains logic similar to this:

```text
wget_wch() returned KEY_CODE_YES
    → Treat input as a special key and return it unchanged

Otherwise
    → Convert the wide character to UTF-8
```

Because ncurses incorrectly identifies "あ" as `KEY_CODE_YES`, the wrapper follows the special-key branch.

As a result, the required conversion

```text
U+3042
    ↓ UTF-8 conversion
E3 81 82
```

never takes place.

Instead, the integer value `0x3042` is passed directly to nano. The repository's UTF-8 conversion code itself is capable of correctly converting Japanese BMP characters when execution reaches the ordinary-character branch, but **the status check prevents the conversion logic from ever being reached**.

nano's input processing fundamentally expects either:

* Input bytes roughly in the range 0–255, or
* Known `KEY_*` special key codes defined by ncurses.

The value `0x3042` is neither a UTF-8 byte sequence nor a recognized special key. Consequently, it is not inserted into the text buffer as a character. To the user, this appears as though **nothing happens after confirming Japanese input through the IME**.

### Summary of the Input Path

```text
IME
 │
 │ "あ" U+3042
 ▼
ReadConsoleInputW / UnicodeChar
 │
 │ 0x3042
 ▼
ncurses _nc_wgetch
 │
 │ 0x3042 >= KEY_MIN
 ▼
Incorrectly classified as KEY_CODE_YES
 │
 ▼
Repository wrapper
 │
 │ Enters special-key branch
 │ UTF-8 conversion never executes
 ▼
0x3042 passed to nano
 │
 ▼
Not recognized as a character and therefore not inserted
```

---

## 3. Occurrence Depends on the Terminal Driver

This issue primarily occurs when ncurses uses the **native Windows console driver**.

Broadly speaking, ncurses supports two input paths.

### Native Console Path

```text
TERM=#win32con
    ↓
ReadConsoleInputW
    ↓
UnicodeChar received as an integer
```

In this path, the collision between Unicode code points and the `KEY_*` range occurs as described above.

### VT / Terminal Byte Stream Path

```text
Windows Terminal / ConPTY / ms-terminal
    ↓
UTF-8 byte stream
    ↓
ncurses decodes multibyte characters
```

In this path, Japanese characters may already arrive as UTF-8 byte sequences such as `E3 81 82`, avoiding the same collision.

ncurses explicitly selects the native Windows console driver when `TERM=#win32con` is specified. Depending on the Windows version, VT support, and whether `TERM` is defined, it switches between the standard terminfo path and the console driver path. Therefore, behavior may differ among Windows Terminal, the traditional conhost, PowerShell, cmd.exe, MSYS2 terminals, and similar environments.

This can lead to differences such as:

* IME input fails while paste succeeds.
* Windows Terminal works but the legacy console does not.
* Failure occurs only when `TERM=#win32con` is used.
* Characters can be entered, but deletion or cursor movement behaves incorrectly.

---

## 4. Secondary Factor: nano Runtime UTF-8 Detection

Specifying `--enable-utf8` does **not** force nano to operate in UTF-8 mode unconditionally.

At startup, nano roughly checks the following:

```text
setlocale(LC_ALL, "") succeeds
and
nl_langinfo(CODESET) is exactly "UTF-8"
    → using_utf8 = TRUE
```

Although this repository is compiled with `--enable-utf8`, the runtime value of `using_utf8` still depends on locale detection.

In UCRT, `setlocale(LC_ALL, "")` uses the user's default locale or ANSI code page depending on the environment. Locales explicitly specifying `.UTF8` or `.UTF-8` are also supported, but the actual value returned by `nl_langinfo(CODESET)` must be verified in the runtime environment.

Even if the input wrapper successfully delivers a valid UTF-8 byte sequence, if `using_utf8 == FALSE`, nano may treat those bytes as separate high-bit characters rather than a single Japanese character.

Typical symptoms include:

* Garbled Japanese text.
* Deleting only part of a multibyte character.
* Cursor movement entering the middle of a character.
* Search and display-width calculations becoming inconsistent.
* Incorrect editing behavior even for existing UTF-8 Japanese files.

This is an independent design issue. However, **it is less directly responsible for IME input being completely ignored than the earlier special-key misclassification.**

---

## 5. Items Determined Not to Be the Cause

### Missing UTF-8 Support at Compile Time

Because `--enable-utf8` is enabled, the absence of compiled UTF-8 support is not the direct cause.

### Linking Against the Narrow ncurses Library

MSYS2 ncurses 6.6 is built with `--enable-widec`, and the wide-character library is provided under the standard ncurses library name. Therefore, the problem is **not** simply that a non-wide-character version of ncurses is being used.

### IME Failing to Produce Unicode Characters

The native console path uses `ReadConsoleInputW()` and `UnicodeChar`. The issue is **not** that Unicode characters cannot be obtained, but rather that **the retrieved Unicode value is placed into the same integer namespace as ncurses special key codes**.

### The UTF-8 Conversion Formula Itself

When execution reaches the ordinary-character branch, the UTF-8 conversion logic correctly handles Japanese BMP characters. The real problem is that Japanese characters never reach that branch.

---

## 6. How to Verify Without Modifying the Source

The following values can be observed using a debugger or a debug build without changing the source code.

### Check 1: Terminal Environment

In PowerShell, record:

```powershell
$env:TERM
$env:LANG
$env:LC_ALL
chcp
```

Also identify the terminal being used:

* Windows Terminal
* Traditional conhost
* cmd.exe
* PowerShell
* MSYS2 mintty
* Git Bash, etc.

### Check 2: Input Value for "あ"

After confirming "あ" through the IME, verify the following values.

| Observation Point            |                                  Expected Value |
| ---------------------------- | ----------------------------------------------: |
| `_nc_console_read()`         |                                        `0x3042` |
| `_nc_wgetch()` input         |                                        `0x3042` |
| `_nc_wgetch()` return status |                                  `KEY_CODE_YES` |
| `wget_wch()` `wide_input`    |                                        `0x3042` |
| Repository input wrapper     |                   Enters the special-key branch |
| UTF-8 pending buffer         |                                        Not used |
| nano input processing        | Receives `12354` but does not insert it as text |

If these observations match, the primary cause can be considered confirmed.

### Check 3: Compare Terminal Drivers

Compare behavior between:

```text
TERM=#win32con
```

and

```text
Windows Terminal / ConPTY default configuration
```

If only the native console path fails, it strongly supports the Unicode code point versus special key collision hypothesis.

### Check 4: Compare Input and Paste

Using the same character ("あ"), compare:

1. Direct IME input
2. Clipboard paste
3. Opening an existing UTF-8 file

If only IME input fails, the console event path is implicated. If editing existing UTF-8 files is also broken, the runtime `using_utf8` locale detection is likely contributing as well.

### Check 5: Runtime UTF-8 Detection

Verify the following values:

```text
setlocale(LC_ALL, "") return value
nl_langinfo(CODESET) return value
using_utf8 value
```

The expected condition is:

```text
nl_langinfo(CODESET) == "UTF-8"
using_utf8 == TRUE
```

### Check 6: Saved Data

If "あ" is entered correctly, the saved file should contain the following bytes:

```text
E3 81 82
```

If instead only `0x42`, fragmented high-order bytes, or no data at all is saved, this helps identify where the character was lost in the input pipeline.

---

## 7. Why CI Did Not Detect the Issue

The current CI verifies:

* Package builds successfully.
* Installation succeeds.
* `nano.exe --version` executes.
* Required DLLs are present.
* There is no dependency on `msys-2.0.dll`.

However, it does **not** test:

* Interactive keyboard input.
* Japanese IME input.
* Differences between `win32con` and ConPTY.
* Runtime UTF-8 locale detection.
* Input, deletion, and saving of Japanese characters.

As a result, the build and startup tests can all pass while this input bug remains undetected.

---

## Final Assessment

The fault lies at the following boundary:

```text
ncurses Windows console driver
    ↓
_nc_wgetch() special-key detection
    ↓
wget_wch()
    ↓
nano-builder custom UTF-8 input wrapper
    ↓
nano byte-oriented input processing
```

The **most direct root cause** is:

> Japanese Unicode code points obtained from the Windows wide-character console input path are greater than `KEY_MIN`, causing ncurses to misclassify them as special keys. As a result, execution never reaches the UTF-8 conversion branch implemented by nano-builder.

This is a high-confidence conclusion derived by tracing the input value through the source code. However, because the distributed executable has not yet been dynamically tested in an actual Windows IME environment, systems using the VT/ConPTY path should also verify the runtime locale detection behavior.
