# Stanza Wrapper for the Z3 Solver Library

This project attempts to wrap Z3's solver functionality and provide a
tool for solving and optimizating over domains.

## Example

From `tests/Solver.stanza`:

```
  val cfg = Config()
  val ctx = Context(cfg)

  val s = Solver(ctx)
  val [R1, R2] = to-tuple(RealVars(ctx, ["R1", "R2"])) as [AST, AST]
  val [Vin, Vout] = to-tuple(RealVars(ctx, ["Vin", "Vout"])) as [AST, AST]

  assert-on(s, R1 == 100.0e3)
  assert-on(s, Vin == 24.0)
  assert-on(s, Vout == 6.0)

  assert-on(s, R2 > 0.0)
  assert-on(s, Vout == Vin * (R2 / (R1 + R2)))

  val r = check(s)
  println("Result: %_" % [r])

  val m = get-model(s)
  println("%~" % [m])

```

This will generate:

```
Result: Z3_L_TRUE
R2 -> (/ 100000.0 3.0)
R1 -> 100000.0
Vout -> 6.0
Vin -> 24.0
/0 -> {
  (/ 100000.0 3.0) (/ 400000.0 3.0) -> (/ 1.0 4.0)
  else -> 0.0
}
```

Where R2 = 100k / 3.0 == 33.3k. This creates a voltage divider with an output of 6.0V when there
is an input of 24V.

# How to use in JITX

Add the following to your `slm.toml` file:

```
[dependencies.slm-z3]
pkg="slm-z3"
version="0.1.1"
type="conan"
options.shared = "True"
options.linux.fPIC = "True"
options.macos.fPIC = "True"
```

Then run `$SLM clean`. On next build, the dependency will be downloaded and you will be able to `import z3` to use in your project.

# Dev Setup

This project uses [conan](https://conan.io/) to manage compiling the Z3 library dependency for the current platform.

Currently the process is:

1.  Create a venv and install the dependencies
    1.  `python -m venv venv`
    2.  `source venv/bin/activate`
    3.  `pip install -r requirements.txt`
2.  Run Conan using the `Makefile`
    1.  `make`

This will construct a conan package that includes the stanza wrappers around Z3
and will download the `Z3` library dependency when installed with `slm`

NOTE: The unit tests currently cannot be run because the conan package build only provides the shared library versions. This leaves no method to build the tests and run them.

**OLD instructions**

To build the Z3 dependencies:

1.  Setup a compiler on the `$PATH`
    1.  Ubuntu: `sudo apt install build-essential`
    2.  Mac: `xcode-select --install`
    3.  Windows: Install MinGW
        1.  Use [MinGW-W64 gcc compiler](https://www.mingw-w64.org/).
        2.  I'm currently using version 12.2.0.
2.  Setup [Stanza](https://lbstanza.org/) on your path:
    1.  Check that `stanza version` reports something reasonable.
3.  Setup a virtualenv:
    1.  `python3 -m venv venv`
    2.  `source venv/bin/activate` or `venv/Scripts/Activate.ps1`
    3.  `pip install -r requirements.txt`
4.  Build the tests:
    1. Linux/Mac: `make z3-tests`
    2. Windows:
       1. `$env:SLM_BUILD_STATIC=1`
       2. `./build_conan.ps1`
       3. `stanza build z3-tests`
5.  Run the tests:
    1.  `./z3-tests`
    2.  On Mac/Linux - you can alternatively just run `make tests`

# Running the Tests

```
$> mingw32-make.exe tests
stanza build z3-tests
./z3-tests
[Test 1] test-cfg-basic
[PASS]

[Test 2] test-basic
[PASS]

[Test 3] test-bool-sort
[PASS]

[Test 4] test-int-sort
[PASS]

[Test 5] test-real-sort
[PASS]

[Test 6] test-bitvec-sort
[PASS]

[Test 7] test-finite-domain-sort
[PASS]

[Test 8] test-array-sort
[PASS]

[Test 9] test-numerals
asdf
[PASS]

[Test 10] test-ast-vector
[PASS]

[Test 11] test-basic
Result: Z3_L_TRUE
y -> 0
x -> 7

[PASS]

[Test 12] test-divider
Result: Z3_L_TRUE
R2 -> 100000.0
R1 -> 100000.0
Vout -> 12.0
Vin -> 24.0
/0 -> {
  100000.0 200000.0 -> (/ 1.0 2.0)
  else -> 0.0
}

[PASS]

Tests Finished: 12/12 tests passed. 0 tests skipped. 0 tests failed.

Longest Running Tests:
[PASS] test-basic (28 ms)
[PASS] test-divider (22 ms)
[PASS] test-cfg-basic (12 ms)
[PASS] test-basic (11 ms)
[PASS] test-bool-sort (11 ms)
[PASS] test-numerals (11 ms)
[PASS] test-int-sort (10 ms)
[PASS] test-finite-domain-sort (10 ms)
[PASS] test-ast-vector (10 ms)
[PASS] test-bitvec-sort (10 ms)
[PASS] test-real-sort (10 ms)
[PASS] test-array-sort (9290 us)

```


# Wrapper.stanza & Enums

This project uses the tool defined in [lbstanza-wrappers](https://github.com/callendorph/lbstanza-wrappers) to generate the C function wrappers and the enumeration definitions.

Highly recommend running this in Linux or Mac - Windows is not
well tested.

## Setup for Wrappers

In the same virtualenv, you will need to install additional dependencies:

```
(venv) $> pip install -r wrapper_requirements.txt
```

## Wrapper Build

You can use the `Makefile` to generate the wrapper:

```
(venv) $> make wrapper
```

This will build the static conan dependencies and then attempt
to extract the definitions from the `z3.h` header file into
the `src/Wrapper.stanza` file (as well as the `src/Enums` directory)


# Writing a solver

There are a couple examples in the unit tests. There are primarily two different types - Solvers and Optimizers. There is an interface called `Constrainable` that is intended to
make it easier to work with either.

The `Shellable` provides a means of implementing different scopes:
```
  within shell(s):
    assert-on(s, ((vout * vout) / r1) < 0.01)
    val r = check(s)
    ...
```

The asserts inside the `within` body will only apply during that scope. Once we leave
that scope, they will be removed from the stack of constraints.

