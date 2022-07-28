


Trampolines are small pieces of code that, when called, perform some intermediary operations and then jump to the actual target destination. When you call imp_implementationWithBlock(), a function pointer to a trampoline is returned; it's this trampoline's responsibility to modify the function arguments and then jump to the actual code corresponding to the block's implementation.

