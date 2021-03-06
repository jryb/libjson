# Introduction

libjson is a small C library and small codebase that packs an efficient parser
and a configurable printer. libjson is covered by the LGPLv2 license, or at
your option the LGPLv3 license.

Here's the feature of libjson:

## Interruptible parser

Get the JSON data to the parser any way you want; by appending char by char,
or string chunks, the input reading is completely left to the caller.

## No object model integrated

 easy integration with any model by the means of a simple callback.

## Small codebase

 handcoded parser and efficient factorisation make the code smalls.

## Fast

use efficient code, and small parsing tables to not do any extra work and
remains as fast and efficient as possible.

## Full JSON support

tested through a small and precise testsuite.

## No native conversion

callback only string of data and leave the actual representation of data to the caller

## Security

* Supports putting limits on nesting level. security against DoS over very deep data.
* Supports putting limits on data (string/int/float) size. security against DoS over very large data object.

## Comments

Optionally support YAML/python comments and C comments.

## Allocation

Supports projects-specific allocation functions to integrate completely with projects

## Utility program

jsonlint utility provided with the library to verify, or reformat json stream.
also useful as example on how to use the library.

# Design

## Interruptible

the interruptible parser permits the user to incrementally parse data. each
time new data is passed to the parser, the parser potentially emits call back
to the caller to give a found value, and every time update its internal state
machine. this state machine will returns an error when the data passed doesn't
end up beeing a JSON compliant stream.

Because of the interruptibility of the parser, the caller is in charge of
feeding data to the parser, which give the flexibility to the caller to channel
the data to the parser in its own ways. for example, embedded stream of json in
another stream, can dynamically and without extra allocation, be passed to the
parser. this effectively decoupled the parser from the stream of data.

This also reduce the API related to parser's input to only one function,
instead of specific functions to parse a string in memory, a C FILE handler, a
unix file descriptor, etc.

## No Object Model

There's no object model embedded in the library. This way the object model is
left to the caller wants and needs. like a SAX API, the parser callback on
opening and closing structures, and also for every data values.

The caller is free to represent a json object as a GLIB hashtable, an handmade
chained list, etc. The same is true for any JSON types. This cut down the usual
rewritting of the library object tree to the native object tree after parsing
that takes memory and cpu time.

## Callback without native conversion

Float and int are passed to the caller directly as a string. Since JSON integer
and float are not limited to a specific size, this is up to the caller to
specify how to read the string. for example one implementation can choose to
represent the json int as very big int (GMP), but some other implementation can
choose 64 bits int and truncate data.

If the callback return a non null value, the parser will return failure, so
that you can stop the parsing.

## Security aspects

The design of the library is to emphasis security. the event based parser, the
carefully secure crafted code, and the optional limits in place means that the
parser can be put on the front line and will behave in front of DoS and huge
data.

The parser supports maximum size in the data buffer for string/float/int and
supports a maximum nesting level.

If the data limit is set to 0, the buffer will double when necessary.

If the nestimg limit is set to 0, the stack will double when necessary.

All string data are passed to the caller with an explicit size; NULL bytes can
be handled properly.

# about the API

The libjson API is very simple.

## function returns

every functions returns a int that is either 0 for success, or a `json_error` if different from 0.

## Callbacks object

all callback takes an optional `void *` to be able to pass a pointer to anything to the callback function.

## No assumption

There's no assumption about how a file is supposed to be read, or how we suppose to allocate memory.

## libjson structure

All libjson structure are fully available for the library user so that they can
also be allocated on the function stack for pratical purpose.

# Parsing API
 
## Parsing Context

The first thing you need to parse a document is a parsing context. Every
parsing function take a context as a first argument, and this is to keep the
parsing state machine and the event callback mechanism. It also contains
optional user defined memory functions and the parser configuration (limits,
comments allowed).

The following example initializes a new parser context:

* a NULL config which means the default parser config.
* `my_callback` a function that will be called each time there's a JSON event.
* `my_callback_data` a `void *` value that will be passed to `my_callback` every times it's called.

```C
json_parser parser;

if (json_parser_init(&parser, NULL, my_callback, my_callback_data)) {
	fprintf(stderr, "something wrong happened during init\n");
}
```

Remember that's the parser need to allocate some data, so when the parser
context is not used anymore, it needs to free with the appropriate function:

```C
json_parser_free(&parser);
```

## Parsing Data

The only thing left is feeding data into the parser. This is done with the
`json_parser_string` function which takes a string data and a length, and an
optional offset pointer. This function is completly incremental, which means
you don't have to parse all your document at once.

