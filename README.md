osxinj
======

Another dylib injector. Uses a bootstrapping module since `mach_inject` doesn't fully emulate library loading and crashes when loading complex modules.

- `mach_inject` was taken from `rentzsch/mach_inject`. Thanks!
- `testapp` is a sample app to inject into
- `testdylib` is a sample dylib to inject into an app
- `bootstrap` is a dylib that is initially injected to load another dylib (e.g. `testdylib`)

Released under the MIT License.

Notes
-----

- Build with scheme `BuildAll`



---------------------------------- End of original README.md --------------------------------------------------




## Brads Notes

This is a 2 part process

injector.cpp has code in the init method:
```
        Injector::Injector
```        
Which fill locate the address of a bootstrap method in a ?linked? dylib named bootstrap.dylib

It then will call the mach_inject code to inject bootstrap.dylib and call that bootstrap method from bootstrap.dylib passing any arguments 
which probably include the name of the actual dylib that is getting injected.

bootstrap.dylib looks like it just then uses dlopen - which just loads a dylib into memory via the filename of the dylib

Here are docs, by Apple, on how these dylib C library functions work -

  https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/DynamicLibraries/100-Articles/DynamicLibraryUsageGuidelines.html

The final injected dylib is then expected to have a function designated
```
        __attribute__ ((constructor))   
```        
which will be the code to run once the final dylib is in place in the target processes memory.
        
### Passing argumens into the final, injected dylib - ideas?

I do not see anything here to do that. I read a really easy idea/way to do this. Just use environment variables to pass data/arguments to the dylib constructor function.

Reading - https://theevilbit.github.io/posts/dyld_insert_libraries_dylib_injection_in_macos_osx_deep_dive/

it looks like if a constructor has parameters for argc/argv then these are just the arguments used to execute the target process.


### Passing argumens into the final, injected dylib - possible implementaiton

Actually, since I am passing a string to the bootstrap function in the bootstrap dylib (for the dylib filename to be injected via dlopen). 
Then why also can I not pass any other data?

I can then, from bootstrap.dylib, dlopen the injected dylib. BUT instead of relying on the constructor function to begin whatever injected 
code that is expected to run just use dlsym to lookup some hard coded pre-planned/pre-named init function which takes some pre-agreed
upon structured arguments.

Loading and calling the injected dylib would then just look like this SO answer: 
  https://stackoverflow.com/questions/1354537/dlsym-dlopen-with-runtime-arguments

```
        #include <dlfcn.h>

        typedef void* (*arbitrary)();
        // do not mix this with   typedef void* (*arbitrary)(void); !!!

        int main()
        {
            arbitrary my_function;
            // Introduce already loaded functions to runtime linker's space
            void* handle = dlopen(0,RTLD_NOW|RTLD_GLOBAL);
            // Load the function to our pointer, which doesn't know how many arguments there sould be
            *(void**)(&my_function) = dlsym(handle,"something");
            // Call something via my_function
            (void)  my_function("I accept a string and an integer!\n",(int)(2*2));
            return 0;
        }

```


                
                
## Getting this demo to run on OSX

Build binaries  and place in current directory

        cp testdylib.dylib /Applications/Textastic.app/Contents/MacOS/testdylib.dylib 

Run textastic in diff terminal

        sudo ./osxinj Textastic /Applications/Textastic.app/Contents/MacOS/testdylib.dyli

I also codesigned everything if still running into issues


NOTE: Ran into issues at first with snadboxing requiring the dylib to bin the Applications folder. 
Initial errors:

        Could not load patch bundle: dlopen(/Users/bbarrows/repos/injectDylibOSX2020osxinj/testdylib.dylib, 2): no suitable image found.  Did find:
                file system sandbox blocked open() of '/Users/bbarrows/repos/injectDylibOSX2020osxinj/testdylib.dylib'



