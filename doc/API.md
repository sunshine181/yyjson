# Table of Contents

* [Read JSON](#read-json)
    * [Read JSON from string](#read-json-from-string)
    * [Read JSON from file](#read-json-from-file)
    * [Read JSON with options](#read-json-with-options)
    * [Reader flag](#reader-flag)
* Write JSON
* Build JSON



------
# Read JSON
yyjson provides 3 methods for reading JSON,<br/>
each method accepts an input of UTF-8 data or file,<br/>
returns a document if succeeds, or returns NULL if fails.

### Read JSON from string
The `dat` should be a UTF-8 string, null-terminator is not required.<br/>
The `len` is the byte length of `dat`.<br/>
The `flg` is reader flag, specify 0 if you don't need it, see [reader flag](#reader-flag) for detail.<br/>
If input is invalid, `NULL` is returned.

```c
yyjson_doc *yyjson_read(const char *dat, 
                        size_t len, 
                        yyjson_read_flag flg);
```
Sample Code:

```c
const char *str = "[1,2,3,4]";
yyjson_doc *doc = yyjson_read(str, strlen(str), 0);
if (doc) {...}
yyjson_doc_free(doc);
```

### Read JSON from file

The `path` is JSON file path.<br/>
The `flg` is reader flag, specify 0 if you don't need it, see [reader flag](#reader-flag) for details.<br/>
The `alc` is memory allocator, specify NULL if you don't need it, see [memory allocator](#memory-allocator) for details.<br/>
The `err` is a pointer to receive error message, specify NULL if you don't need it.<br/>
If input is invalid, `NULL` is returned.

```c
yyjson_doc *yyjson_read_file(const char *path,
                             yyjson_read_flag flg,
                             yyjson_alc *alc,
                             yyjson_read_err *err);
```

Sample Code:

```c
yyjson_doc *doc = yyjson_read_file("/tmp/test.json", 0, NULL, NULL);
if (doc) {...}
yyjson_doc_free(doc);
```

### Read JSON with options
The `dat` should be a UTF-8 string, you can pass a const string if you don't use `YYJSON_READ_INSITU` flag.<br/>
The `len` is the `dat`'s length in bytes.<br/>
The `flg` is reader flag, specify 0 if you don't need it, see [reader flag](#reader-flag) for details.<br/>
The `alc` is memory allocator, specify NULL if you don't need it, see [memory allocator](#memory-allocator) for details.<br/>
The `err` is a pointer to receive error message, specify NULL if you don't need it.<br/>

```c
yyjson_doc *yyjson_read_opts(char *dat, 
                             size_t len, 
                             yyjson_read_flag flg,
                             yyjson_alc *alc, 
                             yyjson_read_err *err);
```

Sample code:

```c
const char *dat = your_file.bytes;
size_t len = your_file.size;

yyjson_read_flag flg = YYJSON_READ_ALLOW_COMMENTS | YYJSON_READ_ALLOW_INF_AND_NAN;
yyjson_err err;
yyjson_doc *doc = yyjson_read_opts(dat, len, flg, NULL, &err);

if (doc) {...}
else printf("read error: %s code: %u at position: %ld\n", err.msg, err.code, err.pos);

yyjson_doc_free(doc);
```

### Reader flag
yyjson provides a set of flags for JSON reader.<br/>
You can use a single flag, or combine multiple flags with bitwise `|` operator.

●**YYJSON_READ_NOFLAG = 0**<br/>

This is the default option (RFC 8259 compliant):

- Read positive integer as `uint64_t`.
- Read negative integer as `int64_t`.
- Read floating-point number as `double` with correct rounding.
- Read integer which cannot fit in `uint64_t` or `int64_t` as `double`.
- Report error if real number is infinity.
- Report error if string contains invalid UTF-8 character or BOM.
- Report error on trailing commas, comments, `inf` and `nan` literals.

●**YYJSON_READ_INSITU**<br/>
Read the input data in-situ.<br/>
This option allows the reader to modify and use input data to store string values, which can increase reading speed slightly. The caller should hold the input data before free the document. The input data must be padded by at least 4-byte zero byte. For example: "[1,2]" should be "[1,2]\0\0\0\0", length should be 5.

Sample Code:

```c
size_t dat_len = ...;
char *buf = malloc(dat_len + 4); // create a buffer larger than (len + 4)
read_from_socket(buf, ...);
memset(buf + file_size, 0, 4); // set 4-byte padding after data

yyjson_doc *doc = yyjson_read_opts(buf, dat_len, YYJSON_READ_INSITU, NULL, NULL);
if (doc) {...}
yyjson_doc_free(doc);
free(buf); // the input dat should free after document.
```

●**YYJSON_READ_FASTFP**<br/>
Read floating-point number with a fast method.<br/>

This option can greatly increase the reading speed of long floating-point numbers,<br/>
but may get 0-2 ulp error for each number.<br/>
This flag is very useful when the precision of floating-point numbers is not important,<br/>
such as display a 3D model or play a lottie animation on mobile.

●**YYJSON_READ_STOP_WHEN_DONE**<br/>
Stop when done instead of issues an error if there's additional content after a JSON document.<br/> 
This option may used to parse small pieces of JSON in larger data, such as [NDJSON](https://en.wikipedia.org/wiki/JSON_streaming).<br/>

Code Sample:

```c
// single file with multiple json, such as:
// <[1,2,3] [4,5,6] {"a":"b"}>

size_t file_size = ...;
char *dat = malloc(file_size + 4);
your_read_file(dat, file);
memset(dat + file_size, 0, 4); // add padding
    
char *hdr = dat;
char *end = dat + file_size;
yyjson_read_flag flg = YYJSON_READ_INSITU | YYJSON_READ_STOP_WHEN_DONE;

while (true) {
    yyjson_doc *doc = yyjson_read_opts(hdr, end - hdr, flg, NULL, NULL);
    if (!doc) break;
    your_doc_process(doc);
    hdr += yyjson_doc_get_read_size(doc); // move to next position
    yyjson_doc_free(doc);
}
free(dat);
```

●**YYJSON_READ_ALLOW_TRAILING_COMMAS**<br/>
Allow single trailing comma at the end of an object or array, for example:

```
{
    "a": 1,
    "b": 2,
}

[
    "a",
    "b",
]
```

●**YYJSON_READ_ALLOW_COMMENTS**<br/>
Allow C-style single line and multiple line comments, for example:

```
{
    "name": "Harry", // single line comment
    "id": /* multiple line comment */ 123
}
```

●**YYJSON_READ_ALLOW_INF_AND_NAN**<br/>
Allow nan/inf number or literal  (case-insensitive), such as 1e999, NaN, Inf, -Infinity, for example:

```
{
    "large": 123e999,
    "nan1": NaN,
    "nan2": nan,
    "inf1:" Inf,
    "inf2": -Infinity
}
```


# Write JSON
yyjson provides 3 methods for writing JSON,<br/>
each method accepts an input of yyjson document, returns a UTF-8 string or file.

### Write JSON to string

```c
char *yyjson_write(yyjson_doc *doc, yyjson_write_flag flg, size_t *len);
char *yyjson_mut_write(yyjson_mut_doc *doc, yyjson_write_flag flg, size_t *len);
```

### Write JSON to file

```c
bool yyjson_write_file(const char *path,
                       yyjson_doc *doc,
                       yyjson_write_flag flg,
                       yyjson_alc *alc,
                       yyjson_write_err *err);
bool yyjson_mut_write_file(const char *path,
                           yyjson_mut_doc *doc,
                           yyjson_write_flag flg,
                           yyjson_alc *alc,
                           yyjson_write_err *err);
```

### Write JSON with options
```c
char *yyjson_write_opts(yyjson_doc *doc,
                        yyjson_write_flag flg,
                        yyjson_alc *alc,
                        size_t *len,
                        yyjson_write_err *err);
char *yyjson_mut_write_opts(yyjson_mut_doc *doc,
                            yyjson_write_flag flg,
                            yyjson_alc *alc,
                            size_t *len,
                            yyjson_write_err *err);
```




# Memory Allocator
yyjson use libc's `malloc()`, `realloc()` and `free()` as default allocator, but yyjson allows you to customize the memory allocator to achieve better performance or lower memory usage.

### Single allocator for multiple JSON
If you need to parse multiple small JSON, you can use a single allocator with pre-allocated buffer to avoid frequent memory allocation.

Sample code:
```c
// max data size for single JSON
size_t max_json_size = 64 * 1024;
// calculate the max memory usage for a single JSON
size_t buf_size = yyjson_read_max_memory_usage(max_json_size, 0);
// create a buffer for allocator
void *buf = malloc(buf_size);
// setup the allocator with buffer
yyjson_alc alc;
yyjson_alc_pool_init(&alc, buf, buf_size);

// read multiple JSON with single allocator
for(int i = 0, i < your_json_file_count; i++) {
    const char *your_json_file_path = ...;
    yyjson_doc *doc = yyjson_read_file(your_json_file_path, 0, &alc, NULL);
    ...
    yyjson_doc_free(doc);
}

// free the buffer
free(buf);
```

### Stack memory allocator
If the JSON is small enough, you can use stack memory only to read or write it.

Sample code:
```c
char buf[128 * 1024]; // stack buffer
yyjson_alc alc;
yyjson_alc_pool_init(&alc, buf, sizeof(buf));

yyjson_doc *doc = yyjson_read_opts(dat, len, 0, &alc, NULL);
...
yyjson_doc_free(doc); // this is optional, as the memory is on stack
```

### Use other allocator library
You can use a third-party high-performance memory allocator for yyjson,<br/>
such as [jemalloc](https://github.com/jemalloc/jemalloc), [tcmalloc](https://github.com/google/tcmalloc), [mimalloc](https://github.com/microsoft/mimalloc).

Sample code:
```c
// Use https://github.com/microsoft/mimalloc

#include <mimalloc.h>

static void *my_malloc_func(void *ctx, size_t size) {
    return mi_malloc(size);
}

static void *my_relloc_func(void *ctx, void *ptr, size_t size) {
    return mi_realloc(ptr, size);
}

static void *my_free_func(void *ctx, size_t size) {
    mi_free(ctx);
}

static const yyjson_alc MY_ALC = {
    my_malloc_func,
    my_realloc,
    my_free_func,
    NULL
};

// Read with custom allocator
yyjson_doc *doc = yyjson_doc_read_opts(dat, len, 0, &MY_ALC, NULL);
...
yyjson_doc_free(doc);

// Write with custom allocator
yyjson_alc *alc = &MY_ALC;
char *json = yyjson_doc_write(doc, 0, alc, NULL, NULL);
...
alc->free(alc->ctx, json);

```

# Thread Safe

# Mutable and Immutable
