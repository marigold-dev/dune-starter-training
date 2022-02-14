This training aims to learn how to setup a new OCaml project using Dune build system

## Setup a new project

### Create your sandbox

**Prerequisite**: [install opam](https://opam.ocaml.org/doc/Install.html) : _don't forget to run `opam init` if you install opam for the first time_

Then we can create a sandbox using opam and install dune:

```sh
opam switch create . ocaml-base-compiler.4.13.1
eval $(opam env)
opam install dune
```

### Create a dune project

[Dune](https://dune.build/) is a composable build system for OCaml projects _(and ReasonML and Coq)_. A project is a source tree, maybe containing one or more packages: a typical dune project will have a dune-project and one or more <package>.opam

So create the dune-project file with the [lang](https://dune.readthedocs.io/en/latest/dune-files.html#dune-project-1) and [name](https://dune.readthedocs.io/en/latest/dune-files.html#name) stanzas:

```sh
echo '(lang dune 2.9)\n (name caravanserai)' >> dune-project
```

You notice that `dune-project` is a manifest that use a kind of s-expression format.
It contains the version of Dune we will use and the name of the project.

> You may not be familiar with s-expression. It's just anothe data text format like json, yaml, xml or toml.
> this s-expression
>
> ```lisp
> (lang dune 2.9)
>  (name caravanserai)
> ```
>
> can be read as this equivalent json
>
> ```json
> {
>   "lang": { "dune": "2.9" },
>   "name": "caravanserai"
> }
> ```

Now we have a `dune-project` we can use it to generate our `caravanserai.opam` and describe our dependencies. We would like to add `dune` as build dependency, `ocamlformat`, `ocamlformat-rpc`, `ocaml-lsp-server` as developement dependencies, `alcotest` for testing, and `dream` as a project dependency

Since there is no notion of development dependencies with opam, we will produce 2 packages, one for production purpose and one for development purpose.

> You may prefer to have a Makefile to manage dev dependencies install, that's ok and that's how Tezos manage them. I prefer to have only one manifest to manage all my dependecies. A better option is to use esy.sh but we will not introduce esy in this first training.

Edit `dune-project`:

```lisp
(lang dune 2.9)

(name caravanserai)

(version 0.1)

(maintainers "contact@marigold.dev")

(generate_opam_files true)

(package
 (name caravanserai)
 (synopsis "Toy journey to explore Dune")
 (description "Toy journey to explore Dune")
 (depends
  (alcotest :with-test)
  (dune
   (and
    :build
    (>= 2.9)))
  (dream
   (= 1.0.0~alpha2))))

(package
 (name caravanserai-dev)
 (synopsis "A package to install dev dependencies")
 (description "THIS PACKAGE IS FOR DEVELOPMENT PURPOSE")
 (depends
  (ocamlformat
   (>= 0.20))
  (ocamlformat-rpc
   (>= 0.20))
  (ocaml-lsp-server
   (>= 1.9.1))))
```

We can then run `dune build` to generate the opam manifest, install our dependencies and then generate lockfiles:

```sh
dune build
opam install . --deps-only
opam lock .
```

By using these locked opam files, it is then possible to recover the precise build environment that was setup when they were generated. Latter one can just do `opam install . --locked`

#### Take Away

- `dune-project` describes the project and its dependencies
- `opam switch` creates a sandboxed environment for our project: we can work in an isolated environment
- `opam lock` creates a locked resolution of opam dependencies: we are sure our teammates are using the same version of the dependencies
- By having a `<package>.opam` in our directory, we have define a **[scope](https://dune.readthedocs.io/en/stable/concepts.html#scopes-1)**. Typically, any given project will define a single scope.

## Create a first executable

1. Create a `bin` directory
2. Create a `dune` file inside that add `dream` as a dependency and define an executable

```lisp
(executable
 (name caravanserai)
 (libraries dream))
```

3. Create a `caravanserai.ml` file

```ocaml
let greeting request =
  match Dream.query "name" request with
  | None -> Dream.html "Use ?name=foo to give a message to echo!"
  | Some name ->
      "Welcome at our caravanserai " ^ name ^"! Enjoy your stay."
      |> Dream.html_escape |> Dream.html

let () = greeting |> Dream.run  ~port:3000
```

```sh
# install Dream
opam install . --deps-only
```

4. Compile your exe with `dune build bin/caravanserai.exe`

This will create `_build/default/bin/caravanserai.exe`

5. Run `dune exec bin/caravanserai.exe` and access caravaner http://localhost:3000/?name=epic%20caravaner

### Format and autopromote

We have installed ocamlformat but still not run it (unless you are using format on save in your IDE)

We can run

```sh
dune build @fmt --auto-promote
```

This runs an autoformatter over the files when it builds the files: for OCaml this is OCamlformat. Then it updates the source files with the content of the formatted build files. This is the concept of [promotion](https://dune.readthedocs.io/en/latest/concepts.html?highlight=promotion#promotion)

> `dune build @fmt --auto-promote` is a must-have command for a [pre-commit hook](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)!

If you are checking that the code is properly formatted, simply do not promote the result, using `dune build @fmt`.

## Create a first library

1. Create a `lib` directory
2. Create a `dune` file inside to describe a library

```lisp
(library
 (name caravanserai)
 (public_name caravanserai.lib))
```

- `name` stanza is the name for the root module of the library. Here `Caravanserai`. This will expose the module from a file `caravanserai.ml` if there is one, or create a virtual module with all the modules in the directory as submodules.
- `public_name` is the name for the library, this is the name you will use to link this library to another library or executable

3. Create a `domain.ml` file inside, this file will contains the modelization of the business domain of our caravanserail in a [DDD styled functional architecture](https://pragprog.com/titles/swdddf/domain-modeling-made-functional/)

```ocaml
module Room : sig
  type t
end = struct
  type t =
    | Stable
    | Stall
    | Bedroom
end
```

4. Build the library with `dune build` ... Ooops it doesn't compile because of the [ERROR warning 37](https://www.mankier.com/1/ocamlc#-w)

Dune have a notion of environments. You have `dev` environment which is the default when you do `dune build` and `release` which is used when you do `dune build --release`. You can also defined your owns and call them with `--profile`. `dune build --profile=foo` will look for a `foo` environment.

To define or overide environments you may create `dune` file at the root of the project, along with `dune-project` file:

```lisp
(env
 (dev
  (flags
   (:standard -w +a -warn-error -a)))
 (release
  (flags
   (:standard -w +a -warn-error +a))))
```

`flags` stanza will pass flags to [ocamlc](https://www.mankier.com/1/ocamlc) and [ocamlopt](https://www.mankier.com/1/ocamlopt)
For exemple, here we activate all the warning but unset all errors for dev (we will have only warnings) but set all as error for release (we cannot release without fixing all warnings)

This is not a recommended configuration, it was used to illustrate the environment and how pass flags to the compiler

So remove the flags overide in the file dune file at the root.

### Exercice 1

The [env](https://dune.readthedocs.io/en/stable/dune-files.html#env) stanza can be used to define environment variable. Define an environment variable `port` with a value `8000` for dev env and use it to start our web server.

5. Fix the warnings by editing `domain.ml` to:

```ocaml
module Room = struct
  type t =
    | Stable
    | Stall
    | Bedroom

  let show = function
    | Stable -> "Stable"
    | Stall -> "Stall"
    | Bedroom -> "Bedroom"

  let make s =
    match String.lowercase_ascii s with
    | "stall" -> Some Stall
    | "bedroom" -> Some Bedroom
    | "stable" -> Some Stable
    | _ -> None
end
```

and creating file `domain.mli`:

```ocaml
module Room : sig
  type t

  val show : t -> string
  val make : string -> t option
end
```

You can know build your library!

### Exercice 2

- Use the `caravaneserai.lib` library in your executable.
- Add a new query string parameter `room` that will be used to make a `Room.t` value and display a new greeting message `"Welcome at our caravanserai [NAME]! Enjoy the [ROOM]"`

### Exercice 3

We could avoid to write our own `show` function by using [ppx_deriving](https://github.com/ocaml-ppx/ppx_deriving). Let's do it!

1. Add `ppx_deriving` as a project dependency.

> Don't forget to generate the caravanserai.opam and run opam install

2. Use the ppx to derive the `show` function

```ocaml
module Room : sig
  type t [@@deriving show]

  val make : string -> t option
end
```

```ocaml
module Room = struct
  type t =
    | Stable
    | Stall
    | Bedroom
  [@@deriving show { with_path = false }]

  let make = function
    | "stall" -> Some Stall
    | "bedroom" -> Some Bedroom
    | "stable" -> Some Stable
    | _ -> None
end
```

3. Add a [preprocessing specification](https://dune.readthedocs.io/en/stable/concepts.html#preprocessing-spec) to your `lib/dune` file to tell dune to preprocess the library with the ppx_deriving

> The ppx rewritter to use is `ppx_deriving.show`

## Add tests

1. Install test dependencies

```sh
opam install  . --deps-only --with-test
```

2. Create a `lib/test` directory with a `test_domain.ml` file which contains tests for our `lib/domain.ml`

3. Create a `dune` file:

```lisp
(tests
 (names test_domain)
 (libraries alcotest caravanserai.lib))
```

Here we introduced [tests](https://dune.readthedocs.io/en/stable/dune-files.html#tests) stanza. It ease the definition of test executables.
This will define an executable named test_domain.exe that will be executed as part of the [runtest](https://dune.readthedocs.io/en/stable/dune-files.html#alias) alias

4. We can now add some test case in `lib/test/test_domain.ml`:

```ocaml
open Caravanserai.Domain
open Alcotest

let test_stall () =
  let to_test =
    let open Room in
    make "stall" |> Option.get |> show in
  (check string) "same string" "Stall" to_test

let domain_set =
  [test_case "show Stall should return 'Stall'" `Quick test_stall]

let () = run "Domain Tests Suite" [("Test Domain", domain_set)]
```

5. Run `dune test`

This will run all the tests defined in the current directory and its children recursively. Tests can be also run with `dune build @runtest` or `dune runtest` command, all are equivalent.

Dune support watch mode, you may want to run `dune test -w`.

## Working with interfaces

Sometime it is usefull to define a module interface and use it while we cannot implement it right now.

### Modules without implementation

1. create a `lib/service.mli` file:

```ocaml
module Room : sig
  val create : string -> Domain.Room.t
  val update : Domain.Room.t -> Domain.Room.t
  val delete : Domain.Room.t -> unit
end
```

2. declare service as a module without implementation

```lisp
(library
 (name caravanserai)
 (public_name caravanserai.lib)
 (modules_without_implementation service)
 (preprocess
  (pps ppx_deriving.show)))
```

Such modules are not officially supported by the OCaml compiler, however they are commonly used.

### Virtual librairies

Virtual libraries correspond to dune’s ability to compile parameterized libraries and delay the selection of concrete implementations until linking an executable.

As a exemple we may define a `repository` virtual library:

```lisp
(library
 (name repository)
 ;; repository.mli must be present, but repository.ml must not be
 (virtual_modules repository))
```

This virtual library may be linked to any library as a regular library.

Later you may define a in memory repository:

```lisp
(library
 (name memory_repository)
 ;; repository.ml must be present, but repository.mli must not be
 (implements repository))
```

or a irmin repository

```lisp
(library
 (name irmin_repository)
 ;; repository.ml must be present, but repository.mli must not be
 (implements repository))
```

**In both case the module implementation file must be named `repository.ml`**

This may result of this folders organisation:

```
|
...
|- repository
|-- dune
|-- repository.mli
|- memory-repository
|-- dune
|-- repository.ml
|- irmin-repository
|-- dune
|__ repository.ml
```

Later an executable may select an actual implementation.

```lisp
(executable
 (name caravanserai)
 (libraries dream caravanserai.lib memory_repository))
```

## Take Away

This training is a first step to be able to create a brand new project with opam + dune and have insight to navigate in its [documentation](https://dune.readthedocs.io)

There is more advanced features you can explore to deeply understand dune 🏜️🐫