You're in charge of passing the data, from your file, your socket, your pipe,
your in-memory string to the parsing function.

The following example take a in-memory string and pass it to `json_parser_string`
1 characters by 1 characters:

```C
char my_json_string[] = "{ \"key\": 123 }";

for (i = 0; i < strlen(my_json_string); i += 1)  {
	ret = json_parser_string(&parser, my_json_string + i, 1, NULL);
	if (ret) {
		/* error happened : print a message or something */
		break;
	}
```

or 4 characters by 4 characters:

```C
char my_json_string[] = "{ \"key\": 123 }";

for (i = 0; i < strlen(my_json_string); i += 4)  {
	ret = json_parser_string(&parser, my_json_string + i, 4, NULL);
	if (ret) {
		/* error happened : print a message or something */
		break;
	}
```

or from a file reading 1024-bytes blocks at the same time:

```C
int fd, len, ret;
char block[1024];

fd = open(file, ...);
while ((len = read(fd, block, 1024)) > 0) {
	ret = json_parser_string(&parser, block, len, NULL);
	if (ret) {
		/* error happened : print a message or something */
		break;
	}
}
```

## Parsing events

Each time the function `json_parser_string` function is called with data, the
callback registered at context init time might be called.

There's a callback each time the parser has processed a JSON atom.

a JSON atom can be:

* `JSON_OBJECT_BEGIN` : opening a new object
* `JSON_OBJECT_END` : current object is closing
* `JSON_ARRAY_BEGIN` : opening a new array
* `JSON_ARRAY_END` : current array is closing
* `JSON_INT` : a JSON int has been parsed
* `JSON_FLOAT` : a JSON float has been parsed
* `JSON_STRING` : a JSON string has been parsed
* `JSON_KEY` : a JSON key has been parsed
* `JSON_TRUE` : a JSON true constant has been parsed
* `JSON_FALSE` : a JSON false constant has been parsed
* `JSON_NULL` : a JSON null constant has been parsed

A callback prototype looks like the following:

* `userdata`: is the callback object registered at context init time.
* `type`: is the type of the atom parsed.
* `data`: is a optional string that contained the data associated with this atom.
* `length`: is the length in byte of the data associated with the atom.

```C
int my_callback(void *userdata, int type, const char *data, uint32_t length)
```

the following example is a full callback function that just print the atom received.
as a callback object it can support an optional FILE * to print to, otherwise it will
use stdout.

```C
int my_callback(void *userdata, int type, const char *data, uint32_t length)
{
	FILE *output = (userdata) ? userdata : stdout;
	switch (type) {
	case JSON_OBJECT_BEGIN:
	case JSON_ARRAY_BEGIN:
		fprintf(output, "entering %s\n", (type == JSON_ARRAY_BEGIN) ? "array" : "object");
		break;
	case JSON_OBJECT_END:
	case JSON_ARRAY_END:
		fprintf(output, "leaving %s\n", (type == JSON_ARRAY_END) ? "array" : "object");
		break;
	case JSON_KEY:
	case JSON_STRING:
	case JSON_INT:
	case JSON_FLOAT:
		fprintf(output, "value %*s\n", length, data);
		break;
	case JSON_NULL:
		fprintf(output, "constant null\n"); break;
	case JSON_TRUE:
		fprintf(output, "constant true\n"); break;
	case JSON_FALSE:
		fprintf(output, "constant false\n"); break;
	}
}
```

## Parser configuration

Parser configuration can be set when initializing the parsing context. this is done by
passing a non-NULL pointer to a valid `json_config`.

The configuration structure support 7 differents variables, which can be group
in 3 categories:

* user defined memory functions.
* security.
* optional extensions.

### User defined memory function

The library user can choose to redefine its own allocation functions (realloc
and calloc), in this case the parser will allocate using those functions. this
is controlled by `user_calloc` and `user_realloc`.

### Security

there's 2 security settings available: `max_nesting` and `max_data`.

`max_data` control the size of data allowed, which control the maximum size of
int, string, float, and keys in bytes. this is directly connected to the
size of the data buffer. if set to 0, the buffer will try to grow each time
it's necessary.

`max_nesting` controls the number of nested structures allowed by the parser.
each new nesting increases the parser memory use by 1 byte (for 4096 nested
structures, you need 4K of memory).

For security purpose, if the parser is directly connected to a network stream,
setting those variables, is strongly recommended.

### Comments

You can enable C comments and enable YAML comments arbitrarly from each other.

`allow_c_comment` will enable C comment: starting at /*, and finishing at */

