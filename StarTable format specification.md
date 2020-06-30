# The StarTable format specification

## What is StarTable?

StarTable is a data format designed to conveniently store an arbitrary number of two-dimensional tables of data in a way that is 

- human- and machine-readable; and
- easy to edit, audit, and version. 

In addition to tabular data, StarTable also supports 

- metadata at the file, table, and columns levels; 
- templating and comments; and
- application-specific functionality such as relations between tables and include statements. 

The StarTable format is file-format agnostic. In practice, CSV files and Excel workbooks are most commonly used to store StarTable-formatted data. But any file format that can be used to represent a set of tables, each with columns and rows of cells, can, in principle, adhere to the StarTable format.

### What problem StarTable solves

StarTable is geared towards easing human quality assurance while remaining machine-readable and versionable. Thus it differentiates itself from other data formats:

- Supports "generalist" file formats such as Excel and CSV, which: 
  - Are easy to open/edit/QA with general-purpose software available on most machines (i.e. including non-technical stakeholders); and
  - Can be versioned with enterprise document management systems and/or text-based version control systems.
- One file or many files – up to you
  - One file *can* contain all the tables required for a given project. 
  - Though you can also *choose* to split one StarTable file into multiple files according to some convenient logic; for example, different files having different owners responsible for their content, under different versioning.

Alternate data formats such as SQLite, other databases, and zip archives of text files, do not naturally fulfill these needs. 

### History

