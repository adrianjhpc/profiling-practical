---
title: "Profiling on ARCHER2"
teaching: 30
exercises: 30
questions:
- "What profiling tools are available on ARCHER2 and how can I access them?"
- "Where can I find more documentation on and get help with these tools?"
objectives:
- "Know what tools are available to help you profile parallel applications on ARCHER2."
- "Know where to get further help."
keypoints:
- "The main profiling tool on ARCHER2 is *CrayPat*"
---

ARCHER2 has a range of profiling software available. In this section we provide a brief
overview of the tools available, their applicability and links to more information. A detailed tutorial
on profiling is beyond the scope of this course but if you are interested in this, then
you may be interested in the Performance Analysis Workshop course offered by the ARCHER2 service.

The [Cray Performance Measurement and Analysis Tools User Guide](https://support.hpe.com/hpesc/public/docDisplay?docId=a00115110en_us&docLocale=en_US&page=Cray_Performance_Measurement_and_Analysis_Tools_CPMAT.html)
and the ARCHER2 [profiling](https://docs.archer2.ac.uk/user-guide/profile/) documentation will also be useful.

## Profiling tools overview

Profiling on ARCHER2 is provided through the Cray Performance Measurement and Analysis Tools (CrayPat). These have
a number of different components:

* **CrayPat** the full-featured program analysis tool set. CrayPat in turn consists of the following major components.
    * pat_build, the utility used to instrument programs
    * the CrayPat run time environment, which collects the specified performance data during program execution
    * pat_report, the first-level data analysis tool, used to produce text reports or export data for more sophisticated analysis
* **CrayPat-lite** a simplified and easy-to-use version of CrayPat that provides basic performance analysis information automatically, with a minimum of user interaction.
* **Reveal** the next-generation integrated performance analysis and code optimization tool, which enables the user to correlate performance data captured during program execution directly to the original source, and identify opportunities for further optimization.
* **Cray PAPI** components, which are support packages for those who want to access performance counters
* **Cray Apprentice2** the second-level data analysis tool, used to visualize, manipulate, explore, and compare sets of program performance data in a GUI environment.

See the [Cray Performance Measurement and Analysis Tools User Guide](https://pubs.cray.com/bundle/Cray_Performance_Measurement_and_Analysis_Tools_User_Guide_644_S-2376/page/About_the_Cray_Performance_Measurement_and_Analysis_Tools_User_Guide.html).

Other tools available are:

* **Scalasca** used for the scalable performance analysis of large-scale parallel applications.
* **Linaro MAP** (previously known as Allinea or Arm MAP) is also available for ARCHER2 users.

## Using CrayPat Lite to profile an application

Let's grab and unpack a toy code for training purposes. To do this, we'll use `wget`.

```
auser@ln01:~>  wget {{site.url}}{{site.baseurl}}/files/nbody-par.tar.gz
```
{: .language-bash}

To extract the files from a `.tar.gz` file, we run the command `tar -xvf filename.tar.gz`:
```
auser@ln01:~> tar -xzf nbody-par.tar.gz
```
{: .bash}

Load CrayPat-lite module (`perftools-lite`)
```
auser@ln01:~> module load perftools-lite
```
{: .bash}

and move into the new ``nbody-par`` directory and compile the application normally
```
auser@ln01:~> cd nbody-par
auser@ln01:~/nbody-par> make
```
{: .bash}
```
cc -DMPI -c main.c -o main.o
cc -DMPI -c utils.c -o utils.o
cc -DMPI -c serial.c -o serial.o
cc -DMPI -c parallel.c -o parallel.o
cc -DMPI main.o utils.o serial.o parallel.o -o nbody-parallel.exe
INFO: creating the PerfTools-instrumented executable 'nbody-parallel.exe' (lite-samples) ...OK
```
{: .output}

As the output of the compilation says, the executable `nbody-parallel.exe` binary has been instrumented by CrayPat-lite. A batch script called ``run.slurm`` is in the directory. Change the account to your account so it can be run. You can also use the ``short`` QoS for a quick turnaround.

Once our job has finished, we can get the performance data summarised at the end of the job STDOUT.
```
auser@ln01:~/nbody-par> less slurm-out.txt
```
 {: .language-bash}
```
CrayPat/X:  Version 21.02.0 Revision ee5549f05  01/13/21 04:13:58
.
N Bodies =                     10240
Timestep dt =                  2.000e-01
Number of Timesteps =          10
Number of MPI Ranks =          64
BEGINNING N-BODY SIMULATION
SIMULATION COMPLETE
Runtime [s]:              3.295e-01
Runtime per Timestep [s]: 3.295e-02
interactions:             10
Interactions per sec:     3.182e+09

#################################################################
#                                                               #
#            CrayPat-lite Performance Statistics                #
#                                                               #
#################################################################

CrayPat/X:  Version 21.02.0 Revision ee5549f05  01/13/21 04:13:58
Experiment:                  lite  lite-samples
Number of PEs (MPI ranks):     64
Numbers of PEs per Node:       64
Numbers of Threads per PE:      1
Number of Cores per Socket:    64
Execution start time:  Tue Nov 23 15:18:45 2021
System name and speed:  nid005209  2.250 GHz (nominal)
AMD   Rome                 CPU  Family: 23  Model: 49  Stepping:  0
Core Performance Boost:  All 64 PEs have CPB capability

Avg Process Time:       0.47 secs
High Memory:         4,326.8 MiBytes     67.6 MiBytes per PE
I/O Write Rate:   701.173609 MiBytes/sec


...lots of output trimmed...

```
{: .output}

Instructions are given at the end of the output on how to see more detailed results and how to
visualise them graphically with the Cray Apprentice2 tool (more on this below).

You may have seen that other `perftools-lite` modules are available. The ones of interest to
ARCHER2 users will be `perftools-lite-event` which will instrument executables for event tracing
rather than sampling experiments, and `perftools-lite-loops` which will profile loops in the code.
The latter can only be used with the CCE compilers. To use either, just load these modules instead
of `perftools-lite`.

## Using CrayPat to profile an application

We are now going to use the full CrayPat tool. Here, we'll go through the process of performing an APA 
(Automatic Program Run) instrumented experiment, which is an easy way to run a 'sensible' event tracing experiment.
To do so, we first need to load the required modules

```
auser@ln01:~/nbody-par>  module unload perftools-lite
auser@ln01:~/nbody-par>  module load perftools
```
{: .language-bash}

After loading the modules, we will recompile the application from scratch
```
auser@ln01:~/nbody-par> make clean; make
```
{: .bash}

CrayPat-lite would at this stage have produced an executable ready to run a sampling experiment, but with CrayPat we
now need to decide what kind of experiment we want to perform with it. With the application built, we then use `pat_build`

```
auser@ln01:~/nbody-par> pat_build nbody-parallel.exe
```
{: .bash}

and a new binary called `nbody-parallel.exe+pat` will be generated. This is the binary that we need to run in order to
obtain the performance data. Extra options passed to `pat_build` will change the type of data we gather. Like this, with
no options, we will be performing a sampled run; as another example, using `pat_build -g mpi` would trace calls to MPI.

Continuing with our sampling experiment, replace the ``srun`` line in the Slurm script with the following
```
srun --cpu-bind=cores ./nbody-parallel.exe+pat -n 10240 -i 10 -t 1
```
 {: .language-bash}

After the job has finished, files are stored in an experiment data directory with the following format: `exe+pat+PID-node[s|t]` where:

* `exe`: The name of the instrumented executable
* `PID`: The process ID assigned to the instrumented executable at runtime
* `node`: The physical node ID upon which the rank zero process was executed
* `[s|t]`: The type of experiment performed, either `s` for sampling or `t` for tracing

For example, in our case a new directory called `nbody-parallel.exe+pat+192028-1341s` could be created. It is now time to obtain the performance report. We do this with the `pat_report` command and the new created directory
```
auser@ln01:~/nbody-par> pat_report nbody-parallel.exe+pat+192028-1341s
```
{: .bash}

This will command generate a full performance report and can generate a large amount of data, so you may wish to capture the data in an output file, either using a shell redirect like `>`,  or we could choose to see only some reports.

If we want to see only a profile report by function we can do

```
auser@ln01:~/nbody-par> pat_report -v -O samp_profile nbody-parallel.exe+pat+192028-1341s
```
{: .bash}

```
...run details...

Table 1:  Profile by Function

  Samp% | Samp | Imb. |  Imb. | Group
        |      | Samp | Samp% |  Function
        |      |      |       |   PE=HIDE

 100.0% | 33.4 |   -- |    -- | Total
|-------------------------------------------------------
|  86.4% | 28.8 |  1.2 |  3.9% | USER
||------------------------------------------------------
||  86.4% | 28.8 |  1.2 |  3.9% | compute_forces_multi_set
||======================================================
|  12.0% |  4.0 |   -- |    -- | MPI
||------------------------------------------------------
||   4.3% |  1.4 |  1.6 | 52.9% | MPI_File_write_all
||   2.9% |  1.0 |  0.0 |  1.6% | MPI_Recv
||   2.9% |  1.0 |  0.0 |  3.2% | MPI_File_open
||   1.8% |  0.6 |  2.4 | 81.0% | MPI_Sendrecv
||======================================================
|   1.6% |  0.5 |  1.5 | 74.6% | MATH
||------------------------------------------------------
||   1.6% |  0.5 |  1.5 | 74.6% | sqrt
|=======================================================

...more run details...
```
{: .output}

The table above shows the results from sampling the application. Program functions are separated out into different types, `USER` functions are those defined by the application, `MPI` functions contains the time spent in MPI library functions, `ETC` functions are generally library or miscellaneous functions included. `ETC` functions can include a variety of external functions, from mathematical functions called in by the library to system calls.

The raw number of samples for each code section is shown in the second column and the number as an absolute percentage of the total samples in the first. The third column is a measure of the imbalance between individual processors being sampled in this routine and is calculated as the difference between the average number of samples over all processors and the maximum samples an individual processor gave in this routine.

Another useful table can be obtained profiling by Group, Function, and Line
```
auser@ln01:~/nbody-par> pat_report -v -O samp_profile+src nbody-parallel.exe+pat+192028-1341s
```
{: .bash}

```
...run details...

Table 1:  Profile by Group, Function, and Line

  Samp% | Samp | Imb. |  Imb. | Group
        |      | Samp | Samp% |  Function
        |      |      |       |   Source
        |      |      |       |    Line
        |      |      |       |     PE=HIDE

 100.0% | 33.4 |   -- |    -- | Total
|--------------------------------------------------------------
|  86.4% | 28.8 |   -- |    -- | USER
||-------------------------------------------------------------
||  86.4% | 28.8 |   -- |    -- | compute_forces_multi_set
3|        |      |      |       |  work/ta043/nbody-par/parallel.c
||||-----------------------------------------------------------
4|||   3.6% |  1.2 |  2.8 | 70.6% | line.138
4|||   2.1% |  0.7 |  2.3 | 77.8% | line.140
4|||   1.6% |  0.5 |  2.5 | 83.1% | line.141
4|||   1.1% |  0.4 |  1.6 | 83.3% | line.142
4|||   2.6% |  0.9 |  2.1 | 72.5% | line.144
4|||   3.5% |  1.2 |  2.8 | 72.2% | line.145
4|||   1.5% |  0.5 |  1.5 | 77.0% | line.147
4|||  29.7% |  9.9 |  6.1 | 38.7% | line.148
4|||  18.2% |  6.1 |  4.9 | 45.3% | line.149
4|||  11.8% |  3.9 |  5.1 | 57.1% | line.150
4|||  10.5% |  3.5 |  4.5 | 57.1% | line.151
||||===========================================================
||=============================================================
|  12.0% |  4.0 |   -- |    -- | MPI
||-------------------------------------------------------------
||   4.3% |  1.4 |  1.6 | 52.9% | MPI_File_write_all
||   2.9% |  1.0 |  0.0 |  1.6% | MPI_Recv
||   2.9% |  1.0 |  0.0 |  3.2% | MPI_File_open
||   1.8% |  0.6 |  2.4 | 81.0% | MPI_Sendrecv
||=============================================================
|   1.6% |  0.5 |  1.5 | 74.6% | MATH
||-------------------------------------------------------------
||   1.6% |  0.5 |  1.5 | 74.6% | sqrt
|==============================================================

...more run details...
```
{: .output}

If we want to profile by Function and Callers, with Line Numbers then
```
auser@ln01:~/nbody-par> pat_report -O ca+src nbody-parallel.exe+pat+192028-1341s
```
{: .bash}

```
...run details...

Table 1:  Profile by Function and Callers, with Line Numbers

  Samp% | Samp | Group
        |      |  Function
        |      |   Caller
        |      |    PE=HIDE

 100.0% | 33.4 | Total
|--------------------------------------------------------------
|  86.4% | 28.8 | USER
||-------------------------------------------------------------
||  86.4% | 28.8 | compute_forces_multi_set
3|        |      |  run_parallel_problem:parallel.c:line.81
4|        |      |   main:main.c:line.81
||=============================================================
|  12.0% |  4.0 | MPI
||-------------------------------------------------------------
||   4.3% |  1.4 | MPI_File_write_all
3|        |      |  distributed_write_timestep:parallel.c:line.218
4|        |      |   run_parallel_problem:parallel.c:line.73
5|        |      |    main:main.c:line.81
||   2.9% |  1.0 | MPI_Recv
3|        |      |  run_parallel_problem:parallel.c:line.59
4|        |      |   main:main.c:line.81
||   2.9% |  1.0 | MPI_File_open
3|        |      |  run_parallel_problem:parallel.c:line.59
4|        |      |   main:main.c:line.81
||   1.8% |  0.6 | MPI_Sendrecv
3|        |      |  run_parallel_problem:parallel.c:line.75
4|        |      |   main:main.c:line.81
||=============================================================
|   1.6% |  0.5 | MATH
||-------------------------------------------------------------
||   1.6% |  0.5 | sqrt
3|        |      |  run_parallel_problem:parallel.c:line.81
4|        |      |   main:main.c:line.81
|==============================================================

...more run details...
```
{: .output}

> ## Filtering results
> The reports produced by `pat_report` are by default averages over all tasks. If you only want to see results for some
> subsection of tasks, you can filter them. To view the results from the first 10 tasks only, you would do `pat_report -sfilter_input='pe<10'`.
> On the other hand, `pat_report -sfilter_input='pe==0'` would produce results for task 0 only.
{: .callout}

> ## CrayPat-lite and `pat_report`
> You can use `pat_report` in just the same way with an experiment data directory that was generated by a CrayPat-lite run.
{: .callout}

The run will generate two other files in the experiment directory, one with the extension `.ap2` which holds the same data as the report data (`.xf`) but in a post processed form. The other file is called `build-options.apa` and is a text file with a configuration for generating a traced (as opposed to sampled) experiment. The APA (Automatic Program Analysis) configuration is a targeted trace, based on the results from our previous sampled experiment. You are welcome and encouraged to review this file and modify its contents in subsequent iterations, however in this first case we will continue with the defaults.

This `build-options.apa` file acts as the input to the `pat_build` command and is supplied as the argument to the `-O` flag.

```
auser@ln01:~/nbody-par> pat_build -O nbody-parallel.exe+pat+192028-1341s/build-options.apa
```
{: .bash}

This will produce a third binary with extension `+apa`. This binary should once again be run on the back end, so the submission script should be modified and the name of the executable changed to `nbody-parallel.exe+apa`.
Similarly to the sampling process, a new directory called `exe+apa+PID-node[s|t]` will be generated by the application, which should be processed by the `pat_report` tool. The output format of this new directory is similar to the one obtained with sampling, but now this includes `apa` and `t` to indicate that this is a tracing experiment. For instance, with our code we could get a new directory called `nbody-parallel.exe+apa+114297-5209t`.
```
auser@ln01:~/nbody-par> pat_report nbody-parallel.exe+apa+114297-5209t
```
{: .bash}

```
...run details...

Table 1:  Profile by Function Group and Function

  Time% |     Time |     Imb. |  Imb. | Calls | Group
        |          |     Time | Time% |       |  Function
        |          |          |       |       |   PE=HIDE

 100.0% | 0.402275 |       -- |    -- | 690.0 | Total
|-----------------------------------------------------------------
|  71.3% | 0.286987 | 0.003933 |  1.4% |   1.0 | USER
||----------------------------------------------------------------
||  71.3% | 0.286987 | 0.003933 |  1.4% |   1.0 | main
||================================================================
|  27.6% | 0.111100 |       -- |    -- | 686.0 | MPI
||----------------------------------------------------------------
||  14.0% | 0.056473 | 0.003279 |  5.6% |   1.0 | MPI_Recv
||   9.2% | 0.037024 | 0.000008 |  0.0% |  21.0 | MPI_File_write_all
||   2.3% | 0.009178 | 0.000001 |  0.0% |   1.0 | MPI_File_open
||   1.6% | 0.006504 | 0.000890 | 12.2% | 640.0 | MPI_Sendrecv
||================================================================
|   1.0% | 0.004188 |       -- |    -- |   3.0 | MPI_SYNC
|=================================================================

...more tables and details...
```
{: .output}

The new table above is the version generated from tracing data instead of the previous sampling data table. This version makes available true timing information (average per processor) and the number of times each function is called. We can also see a new section in Table 1, 'MPI_SYNC', which tells us how much time was spent waiting in collectives. This information is only found in event tracing runs.

We can also get very important performance data from the hardware (HW) counters
```
auser@ln01:~/nbody-par> pat_report -v -O profile+hwpc nbody-parallel.exe+apa+114297-5209t
```
{: .bash}
```
...run details...

Table 1:  Profile by Function Group and Function

Group / Function / PE=HIDE


==============================================================================
  Total
------------------------------------------------------------------------------
  Time%                                        100.0%
  Time                                       0.402275 secs
  Imb. Time                                        -- secs
  Imb. Time%                                       --
  Calls                           0.002M/sec    690.0 calls
  CORE_TO_L2_CACHEABLE_REQUEST_ACCESS_STATUS:
    LS_RD_BLK_C                   0.402M/sec  161,687 req
  L2_PREFETCH_HIT_L2              0.150M/sec   60,526 hits
  L2_PREFETCH_HIT_L3              0.034M/sec   13,861 hits
  REQUESTS_TO_L2_GROUP1:L2_HW_PF  0.599M/sec  240,888 ops
  REQUESTS_TO_L2_GROUP1:RD_BLK_X  0.386M/sec  155,439 ops
  Cache Lines PF from OffCore     0.448M/sec  180,362 lines
  Cache Lines PF from Memory      0.414M/sec  166,501 lines
  Cache Lines Requested from
    Memory                        0.371M/sec  149,261 lines
  Write Memory Traffic GBytes     0.017G/sec     0.01 GB
  Read Memory Traffic GBytes      0.050G/sec     0.02 GB
  Memory traffic GBytes           0.067G/sec     0.03 GB
  Memory Traffic / Nominal Peak                  0.0%
  Average Time per Call                      0.000583 secs
  CrayPat Overhead : Time          0.1%
==============================================================================

...lots more information...
```
{: .output}

Finally, you can examine the results of any CrayPat run (including CrayPat-lite) graphically using Apprentice2.
For this you will need to log in with X11-forwarding enabled -- on Linux and macOS this means logging in with
the `-X` option passed to `ssh`. Then, with the `perftools-base` module loaded, we are able to examine the
results of our traced run as follows:
```
auser@ln01:~/nbody-par> app2 nbody-parallel.exe+apa+114297-5209t
```
{: .bash}

## Getting help with profiling tools

You can find more information on the profiling tools available on ARCHER2 in the ARCHER2 Documentation
and the Cray documentation:

* [ARCHER2 Documentation](https://docs.archer2.ac.uk/user-guide/profile/)
* [Cray Technical Documentation](https://support.hpe.com/hpesc/public/docDisplay?docId=a00115110en_us&docLocale=en_US&page=Cray_Performance_Measurement_and_Analysis_Tools_CPMAT.html)

{% include links.md %}