`allow_yaml_comment` will enable YAML/python comment: starting at # and finishing
at the end of line.

comments cannot be nested.

```
{
	# this is a YAML comment
	"key": /* this is a C comment */ 123,
}
```

# Printing API

## Printing context

first, the printing context is an object that is passed to all printing
functions. The main part of the printing context is the printing callback.
There's also some state about indentation level and pretty printing state.

The following example initializes a new printing context:

* a NULL config which means the default parser config.
* `my_callback` a function that will be called each time there's a JSON event.
* `my_callback_data` a `void *` value that will be passed to `my_callback` every times it's called.

```C
json_printer print;

if (json_print_init(&print, my_callback, my_callback_data)) {
	fprintf(stderr, "something wrong happened during init\n");
}
```

The printing context need to free after use using the following:

```C
json_print_free(&print);
```

## Printing JSON

You can choose between pretty printing and raw printing.

raw printing is usually associated for transfering or storing JSON data: it's
terse for common human usage. the function `json_print_raw` is associated with
raw printing

pretty printing is usually targeted for human processing and contains spaces
and line return. the function `json_print_pretty` is associated with pretty
printing

Both functions can be interchanged as they have the same prototype.

the following example print an object with one key "a" with a value of 2505:

```C
json_print_pretty(&print, JSON_OBJECT_BEGIN, NULL, 0);
json_print_pretty(&print, JSON_KEY, "a", 1);
json_print_pretty(&print, JSON_INT, "2505", 4);
json_print_pretty(&print, JSON_OBJECT_END, NULL, 0);
```

## Printing data

The printing API only transform a JSON atom into the equivalent string; the
callback is called one or multiples time at each atom, so that the user
can choose what to do with the marshalled data.

The API is not in charged of allocating a in-memory string or store the
marshalled data in a file, however doing anything similar is very easy.

The following example printing callback would write the previous structure to
disk. The file descriptor is passed through the callback object as void \*, and
need to be valid.

```C
int my_printer_callback(void *userdata, const char *s, uint32_t length)
{
	int fd = (int) userdata;

	write(fd, s, lenght);
	return 0;
}
```

## Printing Multiple JSON atoms

For conveniance, there's a `json_print_args` API function that takes multiples JSON atoms
and repeatly call the associated printing function (raw or pretty). the
function parameters need to be terminated by a -1 atom to indicate the end of arguments.

Here's an example on how to print the same object as previous example but with one call:

```C
json_print_args(&print, json_print_pretty,
                JSON_OBJECT_BEGIN,
                    JSON_KEY, "a", 1,
                    JSON_INT, "2505", 4,
                JSON_OBJECT_END,
                -1);
```

# DOM parsing helper

For convenience, there's some helpers function that permits constructing JSON DOM tree.
they are built on top of the event based parsing API described earlier.

## DOM parsing context

a DOM parsing context hold the state of the DOM building. it also hold
3 differents callbacks to create the JSON values in the tree.

the first callback, `create_structure`, is called when a new object or array
is suppose to be created. the callback can choose the representation of the data, but
need to returns a single void * to represent this new structure.

the second callback, `create_data`, is called for each value that is not
an object or an array. it need returning a void * to represent this value.

the last callback, `append`, is called each time we need to append a value to an
existing structure (array/object). if called with a non-NULL key, then the
first argument of the function represent an object, otherwise an array.

the following example hook into a hypothetical framework with array and object
special method:

```C
void *tree_create_structure(int nesting, int is_object)
{
	if (is_object)
		return new_object();
	else
		return new_array();
}

void *tree_create_data(int type, const char *data, uint32_t length)
{
	switch (type) {
	case JSON_STRING:
	case JSON_INT:
	case JSON_FLOAT:
		return new_json_value(type, data, length);
	case JSON_NULL:
	case JSON_TRUE:
	case JSON_FALSE:
		return new_json_const(type);
	}
}

int tree_append(void *structure, char *key, uint32_t key_length, void *obj)
{
	if (key != NULL) {
		my_object *object = structure;
		append_object(object, key, key_length, obj);
	} else {
		my_array *array = structure;
		append_array(array, obj);
	}
}

```

## Hooking into the event parser

the following example hooks the DOM helper parser into the event parser:

```C
json_parser_dom helper;
json_parser parser;

json_parser_dom_init(&helper, create_structure, create_data, append);
json_parser_init(&parser, json_parser_dom_callback, &helper);
```

# JSONlint utility

JSONlint is a small utility using libjson. it's able to verify and reformat JSON file.

the following will reformat input.json file and create a file output.json:

```
jsonlint --format input.json -o output.json
```
