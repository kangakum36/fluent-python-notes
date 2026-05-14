# Chapter 4 — Text vs Bytes

## Character Issues

- The best definition of a character is a Unicode character.
- Items out of a Python 3 `str` are Unicode characters.
- Unicode separates the identity of characters from specific byte representations.
- The **identity** of a character, aka its **code point**, is a number from 0 to 1,114,111.
- The bytes that represent a character depend on the encoding in use. An **encoding** is an algorithm that converts code points to byte sequences. For example:
  - `A`'s code point is `U+0041`, encoded as the single byte `\x41` in UTF-8, and `\x41\x00` in UTF-16-LE.
  - The Euro sign `U+20AC` is 3 bytes in UTF-8 (`\xe2\x82\xac`), but 2 bytes in UTF-16-LE (`\xac\x20`).
- Code points → bytes is **encoding**; bytes → code points is **decoding**.

## Byte Essentials

- Two basic built-in types for binary sequences: `bytes` (introduced in Python 3) and `bytearray` (added in Python 2.6).
- Each item in `bytes` or `bytearray` is an integer from 0 to 255, **not** a one-character string.
- A slice of a binary sequence always produces a binary sequence of the same type.
- If `x` is of `bytes` type, `x[0]` retrieves an `int` but `x[:1]` retrieves a `bytes` of length 1. This is the same as any other sequence type.
- Display rules for binary sequence literals:
  - For bytes in the printable ASCII range, the ASCII character itself is used.
  - For bytes corresponding to tab, newline, carriage return, and `\`, escape sequences are used.
  - If both string delimiters `'` and `"` appear, the whole sequence is delimited by `'` and any `'` inside are escaped as `\'`.
  - For all other byte values, a hex escape sequence is used.
- `bytes` and `bytearray` support most string methods except those that do formatting and a few that depend on Unicode data.
- Regex operations work on binary sequences.
- Binary sequences have `fromhex`, which builds from pairs of hex digits.
- Can also build from a `str` and encoding keyword, or an iterable of values from 0 to 255.
- Building a binary sequence from a buffer-like object is a low-level operation which might involve type casting. This will always **copy** the source, unlike memory views.

## Basic Encoders/Decoders

- Python bundles more than 100 codecs for text-to-byte conversion and vice versa.
- Each codec has a name like `utf_8` and aliases. You can use the encoding argument in `open`/`encode`/`decode` functions, etc.
- Some encodings like ASCII cannot represent every Unicode character. The UTF encodings are designed to handle every Unicode code point.

### Some Well-Known Encodings

- **`latin1`** (aka `iso8859_1`) — basis for other encodings like `cp1252` and Unicode.
- **`cp1252`** — useful latin1 superset created by Microsoft, adding symbols like curly quotes and the Euro sign.
- **`cp437`** — original character set of the IBM PC, with box drawing characters. Incompatible with latin1.
- **`gb2312`** — legacy standard to encode simplified Chinese ideographs used in mainland China; one of several widely deployed multibyte encodings for Asian languages.
- **`utf-8`** — most common 8-bit encoding on the web by far. As of July 2021, 97% of sites use UTF-8.
- **`utf-16le`** — one form of the UTF-16 encoding scheme. All UTF-16 encodings support code points beyond `U+FFFF` through escape sequences.

## Understanding Encode/Decode Problems

- Most non-UTF codecs only handle a small subset of Unicode characters.
- When converting text to bytes, if a character is not defined in the target encoding, `UnicodeEncodeError` is raised unless special handling is provided by passing an `errors` arg to the encoding method or function.
- **`errors='ignore'`** — skips chars that cannot be encoded. This is usually a bad idea.
- **`errors='replace'`** — substitutes unencodable chars with `?`. Data is lost but users get a clue something is amiss.
- **`errors='xmlcharrefreplace'`** — replaces unencodable chars with an XML entity. If you can't use UTF and can't afford to lose data, this is the only option.
- Since ASCII is a common subset to every encoding, any string of pure ASCII chars (checked with `str.isascii`) should be encodable to bytes in any encoding.
- Not every byte holds a valid ASCII character, and not every byte sequence is valid UTF-8 or UTF-16. If unexpected bytes are found, you get `UnicodeDecodeError`.
- Many legacy 8-bit encodings are able to decode any stream of bytes without reporting errors (`cp1252`, `iso8859_1`, `koi8_r`).
- UTF-8 is the default source encoding for Python 3, but you can force the interpreter to use a different encoding scheme by placing a `# coding` comment at the top of the file, like `# coding: cp1252`.

