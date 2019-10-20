# The Verse Format

This document specifies a format for data storage and
transmission, suitable for binary or UTF-8 string data.
The format is intended to balance the needs of human and
machine consumers, so that it could be used e.g. for
Unix-style command line programs.

## Introduction

> Write programs to work together. Write programs to handle
> text streams, because that is a universal interface.
>
> — Doug McIlroy

The fact that Unix tools process streams of
newline-delimited text makes them delightfully simple to
use. However, it also imposes limitations that are difficult
to work around.

Consider the case of a command like `grep -C3 foo`. This
displays the lines from `STDIN` that contain `foo`, with
3 lines of surrounding context. Each "record" in the output,
consisting of the search result with context, is separated
from its neighbors by a line with two dashes.

For example:

```
$ man grep | grep -C3 foo
     beginning of a line, and the `\' escapes the `.', which would otherwise
     match any character.

     To find all lines in a file which do not contain the words `foo' or
     `bar':

--
     `bar':

           $ grep -v -e 'foo' -e 'bar' myfile

     A simple example of an extended regular expression:
```

The problem is that this doesn't compose with further `grep`
commands. Suppose I want to filter these records to just the
ones that also contain `bar`. That is, I'd like to see all
occurrences of `foo` that are within 3 lines of an
occurrence of `bar`.

I can't achieve this by piping the output of `grep` through
another `grep` command, because `grep` is only aware of
lines, not records. It doesn't know that `--` has a special
meaning, and even if it did, it might be confused if any
search results from the first `grep` contained `--` in the
literal text.

## Preliminary Concepts

The overall goal is to multiplex Unix files to allow them to
contain multiple, unambiguously delimited records. I use
the following terms in the rest of this spec:

- **document:** a binary or UTF-8-encoded string, file, or
  other chunk of data.
- **record:** a unit of data within a document; a document
  has zero to many records.
- **record separator:** an ASCII string that immediately
  precedes the beginning of a record, thus marking a
  boundary between records.

## Design Goals

The text of each record must be preserved exactly as is: no
additional escaping, formatting, or other modification. This
allows the text to be processed in a reasonable way by
programs that are not aware of the Verse format.

Record separators must be unambiguous: it should not be
possible for part of the text of a record to be mistaken for
a record separator.

The format must be easily understood by both humans and
machines.

The format must survive transmission through hostile
environments like paper, web forms, or the system clipboard.
That means no nonprintable characters.

The format must be composable: the individual records should
be able to contain documents that are themselves written in
the Verse format (or any other structured format, like JSON
or XML).

## Overview of the Verse Format

The Verse Format represents structured binary or
UTF-8-encoded string data. It can be thought of as a
multiplexing layer on top of Unix files, allowing one file
or stream to contain multiple documents.

Each record is introduced by a line consisting of the
*record separator* for the document. The record separator
can be any string consisting of non-whitespace, printable
ASCII characters in the range [33, 126]. Since the first
line of the document is *always* a record separator, it is a
de facto declaration of what the record separator is for
that document.

For example, the example document below uses `====` as the
record separator:

```
====
this is record 1
====
this is record 2
====
the next two records are empty
====
====
====/
```

It is up to the program emitting the data to ensure that
the record separator does not appear in the text of the
records themselves. A couple algorithms for doing this come
to mind:

When the full text of all the records to be emitted can fit
in memory at once, the emitting program can choose an
arbitrary record separator (e.g. `====`) and verify that the
separator does not appear in any of the records before it
starts emitting the data. If its chosen separator does
appear, the program can simply lengthen the separator (e.g.
doubling its length) and repeat the check. By repeating this
process, it is guaranteed that a suitable separator will be
found.

When the full text of the records does not fit in memory,
and a multi-pass algorithm is not feasible, the emitting
program must choose a separator before it knows what the
text of the records will contain. In this case, the safest
thing to do is generate a random separator that is
vanishingly unlikely to appear in the data. For example, a
random base64 string longer than 10 characters or so has
sufficient entropy that it is very unlikely to occur in the
data.

The newline characters on either side of the separator
string are considered part of the separator. Thus, the
following C string

```
"====\nthis is record 1\n====\nthis is record 2\n====/\n"
```

parses to a two records:
`"this is record 1"` and `"this is record 2"`.
That is, the records do not contain any newlines.

To represent records with newlines, additional newline
characters can of course be added:

```
"====\n\na record\nwith line breaks\n\n====\nanother one\n====/\n"
```

Here, the first record is
`"\na record\nwith line breaks\n"`.

### End of the Document

The end of the document is signaled by a final record
separator with an appended `/`, followed by a newline. This
allows the program receiving the data to know if the data
was cut short (which could happen if, for instance, the
process emitting it was killed).

TODO: is this really necessary?

### Composition

The format may be embedded in itself by choosing a second,
non-conflicting separator for the inner documents.

For example, the following shows an example of nesting the
format to produce a two-dimensional table:

```
====
----
row 1, column 1
----
row 1, column 2
----/
====
----
row 2, column 1
----
row 2, column 2
----/
====/
```

The receiving program must be aware of how many levels of
nesting there are if it is to correctly parse them all.

## The Empty Document

An empty set of records can be represented as an empty
document.
