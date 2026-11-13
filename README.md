# csv

Comma-separated values for Avin, as [RFC 4180](https://www.rfc-editor.org/rfc/rfc4180) writes them:
reading text, bytes or a stream a row at a time, records whose fields are found by their column's
name, any ASCII delimiter (a tab for tab-separated values), and writing that quotes a field only
where it must. Every error is a `Result`'s `Err` carrying the line and column of the fault.

```text
avin add github.com/avin-lang/csv
```

```avin
import dev.avinlang.csv as csv
```

## Reading

```avin
import dev.avinlang.csv as csv

main()
    val rows = csv.parse("name,note\nAda,\"first, and \"\"best\"\"\"\n") match
        Ok(rs) -> rs
        Err(e) -> fail(s"bad CSV: $e")
    println(rows)                 // [[name, note], [Ada, first, and "best"]]
    println(csv.parse("a,\"b"))   // Err(a quoted field is not closed at line 1, column 3)
```

| Call | Answers |
|---|---|
| `csv.parse(text, delimiter = ',')` | `Result[Vector[Vector[string]], CsvError]`: every row, each a vector of its fields |
| `csv.parse(bytes: Array[byte], delimiter = ',')` | the same, for UTF-8 bytes |
| `csv.parseRecords(text, delimiter = ',')` | `Result[Vector[CsvRecord], CsvError]`: the rows after the header, each a record |
| `csv.parseRecords(bytes, delimiter = ',')` | the same, for UTF-8 bytes |

The rules the reader follows:

- A field that begins with `"` is quoted: it runs to the next lone `"`, and holds delimiters, line
  breaks and `""` (one quote) freely. Only the delimiter, a line end or the end of the text may follow
  its closing quote.
- A field that does not begin with `"` holds no quote and runs to the delimiter or the line end.
  Spaces are part of a field and are never trimmed.
- A row ends at a line feed, a carriage return and line feed, or a carriage return alone. The last row
  needs no line end. A line end inside a quoted field is part of the field, as written.
- A line with nothing on it holds no row; a row of one empty field is written `""`.
- A byte order mark (U+FEFF) at the very start of the text is dropped; anywhere else it is text.
- Rows may differ in length, except in header mode.

## Records: rows by column name

```avin
import dev.avinlang.csv as csv

main()
    val people = csv.parseRecords("name,born\nAda,1815\nGrace,1906\n").ok.get
    for p in people
        println(s"${p("name")} was born in ${p("born")}")
    println(people(0))                       // {name: Ada, born: 1815}
    println(people(0).get("died"))           // None
```

`CsvRecord` gives a field by its column's name or its position: `r("name")` and `r(0)` trap where
there is no such column, `r.get("name")` and `r.get(0)` answer an `Option`. It also has `has(name)`,
`size`, `fields` (a `Vector[string]`), `header` and `toMap`. `CsvHeader` has `names`, `size` and
`column(name) -> Option[int]`. Where a name is written more than once, it finds its first column.
A row with more or fewer fields than the header is an error at the row's line: `the row has 3 fields
where the header names 2 at line 4, column 1`.

## Reading a stream

A `CsvReader` reads from any `avin.io.Reader` — an open file, a connection, a source of the program's
own — 64 KiB at a time, holding only that piece and the row it is reading, so a file larger than
memory reads in a bounded amount of it.

```avin
import dev.avinlang.csv as csv
import dev.avinlang.csv.CsvReader
import avin.fs

// The sum of the "bytes" column of a log far larger than memory.
totalBytes(name: string) -> Result[long, Error]
    val r = CsvReader.fromReader(fs.open(name)?)
    var sum: long = 0
    while r.nextRecord()? is Some(rec)
        sum += long(rec("bytes").toInt())
    Ok(sum)
```

| Call | Does |
|---|---|
| `CsvReader.fromReader(source, delimiter = ',')` | a reader of a stream |
| `CsvReader.fromString(text, delimiter = ',')`, `CsvReader.fromBytes(bytes, delimiter = ',')` | a reader of text in hand |
| `r.next()` | `Result[Option[Vector[string]], CsvError]`: the next row, `None` at the end |
| `r.readHeader()` | `Result[CsvHeader, CsvError]`: the next row as the header, read once; later calls give it again |
| `r.nextRecord()` | `Result[Option[CsvRecord], CsvError]`: the next row under the header, reading the header first where it has not been |
| `r.lastRowLine` | the line the last row given began on |

Text that breaks the rules, or a source that fails to read, is the `Err` of that call and of every call
after it. A ragged record is the `Err` of its own call only: the next call reads the row after it, so a
program can report a bad row and go on.

## Writing

```avin
import dev.avinlang.csv as csv

main()
    print(csv.render([["name", "note"], ["Ada", "first, and \"best\""], [""]]))
    // name,note
    // Ada,"first, and ""best"""
    // ""
```

| Call | Does |
|---|---|
| `csv.render(rows, delimiter = ',', crlf = false)` | the rows as text, one line each |
| `csv.renderRow(fields, delimiter = ',', crlf = false)` | one row, with its line end |
| `CsvWriter(out: Writer, delimiter = ',', crlf = false)` | a writer onto any `Writer` (a `StringWriter`, standard output, one of the program's own) |
| `w.writeRow(fields)`, `w.writeRows(rows)` | rows of `Vector[string]` |
| `w.writeHeader(header)`, `w.writeRecord(record)` | a header's names, a record's fields |

A field is quoted only where it must be: where it holds the delimiter, a quote, a carriage return or a
line feed (each quote inside doubled); where it begins with a byte order mark; and where it is the
only field of its row and empty. Every row ends with a line feed, or with a carriage return and line
feed where `crlf` is true. Anything written reads back as the rows it was written from.

## Errors

`CsvError(message, line, column)` is an `Error`, so `?` carries it into a `Result[T, Error]`. Lines
and columns count from 1; columns count characters, and lines count every line end, those inside a
quoted field too. It displays as `message at line L, column C`. The faults:

| Message | Placed at |
|---|---|
| `a quoted field is not closed` | the opening quote |
| `a quote inside a field that does not begin with one` | the quote |
| `expected the delimiter or the end of the line after a quoted field` | what follows the closing quote |
| `a field that is not UTF-8` | the field's start |
| `the row has N fields where the header names M` | the row's line, column 1 |
| `the text could not be read: …` | where the reader had got to |

A delimiter that cannot separate fields — a quote, a carriage return, a line feed, or a character
outside ASCII — stops the program when the reader or writer is made, as asking a record for a column
it lacks does.

## Tests

```text
avin test .
```

The examples above run as one program in `examples/readme` (`avin run examples/readme`).

## Decisions

Choices RFC 4180 leaves open, made here; each says why and what else was weighed.

1. **Strict quotes.** A quote inside an unquoted field (`ab"c`), and anything but the delimiter or a
   line end after a closing quote (`"a"b`), are errors, not text. *Why:* RFC 4180 forbids both, and
   reading them leniently silently accepts text that was quoted wrongly, which is how a column shifts
   unnoticed. *Alternatives:* keep the quote as text (common in lenient readers); an option to choose.
   An option can be added later without breaking anyone.
2. **Blank lines hold no row.** *Why:* files commonly end with an extra blank line, and in header
   mode a blank line read as a row of one empty field would be a ragged-row error. A row of one empty
   field is still expressible, as `""`, and the writer writes it so. *Alternative:* a blank line as
   `[""]`, which RFC 4180 implies.
3. **A carriage return alone ends a row**, as CRLF and LF do. *Why:* text from old tools uses it, and
   an unquoted carriage return has no other sensible reading. *Alternative:* only CRLF and LF, with a
   lone CR kept in the field.
4. **The writer ends rows with LF by default**, CRLF with `crlf = true`. *Why:* the files are mostly
   read by Unix tools, diffs and the reader itself, which takes any line end. *Alternative:* CRLF by
   default, as RFC 4180 writes. Worth a second look if interchange with spreadsheets is the main use.
5. **A delimiter is one ASCII character other than a quote, CR or LF**; any other is a trap when the
   reader or writer is made. *Why:* the reader scans bytes, and these are every delimiter in use
   (`,` `\t` `;` `|` space). A bad delimiter is a mistake in the program, not in the data, so it traps
   rather than answering an `Err`. *Alternatives:* any character; an `Err` from every call.
6. **A leading byte order mark is dropped on reading, and a field beginning with one is quoted on
   writing**, so every row reads back as written. *Alternative:* keep the mark as text.
7. **Header mode:** a name written twice finds its first column; text with no rows has a header of
   no names and no records; a ragged record is an error of its own call and the reader goes on.
   *Why:* the first column is what a person reading the file sees first; going on lets a program
   report every bad row in one pass. *Alternatives:* the last column (as a JSON object's last member
   wins); an error for a duplicate name; a ragged row stopping the reader.
8. **The column of a ragged-row error is 1**, the row's start, not the extra or missing field.
   *Why:* the fault is the row as a whole. 
9. **Fields are strings.** No type conversion or derived decoding of a program's own types yet; a
   record's fields are converted by the caller (`rec("n").toInt()`). *Alternative:* a decoder
   derived for a data type, as `avin.json` has; a natural next step.
