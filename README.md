# Introduction

As of Emacs version `25.1` Emacs is able to load dynamic modules. And as of
version `1.5` Go supports building packages as dynamice shared libraries.

This means in theory we can write Emacs elisp modules in pure Go.

We'll create a minimal Emacs C module, and then a Emacs module in Go.


# Minimal Emacs module in C

We'll start by writing a basic Emacs module in C.

First we'll create a top level directory and then `cd` into that directory.

    mkdir -pv ~/src/emacs-module
    cd ~/src/emacs-module

From here on any commands we run, will be from our top-level directory. in this
case `~/src/emacs-module`
To compile a Emacs modules we need the `emacs-module.h` header file. The header
file is found in the `src` directory of the Emacs source tarball. It's possible your
OS provides this already. However this guide is OS agnostic and we'll assume
that the header file has not been installed.

Download the emacs source tarball.

    wget -c https://mirrors.kernel.org/gnu/emacs/emacs-25.2.tar.gz

And then extract it.

    tar xf emacs-25.2.tar.gz

Create the src directory

    mkdir src


## src/cmodule.c

Edit `src/cmodule.c`

    #include <emacs-module.h>
    #include <stdio.h>

    #define CMOD_VERSION  "0.1"

    int plugin_is_GPL_compatible;

    static emacs_value
    Fcmodule_glibc_version (emacs_env *env, int nargs, emacs_value args[], void *data)
    {
    emacs_value message = env->intern(env, "message");
    emacs_value *string = env->make_string(env, CMOD_VERSION, 3);
    env->funcall(env, message, 1, CMOD_VERSION);
    return string;
    }

    extern int emacs_module_init(struct emacs_runtime *ert)
    {

    emacs_env *env = ert->get_environment(ert);

    emacs_value Qfeat = env->intern (env, "cmodule");
    emacs_value Qprovide = env->intern (env, "provide");

    emacs_value pargs[] = { Qfeat };
    env->funcall (env, Qprovide, 1, pargs);

    printf("%s\n", CMOD_VERSION);
      return 0;
    }

First we include `emacs-module.h` header file.
declartion that we'll need.

    #include <emacs-module.h>
    #include <stdio.h>

    #define CMOD_VERSION  "0.1"

Emacs requires `plugin_is_GPL_compatible` symbol, if not declared the module
will not be loaded.

    int plugin_is_GPL_compatible;

This will be the function that be called by Emacs once we have registered it.
Then function

    static emacs_value
    Fcmodule_glibc_version (emacs_env *env, int nargs, emacs_value args[], void *data)
    {
    emacs_value message = env->intern(env, "message");
    emacs_value *string = env->make_string(env, CMOD_VERSION, 3);
    env->funcall(env, message, 1, CMOD_VERSION);
    return string;
    }

This function is the entry point of our module and is run when the module is first
loaded by Emacs.

    extern int emacs_module_init(struct emacs_runtime *ert)
    {

Next we need to provision the module. We'll call the elisp `(provide)` function
through the C interface. If not Emacs will error with feature not provided.

First we get the Emacs environment from the Emacs run-time.

    emacs_env *env = ert->get_environment(ert);

Then we convert our feature string into a qouted lisp symbol. The nameing is important
our module is `cmodule.so` so the feature symbol **must** be named `cmodule`,
anything else will not work. Then we get a quoted symbol for the elisp provide function.

    emacs_value Qfeat = env->intern (env, "cmodule");
    emacs_value Qprovide = env->intern (env, "provide");

We then create an arguements array containing our feature symbol. And have the
Emacs environment call the provides function with our agruement. We also let
funcall know that we are passing one argument.

    emacs_value pargs[] = { Qfeat };
    env->funcall (env, Qprovide, 1, pargs);

      printf("%s\n", CMOD_VERSION);
      return 0;
    }


## Compiling the module

Create a lib directory to hold our shared libraries.

    mkdir lib

Now compile `cmodule.c` as a shared C library.

    gcc -I emacs-25.2/src -fPIC -shared src/cmodule.c -o lib/cmodule.so


## Using our C module with Emacs.

To test our module we'll start Emacs in batch mode then call our custom function.

    emacs -Q -L ./lib -batch --eval "(require 'cmodule)"

    0.1

<script src="https://cdnjs.cloudflare.com/ajax/libs/materialize/0.100.2/js/materialize.min.js"></script>
