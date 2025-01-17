# Collaborative Fuzzing

## Design
In this cooperative framework, the fuzzers collaborate using a centralized
scheduler. 


# Components

The project consists of three parts: 1) the cooperative framework, 2) a scheduler, and 3) the drivers for the fuzzers.
In this framework, the scheduler selects input seeds to be scheduled on
different fuzzers, by communicating this over the fuzzer driver. At the same
time the centralized scheduler collects newly generated inputs from the
fuzzers, and repeats the loop.

## Cooperative framework
Communication happens over ZeroMQ. The seeds are send with a message consisting
of the fuzzer type ID and the seed (which contains the seed input and an
optional conditional to be solved).

## Scheduler
The scheduler component selects seed inputs from the global queue, evaluates
them, and schedules them to the fuzzers.

## Fuzzer drivers
The fuzzer drivers implement the communcation mechanisms with the centralized
scheduler. The communication happens over ZeroMQ.

# Docker setup
Th easiest way is to setup the framework using docker.

The setup consists of the following components:

- fuzzer-framework
- fuzzer-{afl,qsy,aflfast,fairfuzz..}-{target}
- fuzzer-generic-driver

## How to run:


1. Setup virtualenv 
    * ```virtualenv --python=python3 venv```
    * ```source venv/bin/activate```
2. Build the container docker images (+ install deps):
    * ```make all```
3. Install the ```collab_fuzz_xxx``` tools
    * ```cd runners && python setup.py install```
4. Test run (runs 1 afl instance and collab framework):
    * ```mkdir myrun && cd myrun && collab_fuzz_compose -f afl --scheduler=enfuzz -- objdump && docker-compose up```
    * To run with more fuzzers (2 afl + qsym + fairfuzz + aflfast): ```collab_fuzz_runner -f afl afl qsym fairfuzz aflfast --scheduler=enfuzz -- who```

Alternatively, to avoid the long building times, you can fetch the docker
images from a remote repo. For example `collab_fuzz_build --remote
sarek.osterlund.xyz --pull-reqs` to pull all images.


## Extra info
The ```afl__generic_driver``` runs in a container (fuzzer-generic-driver). This container needs to be privileged and mount ```/var/run/docker.sock```, such that it can control the container of the fuzzer to sync with.

### Docker settings
The logs generated by the containers might eat up a lot of space on your
machine. To avoid this, edit your `daemon.json` (typically `/etc/docker/daemon.json`) to contain something like:

```
{
    "log-level":        "error",
    "storage-driver":   "overlay2",
    "log-opts": {
	    "max-size": "512m"
    }
}
```

# Binutils Quickstart guide
At the moment the LAVA-M and GFT builds are broken. Binutils should still work, though.

## Setup virtualenv 
    * ```virtualenv --python=python3 venv```
    * ```source venv/bin/activate```


## Build the framework (for now only supports binutils, other builds are broken):

```make framework-binutils```

## Build fuzzer base images:

Remember the AFL/QSYM prerequisites (needed on every reboot):

```
echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope
cd /sys/devices/system/cpu
echo performance | sudo tee cpu*/cpufreq/scaling_governor
```
Then build the docker images (we only build AFL for now):

```cd docker && make fuzzer-afl```

## Build helper tools

```make tools```


You might need to install the collab_fuzz_runner tools in the following way:

```cd runner && make collab_fuzz_runner```

## Build actual targets (for now only build binutils)

```collab_fuzz_build -f afl -t objdump```

Or to build all of binutils for all fuzzers:
```collab_fuzz_build -s binutils```

## Test it all (run objdump with afl)

```
mkdir tmp_run && cd tmp_run && collab_fuzz_compose -f afl -- objdump
```

And then try to start the campaign:

```docker-compose up --abort-on-container-exit```

To clean up the campaign after exiting (i.e., delete the volumes):
```docker-compose down -v```

# Cite

CollabFuzz was presented at EuroSec 2021. [CollabFuzz: A Framework for Collaborative Fuzzing](https://download.vusec.net/papers/collabfuzz_eurosec21.pdf).

Video: [available on YouTube](https://www.youtube.com/watch?v=nf63VmIhWJQ)

Bibtex:

```bibtex
@inproceedings{eurosec_collabfuzz_2021,
	title = {{CollabFuzz}: {A} {Framework} for {Collaborative} {Fuzzing}},
	booktitle = {{EuroSec}},
	author = {Österlund, Sebastian and Geretto, Elia and Jemmett, Andrea and Güler, Emre and Görz, Philipp and Holz, Thorsten and Giuffrida, Cristiano and Bos, Herbert},
	month = apr,
	year = {2021},
}
```

# Related work
If you are interested in collaborative fuzzing, also check out our work on how to select the right set of fuzzers to use in a collaborative setting: [Cupid: Automatic Fuzzer Selection for Collaborative Fuzzing](https://www.ei.ruhr-uni-bochum.de/media/emma/veroeffentlichungen/2020/09/26/ACSAC20-Cupid_TiM9H07.pdf). Code: https://github.com/RUB-SysSec/cupid
