### Passing arguments by reference ###

* Primitives, `std::string` and sometimes small classed (like `osg::Vec3` can be passed by value).
* Objects of class type that are not intended to be modified inside a function must be passed by const reference.
* Output parameters must therefore be passed as pointers.
* Output parameters must have prefix `out_`.

Bad:

    bool read_input(big_class obj, int& out_x);  // obj is copied
    //...
    int x = 0;
    bool ok = read_input(obj, x);    // Not clear from the call site that x is modified inside read_input

Good:

    bool read_input(const big_class& obj, int* out_x);
    //...
    int x = 0;
    bool ok = read_input(obj. &x);
    


### #include in header files ###
* Each header file must have `#pragma once` include guard.  
* A header file must not include any other header file that is not essential for the header file itself.
* Order of includes in a .cpp file should be:
    * the corresponding .h file
    * standard c/c++ library
* If it is not obvious from the code what is the purpose of an `#include` it should be explained, for example:

        #include <string>
        #include <vector>
        #define _USE_MATH_DEFINED       // for math constants: PI, PI/2, etc.
        #include <math.h>
        #include <conio.h>              // for _kbhit()

* For pointer and reference types it is enough to provide a *forward declaration* instead of including the header that contains the definition of the whole type.

Bad:

    // foo.h
    #pragma once
    #include <windows.h>    // windows.h is not required in this
                            // header file, it is required only
                            // for the implementation

                            
    #include "cool_cls.h"   // cool_cls.h is not required here
                            // because a forward declaration
                            // would suffice
                            
    void my_fancy_sleep(int msecs_to_sleep, cool_cls* obj);
    
Good:

    // foo.h
    #pragma once
    
    class cool_cls;          // forward declaration
    
    void my_sleep(int msecs_to_sleep, cool_cls* obj);
    
    // ----------
    // foo.cpp
    #include "foo.h"
    #include <windows.h>        // for Sleep
    
    #include "cool_cls.h"
    
    void my_fancy_sleep(int msecs_to_sleep, cool_cls* obj)
    {
        obj->bar();
        Sleep(msecs_to_sleep);
    }

Bad:

    // foo.cpp
    #include <map>           // map gets included before foo.h
    #include "foo.h"
    // ...
    // ... more of foo.cpp
    
    // -----------    
    // foo.h
    #pragma once
    #include <string>
    
    class foo
    {
        std::map<int,std::string> bar;           // foo.cpp passes compilation thanks to
    }                                            // the misplaced #include <map> directive

    // -----------
    // baz.cpp
    // compilation error because std::map is not defined
    
    #include "foo.h"
    // ...
    // ... more of baz.cpp
    
Good:

    // foo.cpp
    #include "foo.h"        // If foo.cpp compiles successfully then it is guaranteed
                            // that the foo.h has all declarations it requires
                            // because it is the first header to be #included in foo.cpp

### #if 0 #endif vs. comment-out ###
* If a piece of code that needs to be temporary disabled is published via version control system, it must be surrounded with `#if 0` and `# endif`.

Commented out code might be error-prone, especially when a code that contains commented out code in `/*...*`/ style is commented out once more.

### Null pointer ###
* In C++11 use `nullptr` to designate the null value of pointer.
* In C++03 use `0` (zero) to designate the null value of pointer.
* Do not misuse null pointer check as a check for an invalid pointer, especially when the pointer is never set to null
* If the pointer is sometimes set to null, but must not be null in a certain place in the code, `assert` it instead of silently skipping statements.

In C++, `0` value of a pointer is not a synonym for an "invalid pointer". In fact, any address that does not belong to the process address space is invalid and is much more troublesome than the null pointer. ~Using `0` for null pointer is standard C++ and it clarifies the fact that the pointer is nothing more than a type-safe address in memory.~
**EDIT** (2018): this is questionable, anyway, use `nullptr`.

