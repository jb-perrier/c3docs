# C Interoperability

C3 is C ABI compatible. That means you can call C from C3, and call C3 from C without having to
do anything special. As a quick way to call C, you can simply declare the function as a 
C3 function but with `extern` in front of it. As long as the function is linked, it will work:

    extern fn void puts(char*); // C "puts"

    fn void main()
    {
        // This will call the "puts"
        // function in the standard c lib.
        puts("Hello, world!"); 
    }

While C3 functions are available from C using their external name, it's often useful to
define an external name using `@extname` to match C usage.


    module foo;
    fn int square(int x)
    {
        return x * x;
    }

    fn int square2(int x) @extname("square")
    {
        return x * x;
    }

Calling from C:

    extern int square(int);
    int foo_square(int) __attribute__ ((weak, alias ("foo.square")));

    void test()
    {
        // This would call square2
        printf("%d\n", square(11));

        // This would call square
        printf("%d\n", foo_square(11));
    }

### Gotchas

- Bitstructs will be seen as its underlying type from C. 
- C3 cannot use C bit fields
- C assumes the enum size is `CInt`
- C3 uses fixed integer sizes, this means that `int` and `CInt` does not need to be the same.
- Passing arrays by value like in C3 must be represented as passing a struct containing the array.
- Atomic types is not supported by C3.
- Volatile and const have no representation in C3.
