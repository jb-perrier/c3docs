# Library

The standard library is currently in development, so frequent changes will occur. Note that all std::core modules and
sub modules are implicitly imported.

## std::core::builtin

All functions and macros in this library can be used without path qualifiers.

### void panic(char* message, char *file, char *function, uint line)
Default function called when the asserts fails.

### void @swap(&a, &b)
Swap values in `a` and `b`.

```c
int a = 3;
int b = 5;
@swap(a, b);
io::printf("%d\n", a); // Prints 5
```

### varcast(variant v, $Type)

Optionally cast the value `v` to type `$Type*` on failure returns `VarCastResult.TYPE_MISMATCH`.

```c
int b;
variant a = &b;
float*! c = varcast(a, float); // Will return TYPE_MISMATCH
int*! d = varcast(a, int);     // Works!
```

### void unreachable($string = "Unreachable statement reached.")

Mark a code path as unreachable.

```c
switch (x)
{
  case 0:
    foo();
  case 1:
    bar();
  default:
    // Should never happen.
    unreachable("x should have been 0 or 1");    
}    
```

On safe mode this will throw a runtime panic when reached. For release mode the
compiler will assume this case never happens.

### bitcast(value, $Type)
Do a bitcast of a value to `$Type`, requires that the types are of the same memory size.
```c
float x = 1.0;
int y = bitcast(x, int); // y = 0x3f800000
```

### enum_by_name($Type, enum_name)
Optionally returns the enum value with the given name. `$Type` must be an enum. Returns `SearchResult.MISSING`
on failure.
```c
enum Foo { ABC, CDE, EFG }

fn void! test()
{
  Foo f = enum_by_name(Foo, "CDE")?; 
  // same as Foo f = Foo.CDE;
}
```

### void @scope(&variable; @body)

Scopes a variable:

```
int a = 3;

@scope(a)
{
    a = 4;
    a++;
};

// Prints a = 3
io::printf("a = %d\n", a, b);
```

### less, greater, less_eq, greater_eq, equals
All macros take two values and compare them. Any type implementing `Type.less` 
or `Type.compare_to` may be compared (or if the type implements `<`). Types 
implementing `Type.equals` may use `equals` even if neither `less` nor `compare_to`
are implemented.

### Faults

- `IteratorResult` returned when reaching the end of an iterator.
- `SearchResult` used when a search fails.
- `VarCastResult` when a variant cast fails.

## std::core::env

### Constants
- `OS_TYPE` the OS type compiled for.
- `COMPILER_OPT_LEVEL` the optimization level used.
- `I128_SUPPORT` true if int128 support is available.
- `COMPILER_SAFE_MODE` true if compiled with safety checks.

## std::core::mem

### malloc, malloc_checked, malloc_aligned

Allocate the given number of bytes. `malloc` will panic on out of memory, 
whereas `malloc_checked` and `malloc_aligned` returns an optional value.
`malloc_aligned` adds an alignment, which must be a power of 2. Any pointer
allocated using `malloc_aligned` must be freed using `free_aligned` rather
the normal `free` or memory corruption may result.

```c
char* data = malloc(8);
char*! data2 = malloc_checked(8);
int[<16>]*! data3 = malloc_aligned(16 * int.sizeof), 128);  
```

### calloc, calloc_checked, calloc_aligned

Identical to `malloc`, except the data is guaranteed to be zeroed out.

### relloc, relloc_checked, realloc_aligned

Resizes memory allocated using `malloc` or `calloc`. Any extra data is 
guaranteed to be zeroed out. `realloc_aligned` can only be used with
pointers created using `calloc_aligned` or `alloc_aligned`.

### free, free_aligned

Frees memory allocated using `malloc` or `calloc`.

### char[] alloc_bytes(usize bytes)

Similar to malloc, but returns the data as a subarray. Will panic on out of memory.

### alloc($Type)

Will allocate enough memory for exactly the type provided:

```c
Foo* f = mem::alloc(Foo);
```

### @scoped(Allocator* allocator; @body())

Swaps the current memory allocator for the duration of the call.

```c
DynamicArenaAllocator dynamic_arena;
dynamic_arena.init(1024);
mem::@scoped(&dynamic_arena) 
{
    // This allocation uses the dynamic arena 
    Foo* f = mem::alloc(Foo);
};
// Release any dynamic arena memory.
dynamic_arena.destroy();    

```

### @tscoped(; @body())

Same as @scoped, but uses the temporary allocator rather than any
arbitrary allocator.

### void* tmalloc(usize size, usize alignment = 0)

Allocates memory using the temporary allocator. Panic on failure.

### void* tcalloc(usize size, usize alignment = 0)

Same as `tmalloc` but clears the memory.

### void* trealloc(void* ptr, usize size, usize alignment = 0)

`realloc` but on memory received using `tcalloc` or `tmalloc`.

### talloc($Type)

Like `alloc` but using the temporary allocator.

### void @pool(;@body)

Opens a temporary memory scope.

```c
@poo() 
{
    // This allocation uses the dynamic arena 
    Foo* f = talloc(Foo);
};

```


### @volatile_load(&x)

Returns the value in `x` using a volatile load.

```c
// Both loads will always happen:
int y = @volatile_load(my_global);
y = @volatile_load(my_global);
```


### @volatile_store(&x, y)

Store the value `y` in `x` using a volatile store.

