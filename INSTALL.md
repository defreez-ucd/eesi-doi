# EESI Artifact Installation and Usage

# 0. Start here

All commands in this document are run from the root of the EESI repository,
unless otherwise specified. 

To enhance reproducibility, extensive use is made of docker. Therefore having
a working docker installation is necessary. On Linux the typical installation procedure 
is to install the docker package (`docker.io` in Ubuntu). You will need to add your user
to the `docker` group. If you receive a permission denied error, check that your user
is in the correct group for your operating system distribution.

Host machine dependencies:
- docker
- curl

Python dependencies for creating Tables:
- pandas 0.17.1

You can install the Python dependencies by hand or run the Python scripts
inside the `defreez/eesi:dev` container.

The publicly available docker repository is defreez/eesi. The image tags are
```
# The development environment with necessary dependencies
# to build EESI.
defreez/eesi:dev    

# The development environment can be entered by running 
docker run -it --rm -v $PATHTOEESI:/eesi defreez/eesi:dev

# A compiled version of EESI. The artifact tag matches latest
# the source code at the time of artifact submission, while
# latest will keep moving forward.
defreez/eesi:latest 
defreez/eesi:artifact
```

The docker image can also be built from source from the Dockerfile in 
this repository.
```
docker build -t defreez/eesi:artifact .
```

The spreadsheets containing the bug reports and specification evaluation that
were referenced in the paper are available in the `results/camera` directory.
These spreadsheets contain the sorted bug reports that were used for the
paper submission (produced in section 2) and also the specification samples
selected for manual evaluation.

The next section contains general usage instructions for EESI. 
If you just want to reproduce the results in the paper, jump to section 2.

## Installation

EESI is meant to be run as a docker command line tool. The docker image
can be obtained with the command `docker pull defreez/eesi:artifact`.

The number of docker flags can become unwieldy, therefore the commands in
this section assume the existence of one alias. You can either type it 
in your shell for the single sessions, or persist it in a shell 
configuration file such as `.bashrc`.

```
alias eesi='docker run -it --rm -v $PWD:/d -w /d defreez/eesi:artifact'
```

## Compiling by hand in the dev environment

The scripts use the artifact docker image which has pre-built binaries.
If you want to build the source artifact by hand, it can be done as follows:

```
git clone https://github.com/ucd-plse/eesi
docker run -it --rm -v eesi:/eesi defreez/eesi:dev
cd /eesi/src
mkdir build && cd build
cmake ..
make -j$(nproc)
```

# 1. General Tool Usage

Running EESI without any parameters (or `--help`) will print the usage.
```
Options:
  --help                produce help message
  --bitcode arg         Path to bitcode file
  --command arg         Command (See README)
  --output arg          Path to output file
  --erroronly arg       Path to error-only functions file
  --inputspecs arg      Path to input specs list file
  --specs arg           Path to specs file
```

### bitcode

`  --bitcode arg         Path to bitcode file`

EESI analyzes LLVM bitcode files generated by `clang` version 7. To analyze a 
program you need to generate a bitcode file for that program.

It is recommend to use either wllvm or gllvm to create the bitcode file.

EESI works best with bitcode files that have the `reg2mem` pass. 

```
opt -reg2mem input.bc -o output.bc
```


### command

`--command arg         Command (See README)`

The available commands are `specs` and `bugs`. The `specs` command is used
to infer functions error specifications. The `bugs` command finds violations
of function error specifications.


#### Example (inferring specifications)

Given a bitcode file, a list of input specifications, and a list of error-only
functions, this command will infer error-specifications.. See below
for input formats.

```
eesi --command specs --bitcode BITCODEFILE --inputspecs INPUTSPECS.txt \
    --erroronly ERRORONLY.txt | tee specs-out.txt
```

### Example (finding bugs)

The output of the `specs` command becomes the input of the `bugs` command.
Note that the `bugs` command uses the `--specs` argument while the `specs`
command uses the `--inputspecs` argument.

```
eesi --command bugs --bitcode BITCODEILE --specs specs-out.txt
```

#### Toy example

This shows a toy example of running EESI on the following C program.
This illustrates the trivial case where a constant is returned immediately
after a call to an error-only function

The file `src/tests/test1-trivial-nonec/test.c`
```
// error-only function
void EO();

int foo(int x) {
	if (x) {
		EO();
		return -1;
	} else {
		return 1;
	}
}
```

Error-only input file `src/tests/test-erroronly.txt`
```
EO
```

Building and running EESI from scratch

```
docker run -it --rm -v $PATHTOREPO:/eesi defreez/eesi:dev
cd /eesi/src
mkdir build && cd build
cmake ..
make -j$(nproc)
cp ../tests/test1-trivial-nonec/test.c .

clang-7 -c -emit-llvm test.c

./eesi --bitcode test.bc --command specs --erroronly ../tests/test-erroronly.txt
```

Correct output
```
WARNING: EMPTY INPUT SPECS LIST!
foo: foo <0
```