The StarTable format traces its origins to a single software project in the Foundations department of Offshore Wind at [Ørsted](https://orsted.com/) in the mid 2010's, in which it was, and still is, used by engineers for the entry, input/output, quality assurance, and version control of technical data related to structural calculations for wind turbine foundations. 

By 2018, the format's use had spread to five independent projects within the company due to its recognized convenience and flexibility. A common governance structure was established in December 2018 to ensure that continued development of the StarTable format would remain unified while meeting the needs of its diverse client projects within Ørsted. 

In February 2019, approval was granted to open source not only the StarTable standard itself, but also the suite of software packages and utilities that allow reading/writing/manipulating/displaying StarTable files in various programming languages and technologies. Governance remains Ørsted-based for the time being, though this is liable to change if, as hoped for, a community of users emerges outside Ørsted. 

This concludes the soft part of this document. What follows is a more formal, detailed, and rigorous description of the StarTable format. 

## Hierarchical structure

Data in StarTable format are contained in a file. Since the StarTable format is file-format agnostic, files are not part of the StarTable format proper, and a
detailed discussion of file formats is beyond the scope of this document. 

Here is an illustration of the hierarchical structure of a StarTable file:

![High-level hierarchical structure of the StarTable format](media/hierarchical-structure-diagram.png)

The highest-level structure of the StarTable format proper is the
*sheet*. Some file formats (such as CSV) can contain only one sheet, while others (e.g. Excel workbook) can contain multiple sheets. 

A sheet contains a series of *blocks* of different types. 

Blocks consist of two-dimensional arrays of cells, each containing an *atomic value*. The detailed structure of any given block depends on its type. Blocks of type table are the main data
container, while other block types provide supplementary information and
functionality.  

We will now describe each of the members of this hierarchy in turn. 

## <a name="atomic-types"></a>Atomic types and values

Cells each contain a value of one of the following atomic types:

- String
- Numeric types:
  - Number (no distinction is made between e.g. floating-point and integer)
  - Boolean (represented as 1 or 0)
  - DateTime

String type cells that are empty are to be treated as containing an empty string. 

Numeric type cells may not be left empty. Null or missing values may 
be represented by either of the following valid null markers: `-`, `nan`, `NaN`, or `NAN`.

#### Restrictions on strings

Strings may not contain characters used to represent the end of a line
(such as end of line and line feed) as this would introduce ambiguity
between cell content and the end of a row. Note that this remark applies
throughout this document: there are a few instances where we indicate
that a given field can consist of “any string”; this should be
understood as shorthand for, any string not including these aforementioned forbidden characters.

#### Additional restrictions on symbol strings

*Symbols*, such as table names, column names, and destinations, are represented as
strings. In addition to general restrictions on strings as described above, these *symbol strings* are subject to the following restrictions:

- They may only contain alphanumeric characters and `_` (underscore); and
- The first character may not be a digit. 

## File

The StarTable format is file-format agnostic. To be StarTable-ready, a given file format must be able to represent a
two-dimensional array of cells, with the array being of arbitrary length
and width, and with each cell containing a value of one of the allowed [atomic
types](#atomic-types). 

Examples of StarTable-ready file formats are CSV (Comma-Separated Values), which can contain only one sheet, and Excel workbooks, which can
contain multiple sheets. At the time of writing this document, these are
the only two StarTable file formats used in practice. This
should not, however, be understood as a formal limitation. In principle,
nigh any file format could be used for StarTable files – though not
all to the same degree of convenience. Further conceivable examples of
a file format that can contain multiple sheets are:

- JSON
- A compressed zip archive of multiple CSV files.

### Parsing a file into sheets

In file formats where files can contain multiple sheets, each sheet must
be named such as to be uniquely identifiable. Parsers may include an option to only process sheets with names matching a certain regular expression. 

## Sheets

A StarTable sheet consists of an arbitrary number of rows, each
containing an arbitrary number of cells. 

### Parsing a sheet into blocks

A sheet's rows can be parsed into a series of blocks. Any given row on a sheet can only be a member of one block, but all rows need not be in a block. 
(Rows that are not in a block are treated as *comments*.)

The block syntax is designed such that only the first column need be
considered in order to unambiguously split a sheet i.e. allocate its rows into blocks. Block
start markers are defined by the contents of cells in this first column.
The same is true of end markers – except in the case of metadata lines,
which are single-row blocks and require no end marker other than their
end of line. 

Blocks start when their start marker is encountered, and end when their
end marker is encountered. Blocks always include their start marker
cell, but exclude their end marker.

## Blocks

The primary content of StarTable files is typically placed in “table” blocks, but there are
additional block types that provide supplementary information and
functionality. The different block types are summarized in this table:

| Block type    | Start marker first-column cell content                       | End marker                                             | Description & remarks                                        |
| ------------- | ------------------------------------------------------------ | ------------------------------------------------------ | ------------------------------------------------------------ |
| Directive     | `***` followed by directive block name                       | - Empty first column cell; or<br>- New block start     | Placeholder for a variety of functions e.g.  <br> - Version control <br> - Allowing tables from various sources to be added dynamically to the set of tables statically present in a sheet |
| Table         | `**`  followed by table name                                               (which may not start with `*`) | - Empty first column cell; or <br> - New block start   | Primary data content                                         |
| Template      | Starts with `:`                                              | -   Empty first column cell; or <br> - New block start | Template data embedded in template files allow input files to be matched against a template, and provide a description of input data. |
| Metadata line | Ends with `:`, and is not a valid start marker for one of the other block types | End of line                                            | Are only accepted at the top of a sheet, before any other block types.<br />Provide information about the current sheet.<br />Always span exactly one line. |

The following sections describe the structure of these block types.

### Metadata line block

Metadata lines contain information about the file. Typical metadata fields are: author, verified by, etc.

StarTable parsers may implement a system of metadata field pseudonyms referring to canonical field names. 

### Directive block

The first cell starts with `***` followed by a directive name symbol. 

The content of a directive block is to be sent to the client application as cell arrays, to be handled by the application. 

Example use cases:

- Revision history
- "Include" statements, indicating to the application that additional StarTable files are to be read. 

### Table block

A table block consists of:

-   The start marker prefix, `**`, followed by a *table name*;

-   *Destination list*, indicating what content this table applies to;
    and

-   An arbitrary number of vertical *table columns*, arranged
    side-by-side horizontally. Each column consists of:
    -   A *column name*, describing the contents of the column;

    -   A data type / unit indicator, hereafter simply referred to in shorthand as *unit*, used to distinguished between
        text content, unitless numerical content, and numerical content
        with units; and

    -   An arbitrary number of values. All columns within a given
        table must have the same number of rows i.e. values. 

An example of these elements is illustrated this figure: 

![Example table block](media/table-block-example.png)


Generic example of a table block:

| `table name`   |                  |                  |       |
| ------------------ | ---------------- | ---------------- | ----- |
| **`destinations`** |                  |                  |       |
| **`col 1 name`**   | **`col 2 name`** | **`col 3 name`** | **…** |
| **`col 1 unit`**   | **`col 2 unit`** | **`col 3 unit`** | **…** |
| `col 1 val 1`      | `col 2 val 1`    | `col 3 val 1`    | …     |
| `col 1 val 2`      | `col 2 val 2`    | `col 3 val 2`    | …     |
| `col 1 val 3`      | `col 2 val 3`    | `col 3 val 3`    | …     |
| …                  | …                | …                | …     |

We will now describe the elements of a table block. 

#### Table name

Along with its prefix `**`, the table name symbol marks the first row of the
table block. It is in the first column.

The *table name* is intended to describe what the table is about. It can
be any single-line string subject to restrictions on symbol strings, and
not starting with `*` (so as not to be confused, in conjunction with its
prefix `**`, with a directive block start marker, `***`).

#### Destination list

The destination list is in the first-column cell on the second row of
the table block. It is a space-delimited list of destination symbols.

The use of destinations is application-specific. Typical use cases include: 

- Establishing relationships between table blocks
- Namespacing

#### Table columns

Table columns start on the third row of the table block and occupy the
remaining rows of the table block all the way down to its end. 

##### Column name

The first cell of a table column is the *column name* symbol, which is intended
to describe the contents of the column. It can be any string.

The column name must be unique within the current table i.e. no two columns of a given table may have the same name.

##### Unit

The second cell of a table column is the *unit* symbol, where "unit" is used as shorthand for "data type / unit indicator". The unit specifies how the column’s values should be interpreted semantically. It does not necessarily represent a physical unit of measurement. 
Valid units and their semantic interpretation are described in the table below.

| Unit                       | Semantic interpretation of column values                     |
| -------------------------- | ------------------------------------------------------------ |
| `text`                     | Text                                                         |
| `datetime`                 | Datetime value, with format conforming to [ISO 8601](https://xkcd.com/1179/). |
| `onoff`                    | Boolean, represented as `1` for True, `0` for False.         |
| `-` <br>(a single "minus") | Unitless (non-dimensional) numerical values                  |
| Any other string           | Numerical values with physical unit specified by the unit string. |


##### Values

The remaining cells of a table column contain data values, which can be
of any of the valid atomic types, but must comply with the column's unit.

#### Gotcha: empty string in first column ends table block

Empty string is a legal value for cells in `text` columns. However, if the first column of a table block is of unit `text` and an empty string is encountered in this column, this will trigger the end of the table block. Therefore, empty strings must not be entered in the first column of a table block. 

If an empty string is inadvertently entered as part of the first column, any data in this and subsequent rows will not be interpreted as being part of this table block. A faithful StarTable parser will interpret them as being comments and ignore them, until the start of a new block is encountered. 

### Template block

*Template blocks* tell us something about the contents of the file; either about the file as a whole, the table immediately preceding the template block, or a column in that table. 

Template data embedded in template files allow input files to be matched
against a template, and provide description of input data.

Start marker cell has the regex form

```
^(?level:{1:3})(?identifier(\w*))(\.(?property\w+))?\s*$
```

The number of colons in the prefix determine the level to which this
template block applies. The characteristics of the three template block
levels are summarized in this table:

| Start marker prefix | Level  | Identifier must be a… | Default identifier (if omitted)                                                                                                                                                          | Default property (if omitted) |
|---------------------|--------|-----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------|
| `:::` <br>(three colons) | Sheet  | File name             | Current file name   | description                   |
| `::` <br>(two colons) | Table  | Table name            | Latest table block | description                   |
| `:` <br>(a single colon) | Column | Column name           | Most recent column name in a column level block. <br />It is an error not to specify a column identifier if no valid column level template block has appeared after the most recent table block. | description                   |

The property is optional. If it appears, it must be one of:

| Property name         | Applies to file | Applies to table | Applies to column | Semantics |
| --------------------- | :--------: | :--------: | :--------: | ---- |
| `description` (default) | x          | x         | x | Use column 2 as description of this item. Multiple descriptions may be given, in which case a single multi-line description text should be reported. |
| `case`                  |            |           | x | This component is optional |
| `use_template`         |            |           | x | For each value *s*, the string [col 3]+*s*+[col 4] is a table name for a table that should match the template in [col 2]. |
| `is_template`          | x          |           | | This table should only be used for templating |
| `is_optional`          |            | x         | x | |

Examples:

| Start marker     | Applies to                        | Property |
| ---------------- | --------------------------------- | -------- |
| `:n_legs`        | Column `n_legs` in previous table | N/A      |
| `::farm_animals` | Table `farm_animals`              | N/A      |
|                  |                                   |          |
|                  |                                   |          |

The main purpose of the template system is to aid work on the file level, where destinations cannot be resolved. For this reason, tables are identified by table names only for the purpose of template-matching.

