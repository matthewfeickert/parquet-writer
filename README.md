# parquet-writer
A C++ library that allows for easily writing Parquet files containing columns of (mostly) whatever type you wish.

## Motivation

There is a lot of boilerplate in setting up basic writing of Parquet
files in C++. The idea is for the `parquet-writer` library to make it
simple to both specify the desired layout of a Parquet file (i.e.
the number and types of columns) to be written
and to subsequently write to that file.

The `parquet-writer` library aims to provide support for:

  * Specifying the layout of Parqeut files (what types of columns to store) via JSON
  * Storage for numeric data types and boolean values
  * Storage of one-, two-, and three-dimensional lists
  * Storage of struct objects with any number of arbitrarily typed fields
  * A common interface for writing data types to output Parquet files

## Basic Usage

The `parquet-writer` library provides users with the `parquetwriter::Writer`
class, which users provide with a JSON object specifying the desired structure
of their output Parquet file and then fill accordingly.
The basic usage is illustrated in the following:

```c++
#include "parquet_writer.h"
#include "nlohmann/json.h"

// JSON from string literal via nlohmann/json
auto file_layout = R"(
  {
    "fields": [
      {"name": "column0", "type": "float"},
      {"name": "column1", "type": "int32"}
    ]
  }
)"_json;

// initialize the Parquet writer instance
namespace pw = parquetwriter;
pw::Writer writer;
writer.set_layout(file_layout);
writer.set_dataset_name("my_dataset"); // must give a name
writer.initialize();

// get and fill the data
float column0_value = 42.0;
int32_t column1_value = 42;

// for each row in the output file, fill each of the columns 
// (note the surrounding curly braces)
writer.fill("column0", {column0_value});
writer.fill("column1", {column1_value});

// signal that writing to the current row is complete
writer.end_row();

// when finished writing the file, call finish()
writer.finish();
```

## Supported Data Types

The primary data types that can be written to output Parquet files
by the `parquet-writer` library are summarized in the table below,

| Type | Supported Types in `parquet-writer` |
| --- | --- |
| **Logical types** | `bool` |
| **Signed integers** | `int8`, `int16`, `int32`, `int64` |
| **Unsigned integers** | `uint8`, `uint16`, `uint32`, `uint64` |
| **Floating Point** | `float` (32-bit precision)|
|                    | `double` (64-bit precision) |
| **Lists** | 1 dimensional: `list[<type>]` |
|       | 2 dimensional: `list[list[<type>]]` |
|       | 3 dimensional: `list[list[list[<type>]]]` |
| **Structs** | `struct{<fields>}` |
| **List of Structs** | 1 dimensional: `list[struct{<fields>}]` |
|                 | 2 dimensional: `list[list[struct{<fields>}]` |
|                 | 3 dimensional: `list[list[list[struct{<fields>}]` |

Where `struct{<fields>}` demarcates a data type comprised of any number
of arbitrarily-typed named-fields (think: C++ `struct`).

Fields of `struct` type can have fields that are basic value types
(e.g. the integer, floating point, and logical data types)
as well as 1-, 2-, and 3-dimensional lists of these basic value types.

The fields of `struct` type can also be of `struct`-type or `list[...[struct]]`-type, so long as these
sub-`struct`-type fields do not contain fields that are themselves
of `struct` type.
That is, **there can only be one level of sub-`struct` nesting**:
  * `struct`-type elements of lists of structs cannot have fields that are of `struct`-type
  * `struct`-type top-level columns can have fields that are themselves of `struct`-type, so long as these sub-`struct`s do not themselves have fields that are of `struct`-type

## Parquet File Layout Specification

Specifying the desired layout of a given Parquet file
with the `parquet-writer` library is done using JSON.
For basic types one need only provide the name of the output column
to be stored as well as its corresponding data type, as in the following:

```c++
auto file_layout = R"(
  "fields": [
    {"name": "int32_column", "type": "int32"},
    {"name": "float_column", "type": "float"},
    {"name": "double_column", "type": "double"}
  ]
)"_json;
```
The above JSON specifies a Parquet file containing 3 columns
named `int32_column`, `float_column`, and `double_column`
holding `int32`, `float`, and `double` data types, respectively.
Note that the `name` field associated with a given column
is completely arbitrary.

The specification for more complex data structures is detailed in the
sections below, specifically
going over how to declare columns storing `list` and `struct`
typed data structures.