## How to Discover the Encoding of a Byte Sequence

- You can't a priori find the encoding of a byte sequence; you must be told.
- Some protocols and file formats like HTTP and XML contain headers that tell us how content is encoded.
- It is almost impossible for a random sequence of bytes, or a nonrandom sequence coming from a non-UTF-8 encoding, to be accidentally decoded as garbage in UTF-8.
- If you assume a stream of bytes is human plain text, it might be possible to sniff out its encoding using heuristics.
  - For example, if `b'\x00'` is common, it's probably a 16- or 32-bit encoding, because null characters in plain text are bugs.
  - When `b'\x20\x00'` appears, it is likely to be the space character in a UTF-16-LE encoding.
- There is a library called **chardet** (and a CLI utility called `chardetect`), which is designed to guess one of more than 30 supported encodings.

## BOM

- At the beginning of UTF-16 encodings there is a **byte order marker** (BOM) which denotes if the bytes are in little-endian or big-endian order (depends on the CPU of the machine doing the encoding).
- On little-endian, the least significant bytes come first, so `E` (decimal 69) is encoded as `69` and `0`. On big-endian, the encoding would be reversed: `0` and `69`.
- To avoid confusion, UTF-16 prepends the text with the invisible character `U+FEFF`, which in little-endian is `b'\xff\xfe'`.
- **UTF-16-LE** and **UTF-16-BE** are explicitly little- and big-endian, so a BOM is not generated with them.
- There is a UTF-8 variant called **UTF-8-SIG** which does include a BOM, even though UTF-8 generally does not require one. It is useful to use UTF-8-SIG when reading just in case.

## Handling Text Files

- Best practice for handling text I/O is that bytes should be decoded to `str` as early as possible on input and as late as possible on output — called the **Unicode sandwich**. This allows the application to process 100% text.
- Python 3 makes this easier to follow, as the `open()` built-in does the necessary decoding when reading and encoding when writing.
- Never depend on encoding defaults — always pass an explicit `encoding=` argument.
- Interestingly, `fp.write()` returns the number of Unicode characters written, which may not be the number of bytes written.

## Encoding Defaults

- macOS and Linux use UTF-8 everywhere.
- Windows is more complicated: `locale.getpreferredencoding()` is `cp1252`. This is used by text files.
- Recently, UTF-8 support in Windows has gotten better.
- When directing output to a file, the encoding on Windows becomes `cp1252`, so trying to print UTF characters to a file will not work if using the default encoding.
- If you omit `encoding` when opening a file, it is set by `locale.getpreferredencoding()`.
- Encoding of `sys.stdout`/`stdin`/`stderr` is UTF-8 for interactive I/O, or `locale.getpreferredencoding()` if output/input is redirected to/from a file.
- `sys.getdefaultencoding()` is used internally by Python in implicit conversions of binary to `str`.
- `sys.getfilesystemencoding()` is used to encode/decode filenames.
- Based on the documentation, `locale.getpreferredencoding()` only returns a guess.
- Basically, **do not rely on defaults**.

## Normalizing Unicode for Reliable Comparisons

- String comparisons are complicated because Unicode has **combining characters**. This means diacritics and other marks that attach to the preceding character, appearing as one when printed.
- For example, `café` can be composed in 2 ways, using 4 or 5 code points, but the result looks the same.
- In this example, `'é'` and `'é'` are called "canonical equivalents," but Python sees 2 different sequences of code points and considers them not equal.
- The solution is `unicodedata.normalize()`, which takes a normalization form.
- **NFC** (Normalization Form C) composes the code points to produce the shortest equivalent string.
- **NFD** decomposes, separating characters.
- The W3C (World Wide Web Consortium) recommends NFC.
- **NFKC** and **NFKD** are stronger forms of normalization, affecting compatibility characters. Some characters in Unicode like the Micro Sign `µ` appear more than once for compatibility with preexisting standards. It is both the micro sign (`U+00B5`) and the Greek small letter mu (`U+03BC`). In NFKC and NFKD, each compatibility character is replaced by a compatibility decomposition of one or more characters considered a preferred representation, even if there is some formatting loss.
- This results in some interesting outcomes, like `4²` (4 squared) being converted to `42`, which changes the meaning. Therefore NFKC and NFKD may lose or distort information, but produce convenient representations for searching and indexing.

