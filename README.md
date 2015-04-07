## Wrapped CBench (WCBench)

[![Join the chat at https://gitter.im/dfarrell07/wcbench](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/dfarrell07/wcbench?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

CBench, wrapped in stuff that makes it useful.

- [Wrapped CBench (WCBench)](#user-content-wrapped-cbench-wcbench)
    - [Overview](#user-content-overview)
    - [Usage](#user-content-usage)
        - [Usage Overview](#user-content-usage-overview)
        - [Usage Details: wcbench.sh](#user-content-usage-details-wcbenchsh)
        - [Usage Details: loop_wcbench.sh](#user-content-usage-details-loop_wcbenchsh)
        - [Usage Details: stats.py](#user-content-usage-details-statspy)
    - [WCBench Results](#user-content-wcbench-results)
    - [Contributing](#user-content-contributing)
    - [Contact](#user-content-contact)

### Overview

CBench is a somewhat classic SDN controller benchmark tool. It blasts a controller with OpenFlow packet-in messages and counts the rate of flow mod messages returned. WCBench consumes CBench as a library, then builds a robust test automation, stats collection and stats analysis/graphing system around it.

WCBench currently only supports the Helium release of the OpenDaylight SDN controller, but it would be fairly easy to add support for other controllers. Community contributions are encouraged!

Thsi is the Ubuntu version of wcbench. It will create a "wcbench" directory at the location you are cloning this repository. It will then install cbench and download OpenDayLight in the same folder.

### Usage

#### Usage Overview

The help outputs produced by `./wcbench.sh -h`, `./loop_wcbench.sh -h` and `stats.py -h` are quite useful:

```
Usage ./wcbench.sh [options]

Setup and/or run CBench and/or OpenDaylight.

OPTIONS:
    -h Show this message
    -v Output verbose debug info
    -c Install CBench
    -t <time> Run CBench for given number of minutes
    -r Run CBench against OpenDaylight
    -i Install OpenDaylight Helium 0.2.1
    -p <processors> Pin ODL to given number of processors
    -o Start and configure OpenDaylight Helium 0.2.1
    -k Kill OpenDaylight
    -d Delete local ODL and CBench code
```

```
Usage ./loop_wcbench.sh [options]

Run WCBench against OpenDaylight in a loop.

OPTIONS:
    -h Show this help message
    -v Output verbose debug info
    -l Loop WCBench runs without restarting ODL
    -r Loop WCBench runs, restart ODL between runs
    -t <time> Run WCBench for a given number of minutes
    -p <processors> Pin ODL to given number of processors
```

```
usage: stats.py [-h] [-S]
                [-s {five_load,ram,steal_time,one_load,iowait,fifteen_load,runtime,flows} [{five_load,ram,steal_time,one_load,iowait,fifteen_load,runtime,flows} ...]]
                [-G]
                [-g {five_load,ram,steal_time,one_load,iowait,fifteen_load,runtime,flows} [{five_load,ram,steal_time,one_load,iowait,fifteen_load,runtime,flows} ...]]

Compute stats about CBench data

optional arguments:
  -h, --help            show this help message and exit
  -S, --all-stats       compute all stats
  -s {five_load,ram,steal_time,one_load,iowait,fifteen_load,runtime,flows} [{five_load,ram,steal_time,one_load,iowait,fifteen_load,runtime,flows} ...], --stats {five_load,ram,steal_time,one_load,iowait,fifteen_load,runtime,flows} [{five_load,ram,steal_time,one_load,iowait,fifteen_load,runtime,flows} ...]
                        compute stats on specified data
  -G, --all-graphs      graph all data
  -g {five_load,ram,steal_time,one_load,iowait,fifteen_load,runtime,flows} [{five_load,ram,steal_time,one_load,iowait,fifteen_load,runtime,flows} ...], --graphs {five_load,ram,steal_time,one_load,iowait,fifteen_load,runtime,flows} [{five_load,ram,steal_time,one_load,iowait,fifteen_load,runtime,flows} ...]
                        graph specified data
```

#### Usage Details: wcbench.sh

Most of the work in WCBench happens in `wcbench.sh`. It's the bit that directly wraps CBench, automates CBench/ODL installation, collects system stats, starts/stops ODL, and runs CBench against ODL.

In more detail, the `wcbench.sh` script supports:

* Trivially cloning, building and installing CBench via the [oflops repo](https://github.com/andi-bigswitch/oflops).
* Changing a set of CBench params (MS_PER_TEST, TEST_PER_SWITCH and CBENCH_WARMUP) to easily set the overall length of a CBench run in minutes. Long CBench runs typically produce results with a lower standard deviation.
* Running CBench against an instance of OpenDaylight. The CBench and ODL processes can be on the same machine or different machines. Change the `CONTROLLER_IP` and `CONTROLLER_PORT` variables in `wcbench.sh` to point CBench at the correct ODL process. To run against a local ODL process that's on the default port, set `CONTROLLER_IP="localhost"` and `CONTROLLER_PORT=6633`. To run CBench against a remote ODL process, set `CONTROLLER_IP` to the IP address of the machine running ODL and `CONTROLLER_PORT` to the port ODL is listening on. Obviously the two machines have to have network connectivity with each other. Additionally, since WCBench connects to the remote machine to pull system stats, the remote machine needs to have sshd running and properly configured. The local machine should have its `~/.ssh/config` file and public keys setup such that `ssh $SSH_HOSTNAME` works without a password or RSA unknown-key prompt. To do this, setup something like the following in `~/.ssh/config`:

```
Host cbench
    Hostname 209.132.178.170
    User fedora
    IdentityFile /home/daniel/.ssh/id_rsa_nopass
    StrictHostKeyChecking no
```

As you likely know, `ssh-copy-id` can help you setup your system to connect with the remote box via public key crypto. If you don't have keys setup for public key crypto, google for guides (very out of scope). Finally, note that the `SSH_HOSTNAME` var in `wcbench.sh` must be set to the exact same value given on the `Host` line above.
* Trivially installing/configuring ODL from the last successful build (via an Integration team Jenkins job).
* Pinning the OpenDaylight process to a given number of CPU cores. This is useful for ensuring that ODL is properly pegged, working as hard as it can with given resource limits. It can also expose bad ODL behavior that comes about when the process is pegged.
* Running OpenDaylight and issuing all of the required configurations.
* Stopping the OpenDaylight process. This is done cleanly via the `./bin/stop` script, not `kill` or `pkill`.
* Cleaning up everything changed by the `wcbench.sh` script, including deleting ODL and CBench sources and binaries.


#### Usage Details: loop_wcbench.sh

The `loop_wcbench.sh` script is a fairly simple wrapper around `wcbench.sh` ("I hear you like wrappers, so I wrapped your wrapper in a wrapper"). Its reason for existing is to enable long series of repeated WCBench runs. As described in the [WCBench Results](https://github.com/dfarrell07/cbench_regression#wcbench-results) section, these results will be stored in a CSV file and can be analyzed with `stats.py`, as described in the [Usage Details: stats.py](https://github.com/dfarrell07/cbench_regression#usage-details-statspy) section. Doing many WCBench runs allows trends over time to be observed (like decreasing perf or increasing RAM). More results can also yield more representative stats.

In more detail, the `loop_wcbench.sh` script supports:

* Repeatedly running WCBench against ODL without restarting ODL between runs. This test revealed the perf degradation over time described in [bug 1395](https://bugs.opendaylight.org/show_bug.cgi?id=1395).
* Repeatedly running WCBench against ODL, restarting ODL between runs. This acted as a control when finding [bug 1395](https://bugs.opendaylight.org/show_bug.cgi?id=1395), as restarting ODL between runs mitigated perf decreases. 
* Pass run length info to WCBench, causing WCBench runs to last for the given number of minutes. Note that longer runs seem to result in lower standard deviation flows/sec results.
* Pin ODL to a given number of processors. This is basically a thin hand-off to `wcbench.sh`. As mentioned above, pinning ODL allows it to be tested while the process is properly pegged.

#### Usage Details: stats.py

The `stats.py` script parses the output of `wcbench.sh`, which is stored in the file pointed at by the `RESULTS_FILE` variable in `wcbench.sh`. See the [WCBench Results](https://github.com/dfarrell07/cbench_regression#wcbench-results) section for more info on the results file. Both pure stats and graphs of results are supported by `stats.py`.

Any set of names for data points can be given to the `-s` flag to calculate their stats and to the `-g` flag to graph them against run numbers. All stats can be computed with `./stats.py -S`, just as all graphs can be generated with `-G`.

Examples are useful:

```
# CBench flows/sec stats
./stats.py -s flows
{'flows': {'max': 15036,
           'mean': 8958.426,
           'min': 4981,
           'relstddev': 32.032,
           'stddev': 2869.584},
 'sample_size': 115}
```

```
# All stats
./stats.py -S
{'fifteen_load': {'max': 0,
                  'mean': 0.62,
                  'min': 0,
                  'relstddev': 0.0,
                  'stddev': 0.0},
 'five_load': {'max': 0,
               'mean': 0.96,
               'min': 0,
               'relstddev': 0.0,
               'stddev': 0.0},
 'flows': {'max': 22384,
           'mean': 22384.52,
           'min': 22384,
           'relstddev': 0.0,
           'stddev': 0.0},
 'iowait': {'max': 0, 'mean': 0.0, 'min': 0, 'relstddev': 0.0, 'stddev': 0.0},
 'one_load': {'max': 0,
              'mean': 0.85,
              'min': 0,
              'relstddev': 0.0,
              'stddev': 0.0},
 'runtime': {'max': 120,
             'mean': 120.0,
             'min': 120,
             'relstddev': 0.0,
             'stddev': 0.0},
 'sample_size': 1,
 'steal_time': {'max': 0,
                'mean': 0.0,
                'min': 0,
                'relstddev': 0.0,
                'stddev': 0.0},
 'used_ram': {'max': 3657,
              'mean': 3657.0,
              'min': 3657,
              'relstddev': 0.0,
              'stddev': 0.0}}
```

```
# Create graphs of flows/sec and used RAM stats
./stats.py -g flows ram
```

Graph results:

![Graphs of flows/sec and used RAM against run number](https://cloud.githubusercontent.com/assets/880273/3562723/5b854538-0a02-11e4-8fb1-dd1544d20ae6.png)


### WCBench Results

Results from `wcbench.sh` are stored in the file pointed at by the `RESULTS_FILE` variable in `wcbench.sh`. That variable defaults to `RESULTS_FILE=$BASE_DIR/"results.csv"`, which in turn resolves to `~/results.csv` by default. As you can guess from the file name, results are stored in CSV format. Note that this format was chosen because it's what's consumed by the Jenkins Plot Plugin, which ODL uses to [automatically run a subset of the functionality provided by WCBench against ODL builds](https://jenkins.opendaylight.org/integration/job/integration-master-performance-plugin/plot/).

Note that manually modifying the results file (adding/deleting lines) will cause incorrect run number values.

The data collected by WCBench and stored in the results file for each run includes:
* A run number for each run, starting at 0 and counting up
* The flows/sec average from the CBench run
* Unix time in seconds at the beginning of the run
* Unix time in seconds at the end of the run
* The IP address of the controller
* Human-readable time that the run finished
* The number of switches simulated by CBench
* The number of MAC addresses used by CBench
* The `TESTS_PER_SWITCH` value passed to CBench
* The duration of each test in milliseconds
* The steal time on the system running ODL at the start of the test
* The steal time on the system running ODL at the end of the test
* The total RAM on the system running ODL
* The used RAM on the system running ODL at the end of a test run
* The free RAM on the system running ODL at the end of a test run
* The number of CPUs on the system running ODL
* The one minute load of the system running ODL
* The five minute load of the system running ODL
* The fifteen minute load of the system running ODL
* The name of the controller under test
* The iowait value at the start of the test on the system running ODL
* The iowait value at the end of the test on the system running ODL

### Contributing

Contributions are encuraged! Contributions are encuraged! Contributions are encuraged! <- I can't say this enough.

The best way to contribute code is to jump on an existing [GitHub Issue](https://github.com/dfarrell07/wcbench/issues) or raise your own, then fork the code, make your changes and submit a pull request to merge your changes back into the main codebase.

Bugs or feature requests should be raised as [Issues](https://github.com/dfarrell07/wcbench/issues).

Note that the code is Open Source under a BSD 2-clause license.

### Contact

For feature requests, bug reports and questions please raise an [Issue](https://github.com/dfarrell07/wcbench/issues). Daniel Farrell is the primary developer of this tool. He can be contacted directly at dfarrell@redhat.com or on IRC (dfarrell07 on Freenode). **Prefer public, documented communication like Issues over direct 1-1 communication. This is an Open Source project. Keep the community in the loop.**