Bad:

    void foo(bar* x, int* out_y)
    {
        if(x == NULL || out_y == NULL)
            return;
        
        out_y = x->get_some();
    }
    
    bar* x = 0;
    // ...
    // ...
    x = new bar();
    //...
    int y = 0;
    foo(x, &y);         // out_y is always bound, never a null pointer
                        // no need to check it
                        
Good:

    void foo(bar* x, int* out_y)
    {
        assert(x != 0 && "x is expected to be initialized at this point");
        out_y = x->get_some();
    }
    
### Assertions ###
* Use `assert` to do internal sanity checks about invariants and preconditions.
* Do not use `assert` to notify about external errors, including i/o errors, files, etc.
* In Visual Studio, if `NDEBUG` is defined (e.g in the Release build configuration), the `assert` has no effect. It is recommended to define a custom assert macro that will fallback to, for example, silently writing to log file if an assertion has failed.

### Exceptions and error codes ###
* Avoid using exceptions if possible.
* Avoid using error codes if possible.
* Avoid mixing exception based flow with error codes based flow.

That means, write code that has as little exceptional conditions as possible: use negative/zero/float infinity values, use defaults, use empty collections, return false or even state that the return value is undefined if it is appropriate. Do **not** overuse exceptions/error codes for scenarios/conditions that are not exceptional.
If you have to, prefer using exceptions over error codes.

* All exceptions must inherit (possibly indirectly) from `std::exception`.
* Do not catch an exception in a code that is not a suitable place to handle it, in particular, do not catch an exception in multiple sites if all the handlers only log more or the same diagnostic and then rethrow it.
* Do not add "general" try-catch guards for exceptions that are not known to be thrown by the surrounded code, except for global exception silencer in production environment only.

Bad:

    class config
    {
        bool has_key(const char* key);
        int get(const char* key)
        {
            if(!has_key(key))
                throw std::exception("key not found");
            // ...
        }
    };
    
    // ...
    // ...
    
    config cfg;
    int x = 5;      // default value of x
    try
    {
        x = cfg.get("xx");
    }
    catch(std::exception& e)
    {
        LOG_WARN("xx not found, fallback to default");
    }
    
Good:

    class config
    {
        bool has_key(const char* key);
        int get_or_default(const char* key, int default_ = 0)
        {
            if(!has_key(key))
                return default_;        // avoids exceptional flow because anyway
            // ...                      // the client code has nothing to do but fallback
        }                               // to a default value
    };
    
    // ...
    // ...
    config cfg;
    int x = cfg.get_or_default("xx", 5);
    // optional: write a message to log, note that it is debug level severity only, not warning
    if(!cfg.has_key("xx"))
        LOG_DEBUG("xx not found, fallback to default value=5");
    

    
### To be expanded:  
[ ] classes, namespaces, variables in snake case  
[ ] consts, enums, macros in snake or all caps case  
[ ] template args should be camelcase  
[ ] class privarte/protected fields end with underscore  
[ ] no hungarian  
[ ] no inverted comparisons (e.g `if (5 == x)`)   
[ ] use standard names/pre/suffixes (curr, it, j, count, is_, has_, was_,should_, num_, _count, init_, xcurr, ycurr, xxcurr)  
[ ] file level namespaces do not inc. indentation level  
[ ] struct public fields are not suffixed with underscore  
[ ] early exit, no redundant else blocks  
[ ] prefer omitting curly braces in if, while etc. where appropriate  
[ ] source code filenames in snake case  
[ ] avoid using singletons  
[ ] avoid using static/global variables if they have non-trivial ctor or dtor  
[ ] avoid using virtual dispatch in constructors  
[ ] use explicit on ctors  
[ ] disable unnecessary copy-assignment operators  
[ ] use struct for passive data, functors, wrappers etc. only  
[ ] use set_x, x for set/get. if x is already bound fallback to get_x. also update_x, read_x could be appropriate for complex operatons.  
[ ] if get/set is both trivial and public consider making the field public and remove getter and setter  
[ ] line length < 81  
[ ] curly braces either in the same column or in the same line  
[ ] no trailing whitespace  
[ ] use spaces, not tabs