The warning is printed because usually we run EESI with input specifications 
*and* error only functions, but either is optional.

The output shows that EESI has inferred a specification of `<0` because 
the value -1 is returned immediately after calling the error-only function
`EO`.


# 2. Reproducing Paper Results

The primary script used to run EESI on the targets discussed in the 
paper is `src/scripts/tabledata.sh`. Examples are shown below.

**NOTE: Running tabledata.sh for all targets requires 128GB of RAM!**
If you have less than 128GB of RAM then you can replace `all` with 
`small` in the tabledata scripts. This will skip `linux-fullkernel`
`linux-fs`.

## The results directory

The output of scripts you run here will put results in the `results/artifact`
directory. The output from the scripts when we ran them is in the 
`results/camera` directory. You can compare the output that you get
with the output in `results/camera` to determine if the script has
run correctly.

## Input Data

The input data used for the paper is available at 
https://eesi.defreez.com/data.tgz.
This tarball needs to be unpacked to create the directory `data` in the 
root of this repoistory.

The `tabledata.sh` script will attempt to download this file automatically
with `curl` if the `data` directory does not exist.

## Generating specification and bug reports

To generate the specification and bug reports for the tables in the
paper run: 

```
# All results (128GB of RAM)
src/scripts/tabledata.sh specs artifact all docker
src/scripts/tabledata.sh bugs artifact all docker

# All results except linux-fullkernel and linux-fs (32GB of RAM)
src/scripts/tabledata.sh specs artifact small docker
src/scripts/tabledata.sh bugs artifact small docker
```

If, for some reason this fails, try running inside the dev docker container

```
docker run -it --rm -v $PATHTOREPO:/eesi defreez/eesi:dev
cd /eesi/src
mkdir build && cd build
cmake ..
make -j$(nproc)
cd ../..

# Note the change to local for running inside dev environment
src/scripts/tabledata.sh specs artifact all local
src/scripts/tabledata.sh bugs artifact all local
```

If you want to generate results for individual targets it can 
be done with the commands below. This is not necessary if you 
use the commands above with the "all" target.

```
src/scripts/tabledata.sh specs artifact pidgin-otrng docker
src/scripts/tabledata.sh bugs artifact pidgin-otrng docker

src/scripts/tabledata.sh specs artifact openssl docker
src/scripts/tabledata.sh bugs artifact openssl docker

src/scripts/tabledata.sh specs artifact mbedtls docker
src/scripts/tabledata.sh bugs artifact mbedtls docker

src/scripts/tabledata.sh specs artifact netdata docker
src/scripts/tabledata.sh bugs artifact netdata docker

src/scripts/tabledata.sh specs artifact linux-fs docker
src/scripts/tabledata.sh bugs artifact linux-fs docker

src/scripts/tabledata.sh specs artifact linux-nfc docker
src/scripts/tabledata.sh bugs artifact linux-nfc docker

src/scripts/tabledata.sh specs artifact linux-fullkernel docker
src/scripts/tabledata.sh bugs artifact linux-fullkernel docker

src/scripts/tabledata.sh zlib artifact linux-fullkernel docker
src/scripts/tabledata.sh zlib artifact linux-fullkernel docker
```

The script will put data into `results/artifact`.

## Table 1 (error specifications) 

Table 1 has the counts for EESI specifications for each program.
The script `src/scripts/table1.py` is used to parse the EESI
outputs to get these numbers. It takes one parameter which is the 
path to the results subdirectory for your runs.

To parse the results from the artifact output produced here:
```
python3 src/scripts/table1.py --results results/artifact
```

To parse the results that were created from our runs:
```
python3 src/scripts/table1.py --results results/camera
```

## Table 2 (bugs)

In some cases EESI produces a large quantity of bug reports. We employ
a simple ranking scheme to prioritize bug reports - the ratio of 
buggy uses of the function returning an error to correct uses of the function.

To create the ranked list of bugs that we inspected, including the OpenSSL
bugs that were reported and merged, use the `src/sort_bugs.py` script. The
one parameter to this script is the EESI bugs output to be sorted. For targets
that produced a large number of bug reports (e.g. OpenSSL) we inspected only
the top ranked reports.

```
python3 src/scripts/sort_bugs.py --bugs results/camera/openssl-bugs.txt
```

The actual counts were the result of the manual inspection. These 
counts are available in spreadsheet form [on Figshare](https://figshare.com/s/ec71e238010d601d7c0e).

_Correction from original data_
We discovered 12 duplicate bug reports in the original Pidgin OTRv4 bug data.
This has since been corrected.

## Table 3

This table involves a count of manual inspection of results and is therefore
not automatically produced by this artifact. The bug reports we looked 
at to produce the false positive breakdown are the same as those 
on Figshare.

## Table 4

Table 4 has the performance numbers to produce these specifications. 
When using `tabledata.sh` to create the table 1 data, wall clock 
time is logged in the results directory as `$TARGET-spectime.txt` or
`$TARGET-bugtime.txt` depending the command being run. 