## Case Folding

- This is essentially converting all text to lowercase, with some additional transformations. This is supported by `str.casefold()`.
- For any string containing only latin1 characters, `casefold()` is the same as `s.lower()` except for the micro sign being changed to Greek lowercase mu and the German Eszett becoming `ss`.
- There are nearly 300 code points for which `str.casefold` and `str.lower` return different results.

Defining functions like:

```python
def nfc_equal(str1, str2):
    return normalize('NFC', str1) == normalize('NFC', str2)

def fold_equal(str1, str2):
    return (normalize('NFC', str1).casefold() ==
            normalize('NFC', str2).casefold())
```

is useful if working with text in many languages.

## Extreme Normalization: Taking Out Diacritics

- Google Search removes diacritics because many times, people use them incorrectly. Websites like Wikipedia also remove them from URLs to make the URLs more readable.
- You can remove diacritics by decomposing with NFD, removing all combining marks, and recomposing.
- Can also do things like removing diacritics only on Latin characters by decomposing and then checking if the base character is ASCII.
- Could also do things like replacing common symbols in Western texts with ASCII.
- These steps are extreme and might change the meaning of text.

## Sorting Unicode Text

- Python sorts by comparing items in each sequence one by one. For strings, this compares code points. This produces unacceptable results for anyone using non-ASCII characters.
- To sort non-ASCII text, the standard way is to use `locale.strxfrm`, which transforms a string to one that can be used in locale-aware comparisons.
- To enable this, you must set a suitable locale for your application and pray the OS supports it.
- Because locale settings are global, calling `setlocale` in a library is not recommended — the application should do it.
- The locale must be installed on the OS, otherwise you will get an "unsupported locale setting" exception.
- The locale must be correctly implemented by the makers of the OS.

## Sorting with the Unicode Collation Algorithm

- There is a simpler solution called **pyuca**, a pure Python implementation of the Unicode Collation Algorithm.
- You can use `pyuca.Collator()` as the sort key and get a good sorting result.
- pyuca doesn't take locale into account. If you need to customize the sorting, you can provide the path to a custom collation table to the `Collator()` constructor.
- pyuca has one sorting algorithm that doesn't respect sorting order in individual languages.
- There is also **PyICU**, which works like locale without changing the locale of the process, but has an extension that must be compiled.

## The Unicode Database

- Structured text files that include not only the table mapping code points to character names, but also metadata about individual characters and how they are related.
- Metadata like whether a character is printable, is a letter, is a decimal digit, or is another numeric symbol. This is how `isalpha`, `isprintable`, `isdecimal`, and `isnumeric` work. `str.casefold` also uses information from a Unicode table.

### Finding Characters by Name

- The `unicodedata` module has functions to retrieve character metadata, including `unicodedata.name()`.
- Can use this to build apps that let users search for characters by name.

### Numeric Meaning of Characters

- Unicode is able to tell whether any character is numeric and/or is a digit, and also display its value via `unicodedata.numeric(char)`. The regex for digits (`r'\d'`) is not as savvy — it isn't able to recognize things like a circled number 7, a superscript 2, or an Ethiopic digit three.

## Dual-Mode `str` and `bytes` APIs

### `str` vs `bytes` in Regex

- If you build a regex with `bytes`, patterns like `\d` and `\w` only match ASCII characters. If they are given as `str`, they match Unicode digits or letters beyond ASCII.
- For example, the Tamil digits for 1729 will only be matched by `\d` and `\w` regex patterns if they are in a string. Once they are encoded into bytes using UTF-8, regex will not match them.
- There is a `re.ASCII` flag that makes regex perform ASCII-only matching on `str`.

### `str` vs `bytes` in `os` Functions

- The Linux kernel is not Unicode-savvy, so in the real world there are filenames made of byte sequences that are not valid in any sensible encoding scheme.
- To work around this issue, all `os` module functions that accept file or path names take arguments as `str` or `bytes`.
- If a function is called with a `str` argument, it will be converted with a codec named by `sys.getfilesystemencoding()`.
- If you must deal with filenames that cannot be handled in that way, you can pass `bytes` arguments to the `os` functions to get `bytes` return values.
- `os` provides special encoding and decoding functions `os.fsencode` and `os.fsdecode` to help with manual handling of sequences that are filenames or path names.