```c
// Both stores will always happen:
@volatile_store(y, 1);
@volatile_store(y, 1);
```
### usize aligned_offset(usize offset, usize alignment)

Returns an aligned size based on the current offset. The alignment
must be a power of two. E.g. `mem::aligned_offset(17, 8)` would return `24`

### usize aligned_pointer(void* ptr, usize alignment)

Returns a pointer aligned to the given alignment, using `aligned_offset`.

### bool ptr_is_aligned(void* ptr, usize alignment)

Return true if the pointer is aligned, false otherwise.

### void copy(void* dst, void* src, usize len, usize $dst_align = 0, usize $src_align = 0, bool $is_volatile = false)

Copies bytes from one pointer to another. It may optionally be set as volatile,
in which case the copy may not be optimized away. Furthermore the source
and destination alignment may be used.

```c

Foo* f = talloc(data_size);
mem::copy(f, slice.ptr, size); 
```

### void set(void* dst, char val, usize len, usize $dst_align = 0, bool $is_volatile = false)

Sets bytes to a value. This operation may be aligned and/or volatile. See the `copy` method.

### void clear(void* dst, usize len, usize $dst_align = 0, bool $is_volatile = false)

Sets bytes to zero. This operation may be aligned and/or volatile. See the `copy` method.

### @clone(&value)

Makes a shallow copy of a value using the regular allocator.

```c
Foo f = ...

return @clone(f); 

```

### @tclone(&value)

Same as `@clone` but uses the temporary allocator.

## std::core::mem::array

### alloc($Type, usize elements)

Allocate a slice with `elements` number of elements, returning
a subarray of the given length. Elements are not initialized.
`array::talloc` variant uses the temporary allocator.

```c
int[] ints = array::alloc(int, 100);
```

### make($Type, usize elements)

Like `array::alloc` but all elements are cleared. `tmake` variant uses
the temporary allocator.

## std::core::types

### bool is_comparable($Type)

Return true if the type can be used with comparison operators.

### bool is_equatable_value(value)

Return `true` if the value can be compared using the `equals` macro.

### bool is_equatable_value(value)

Return `true` if the value can be compared using the comparison macros.

### kind_is_int(TypeKind kind)
### variant_to_int(variant v, $Type)

Returns an optional value of `$Type` if the variant value losslessly
may be converted into the given type. Returns a `ConversionResult` otherwise.

```c
variant v = &&128;
short y = variant_to_int(v, short)!!; // Works 
ichar z = variant_to_int(v, ichar)!!; // Panics VALUE_OUT_OF_RANGE
```

## std::core::string::conv

### usize! char32_to_utf8(Char32 c, char* output, usize available)
Convert a UTF32 codepoint to an UTF8 buffer. `size` has the number of
writable bytes left. It returns the number of bytes used, or 
`UnicodeResult.CONVERSION_FAILED` if the buffer is too small.

### void char32_to_utf16_unsafe(Char32 c, Char16** output)
Convert a UTF32 codepoint to an UTF16 buffer without bounds checking,
moving the output pointer 1 or 2 steps.

## std::io

### usize! printf(char[] format, args...) @maydiscard
Regular printf functionality: `%s`, `%x`, `%d`, `%f` and `%p` are supported.
Will also print enums and vectors.

### usize! String.printf(String* str, char[] format, args...) @maydiscard
Same as printf but on strings.

### usize! File.printf(File file, char[] format, args...) @maydiscard
Same as printf but on files.

### void! File.open(File* file, char[] filename, char[] mode)
Open a file with the given file name with the given mode (r, w etc)

### void! File.seek(File *file, long offset, Seek seekMode = Seek.SET)
Seek in a file. Based on the libc function.

### void! File.close(File *file) @inline
Close a file, based on the libc function.

### bool File.eof(File* file) @inline
True if EOF has been reached. Based on the libc function.

### void! File.putc(File *file, char c)
Write a single byte to a file. See the libc function.

### usize File.read(File* file, void* buffer, usize items, usize element_size = 1)
Read into a buffer, based on the libc function.

### usize File.write(File* file, void* buffer, usize items, usize element_size = 1)
Write to a buffer, based on the libc function.

### stdout(), stdin(), stderr()
Return stdout, stdin and stderr respectively.

## std::array::list<Type>

Generic list module, elements are of `Type`.

```c
import std::array::list;
define MyIntList = List<int>;

...

MyIntList list;
list.push(123);
list.free();
```

### List.push(List *list, Type element), append(...)
Append a single value to the list.

### Type List.pop(List* list)
Removes and returns the last entry in the list.

### Type List.pop_first(List *list)
Removes the first entry in the list.

### void List.remove_at(List *list, usize index)
Removes the entry at `index`.

### void List.insert_at(List *list, usize index, Type type)
Inserts a value at `index`.

### void List.push_front(List *list, Type type)
Inserts a value to the front of the list.

### void List.remove_last(List* list)
Remove the last value of the list.

### void List.remove_first(List *list)
Remove the first element in the list.

### Type* List.first(List* list)
Return the first element in the list if available.

### Type* List.last(List *list)
Return the last element in the list if available.

### List.is_empty(List *list)
Return `true` if the list is empty.

### usize List.len(List *list)
Return the number of elements in the list.

### Type List.get(List *list, usize index)
Return the value at `index`.

### void List.free(List *list)
Free all memory associated with this list.

### void List.swap(List *list, usize i, usize j)
Swap two elements in the list.