Complete examples can be found in the [Examples](#examples) section.

### Lists of Basic Data Types

Storing one, two, and three dimensional lists of the [basic types](#supported-data-types) is supported
by `parquet-writer`. Specifying lists of these types is done via
the JSON layout provided to a given `parquetwriter::Writer` instance.

For example, the following JSON layout specifies a Parquet file
containing a one-dimensional variable-lengthed list column
named `my_1d_list`, a two-dimensional variable-lengthed list column named `my_2d_list`,
and a three-dimensional variable-lengthed list column named `my_3d_list` holding
`float`, `uint32`, and `double` types, respectively:
```c++
auto file_layout = R"(
  {
    "fields": [
      {"name": "my_1d_list", "type": "list", "contains": {"type": "float"}},
      {"name": "my_2d_list",
                "type": "list", "contains":
                        {"type": "list", "contains": {"type": "uint32"}}
      },
      {"name": "my_3d_list",
                "type": "list", "contains":
                       {"type": "list", "contains":
                                {"type": "list", "contains": {"type": "double"}}
                       }
      }   
    ]
  } 
)"_json;
```
As can be seen in the above, specifying `list` types for output columns requires
an additional `contains` object in the JSON defining the column. This
`contains` object defines the data type to be stored in the output variable-lengthed
list.

Filling these `list` types with a `parquetwriter::Writer` instance is done
using standard `C++` `std::vector` instances of the associated `C++` type.
For example, taking the above specification one would do:
```c++
// one-dimensional case
std::vector<float> my_1d_list_data{1.2, 2.3, 3.4};
writer.fill("my_1d_list", {my_1d_list_data});

// two-dimensional case
std::vector<std::vector<uint32_t>> my_2d_list_data{
                                    {42}, {19, 27, 32}, {}, {72, 101}
                                  };
writer.fill("my_2d_list", {my_2d_list_data});

// three-dimensional case
std::vector<std::vector<std::vector<double>>> my_3d_list_data{
                                    { {0.5, 1.2}, {3.0, 4.0, 5.0}, {} },
                                    { {42.0}, {10.23}, {11.34} }
                                  };
writer.fill("my_3d_list", {my_3d_list_data});
```

### Struct Data Types

Storing complex data structures with
any number of named fields of possibly different data type (i.e. a `C++` `struct`) is possible.
These correspond to Parquet's [StructType](https://arrow.apache.org/docs/cpp/api/datatype.html#_CPPv4N5arrow10StructTypeE).
Specifying these complex data types is done via the `struct` type.

For example, storing a structure named `my_struct` having three fields named `field0`,
`field1`, and `field2` of data types `int32`, `float`,
and `list[float]`, respectively, is done as follows:

```c++
auto file_layout = R"(
  "fields": [
    {"name": "my_struct", "type": "struct",
                                  "fields": [
                                    {"name": "field0", "type": "int32"},
                                    {"name": "field1", "type": "float"},
                                    {"name": "field2", "type": "list", "contains": {"type": "float"}}
                                  ]}
  ]
)"_json;
```

As can be seen, `struct` types contain an additional `fields` array, which contains an array of
the usual `{"name": ..., "type": ...}` objects which describe each of the fields contained
in the output `struct` data structure.

#### Filling Struct Data Types

Since the `struct` type implies a complex data structure with arbitrarily-typed nested fields,
there is a convenience type that is used for filling this data type: `parquetwriter::struct_t`.
An instance of `parquetwriter::struct_t` can be treated as an `std::vector`, but one that
holds any of the [supported types](#supported-data-types).
For example, one would fill the three-field structure `my_struct` from above as follows:
```c++
namespace pw = parquetwriter;
int32_t field0_data = 42;
float field1_data = 10.5;
std::vector<float> field2_data{1.2, 2.3, 3.4};
parquetwriter::struct_t my_struct_data{field0_data, field1_data, field2_data};
/*
 // could also do:
 namespace pw = parquetwriter;
 pw::struct_t my_struct_data;
 my_struct_data.push_back(field0_data);
 my_struct_data.push_back(field1_data);
 my_struct_data.push_back(field2_data);
*/
writer.fill("my_struct", {my_struct_data});
```

#### Struct DataType Fill Ordering

The ordering of the elements in an instance of `parquetwriter::struct_t` must
absolutely follow the order in which they are specified in the JSON specification
of the corresponding `struct` type. That is, doing
```c++
pw::struct_t my_struct_bad_data{field2_data, field0_data, field1_data};
```
instead of what is shown in the previous code snippet would lead to an error since
the file layout for the data structure `my_struct` expects data types ordered
as `int32, float, list[float]` but `my_struct_bad_data` fills the `parquetwriter::struct_t`
with data ordered as `list[float], int32, float`.

### Lists of Struct DataType

There are convenience types for filling columns containing data types that are nested
lists of `struct` typed objects: `parquetwriter::struct_list1d`, `parquetwriter::struct_list2d`,
and `parquetwriter::struct_list3d`.

Examples of filling one-, two-, and three-dimensional lists containing `struct` typed
objects with three `float` typed fields are below,

```c++
namespace pw = parquetwriter;

// dummy values for the three fields of the struct
float field0_data = 42.0;
float field1_data = 84.0;
float field2_data = 126.0;

// one-dimensional case: list[struct{float, float, float}]
pw::struct_list1d my_1d_structlist_data;
for(...) {
  pw::struct_t struct_data{field0_data, field1_data, field2_data};
  my_1d_structlist_data.push_back(struct_data);
}
writer.fill("my_1d_structlist", {my_1d_structlist_data});

// two-dimensional case: list[list[struct{float, float, float}]]
pw::struct_list2d my_2d_structlist_data;
for(...) {
  std::vector<pw::struct_t> inner_list_data;
  for(...) {
    pw::struct_t struct_data{field0_data, field1_data, field2_data};
    inner_list_data.push_back(struct_data);
  }
  my_2d_structlist_data.push_back(inner_list_data);
}
writer.fill("my_2d_structlist", {my_2d_structlist_data});

// three-dimensional case: list[list[list[struct{float, float, float}]]]
pw::struct_list3d my_3d_structlist_data;
for(...) {
  std::vector<std::vector<pw::struct_t>> inner_list_data;
  for(...) {
    std::vector<pw::struct_t> inner_inner_list_data;
    for(...) {
      pw::struct_t struct_data{field1_data, field2_data, field3_data};
      inner_inner_list_data.push_back(struct_data);
    }
    inner_list_data.push_back(inner_inner_list_data);
  }
  my_3d_structlist_data.push_back(inner_list_data);
}
writer.fill("my_3d_structlist", {my_3d_structlist_data});
```

Further examples illustrating `struct` data types can be found in [examples/struct-example](examples/cpp/struct_example.cpp).

### Structs with Struct Fields

You may need to have a `struct` typed column which itself contains a field that is a `struct`.
Specifying this data structure in JSON follows from the above. For example,
```c++
auto file_layout = R"(
  "fields": [
    {"name": "outer_struct", "type": "struct",
                             "fields":[
                               {"name": "field0", "type": "int32"},
                               {"name": "inner_struct", "type": "struct",
                                                        "fields":[
                                                          {"name": "inner_field0", "type": "float"},
                                                          {"name": "inner_field1", "type": "int32"}
                                                        ]}
                             ]}
  ]
)"_json;
```
The above specifies a column named `outer_struct` which contains the following fields:
  * `field0` with `int32` type
  * `inner_struct` with `struct` type with the following fields:
    * `inner_field0` with type `float`
    * `inner_field1` with type `int32`

#### Filling Structs with Struct Fields

Filling the above `struct` column that has an internal `struct` field would be done as follows,
```C++
// data for the non-struct fields of the struct "outer_struct" 
int32_t field0_data = 42;
parquetwriter::struct_t outer_struct_data{field0_data};

// data for the non-struct fields of the internal struct "inner_struct"
float inner_field0_data = 42.5;
int32_t inner_field1_data = 42;
parquetwriter::struct_t inner_struct_data{inner_field0_data, inner_field1_data};

// write the data to the Parquet file
writer.fill("outer_struct", {outer_struct_data});
writer.fill("outer_struct.inner_struct", {inner_struct_data});
```

As can be seen, for each level of `struct` nesting one provides a `parquetwriter::struct_t` containing the data
for all non-`struct` fields. Internal `struct` fields are then provided their
`parquetwriter::struct_t` using the dot (`.`) notation in the call to `parquetwriter::Writer::fill`:
`<outer_struct_level>.<inner_struct_level>`.

Note that the same number of calls to `parquetwriter::Writer::fill` must be made for each of the
nested structs, otherwise there will be a mismatch in the sizes (number of rows) of columns
in the output Parquet file, which leads to an error.

Further examples illustrating `struct` data types can be found in [examples/struct-example](examples/cpp/struct_example.cpp).

## Adding File Metadata

Arbitrary metadata in the form of a JSON can be added to the output Parquet files
using the `parquet-writer` library.

This is done by first creating the metadata JSON object with whatever arbitrary
information you wish, and then calling `parquetwriter::Writer::set_metdata`,
```c++
auto metadata = R"(
  {
    "dataset_name": "example_dataset",
    "foo": "bar",
    "creation_date": "2021/09/15",
    "bar": {"faz": "baz"}
  }
)"_json;
writer.set_metdata(metadata); // can also be JSON serialized to std::string
```

The above stores the `metadata` JSON object to the output Parquet file
as an instance of `key:value` pairs. 

An example Python script (requires the [`pyarrow`](https://pypi.org/project/pyarrow/) Python module)
that extracts the metadata stored by the `parquet-writer` library
is provided in [examples/dump-metadata.py](examples/python/dump-metadata.py) and can be run as follows,
```verbatim
$ python examples/python/dump-metadata.py <file>
```
where `<file>` is a Parquet file written by `parquet-writer` to which has been added 
metadata.

Running the `dump-metadata.py` script on any of the Parquet files generated by the [examples](#examples)
looks like,
```verbatim
$ python examples/python/dump-metadata.py example_dataset/example_dataset_0000.parquet
{
    "dataset_name": "example",
    "foo": "bar",
    "n_things": 42,
    "things": {
        "foo": "bar"
    }
}
```

## Examples

The procedure of filling [basic types](#supported-data-types) using the corresponding C++ data types
can be seen in the [section above](#basic-usage).

More complete examples for how to write any of the supported
data types to a Parquet file are found in the [examples](examples/cpp)
directory:

  * [examples/basic-example](examples/cpp/basic_example.cpp): Example showing how to fill all supported data types (other than those with `struct` type)
  * [examples/struct-example](examples/cpp/struct_example.cpp): Example showing how to fill `struct` type objects of various complexities

These examples are built alongside the [build of the `parquet-writer` library](#building-the-parquet-writer-library).

## Building the `parquet-writer` Library

Below are the steps to build the `parquet-writer` shared library for your system.

<details>
  <summary>MacOS</summary>
  
  ```
  mkdir build/ && cd build/
  cmake -DARROW_PATH=/usr/local/Cellar/apache-arrow/<version>/ ..
  make
  ```
  Where `<version>` must be replaced by the specific version, e.g. `5.0.0`.
  
</details>
  
<details>
  <summary>Debian/Ubuntu</summary>
  
  ```
  mkdir build/ && cd build/
  cmake -DCMAKE_MODULE_PATH=/usr/lib/<arch>/cmake/arrow/ ..
  make
  ```
  Where `<arch>` may be something like `x86_64-linux-gnu` if present.
  
</details>
  
Upon a successful build, the shared library `parquet-writer` will be located under `build/lib`.
  
It is assumed that you have installed Apache Arrow and Apache Parquet
following the [steps below](#installing-apache-arrow-and-parquet-libraries).


## Installing Apache Arrow and Parquet Libraries

See the [official docs](https://arrow.apache.org/install/) for complete details.
Below are the tested ones:

<details>
  <summary>MacOS</summary>
  
  Via `homebrew`:
  
  ```
  brew install apache-arrow
  ```
  
  Which will install everything under `/usr/local/Cellar/apache-arrow/`.
  
</details>

<details>
  <summary>Debian/Ubuntu</summary>
  
  ```
  sudo apt update
  sudo apt install -y -V ca-certificates lsb-release wget
  wget https://apache.jfrog.io/artifactory/arrow/$(lsb_release --id --short | tr 'A-Z' 'a-z')/apache-arrow-apt-source-latest-$(lsb_release --codename --short).deb
  sudo apt install -y -V ./apache-arrow-apt-source-latest-$(lsb_release --codename --short).deb
  sudo apt update
  sudo apt install -y -V libarrow-dev
  sudo apt install -y -V libparquet-dev
  sudo apt install build-essential
  sudo apt install pkg-config
  ```
</details>
