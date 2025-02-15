# Table fuzzer

`build.sh` and `table_fuzzer.cc` are run continously through OSS-fuzz.

Link to OSS-fuzz integration: Pending

# Translation fuzzers

Currently, there is 2 fuzzers related to translation, **fuzz_translate** that will target **lou_translateString** and **fuzz_backtranslate** that will, in a similar way, fuzz **lou_backTranslateString**. The following sections will explain how to configure the fuzzers, how to use them and how to get a coverage report of the fuzzing result. 

## Configure the project for fuzzing

We have added some switchs to configure.ac for fuzzing and coverage. The `--with-fuzzer` switch will check if your are actually using clang and clang++ as compilers (by looking at CC and CXX) and allows generation of compilation instructions for fuzzer targets. The `--with-coverage` will add `-fprofile-instr-generate -fcoverage-mapping` to AM_CPPFLAGS in tests/makefile.am and liblouis/makefile.am.

To configure and build the project with coverage and fuzzer.
```
./autogen.sh
CC=clang CXX=clang++ ./configure --with-coverage --with-fuzzer
make -j8
```

## Run the fuzzers

You are now able to run the fuzzer and will have to give 2 parameters to it. First, you need to choose a table (or many) to fuzz and set the `FUZZ_TABLE` environment variable to them. Then, you need to provide a corpus with files containing sample inputs that will be used by libfuzzer to craft the data passed to the fuzzing function (the idea is to keep corpus as minimal as possible). If you don't provide any corpus directory, libfuzzer just generates random inputs.

Here is how you can start fuzzing for `lou_translateString` function.
```
# first we move to tests/fuzzing directory
cd tests/fuzzing

# we consider here you have added corpus files into tests/fuzzing/CORPUS directory
FUZZ_TABLE=../../tables/en-us-g2.ctb ./fuzz_translate CORPUS/

# to run the fuzzer using parallelization
# you can even set more jobs than workers (the ones that just stopped will be instantly replaced by a new fuzzer process)
FUZZ_TABLE=../../tables/en-us-g2.ctb ./fuzz_translate CORPUS/ -workers=8 -jobs=8

# start the fuzzer and let it generate random input (without corpus files), here we doesn't parallelize the process
FUZZ_TABLE=../../tables/en-us-g2.ctb ./fuzz_translate 

```

After running the fuzzer multiple times with the same corpus directory, it might be possible that many corpus files added by the fuzzer explores the same paths. Hopefully, libfuzzer allows you to minimize a corpus. There is a simple bash script in tests/fuzzing that allows you to do that.
```
./minimize-corpus.sh CORPUS/

# if you have added a POC file in the corpus directory and you want to keep it intact, change his extension to .txt and use --preserve-txt switch that keep .txt files intact in the directory
./minimize-corpus.sh --preserve-txt CORPUS/
```

## Look at fuzzer coverage

If you want to see what are the source code parts that are explored by the fuzzer, you can use clang coverage. So, you have to configure with coverage switch, run the fuzzer and show coverage data from the run with llvm tools. 
To be able to use the coverage data, you need first to compile the raw profile data file of the run. By default, this file is created after execution under the name of default.profraw but you can specify it with `LLVM_PROFILE_FILE`.
Here is how to do that.
```
LLVM_PROFILE_FILE=fuzz_translate.profraw FUZZ_TABLE=../../tables/en-us-g2.ctb ./fuzz_translate CORPUS/ -workers=8 -jobs=8

# wait for a bit and press CTRL+C

# compile raw profile
llvm-profdata merge -sparse fuzz_translate.profraw -o fuzz_translate.profdata

# show coverage (redlines are the one wich are reached)
# Actually tests/fuzzing/fuzz_translate is just a wrapper to tests/fuzzing/.libs/fuzz_translate, 
# so for covering we directly pass the binary
llvm-cov show .libs/fuzz_translate -instr-profile=fuzz_translate.profdata

```
