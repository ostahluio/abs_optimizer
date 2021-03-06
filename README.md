# ABS Parameter Optimizer

This tool, called ABS Parameter OPTimizer (POPT) allows the optimization of
the parameters of a model written in ABS.

The basic idea is to run the simulation of the ABS model changing its
parameters and trying to figure out what is the best combination of parameter
based on the output of the model and a metric to optimize.

To understand what are the best parameters, instead of doing a grid search that
could result in an explosion of the number of simulations to be run, we use the
configurator optimizer [SMAC](http://www.cs.ubc.ca/labs/beta/Projects/SMAC/).

## Tool installation

The tool can be easily install by using by using the
[Docker](https://www.docker.com/) container technology.  Docker supports a huge
variety of OS. In the following we assume that it is correctly installed on
a Linux OS (a similar procedure can be used to install the tool on a Windows
or MacOS).

To create the Docker image please run the following command.

```
sudo docker pull jacopomauro/abs_optimizer:v1.1
```

For more information related to Docker and how to use it we invite the reader
to the documentation at [https://www.docker.com/](https://www.docker.com/).


## Usage

The tool requires the definition of an ABS model, the definition of the
metric to evaluate a simulation, the definition of the settings to tune,
and additional parameters to control the execution of the SMAC back-end.

### Running the Docker container

After the Docker image has been pulled, it is possible to start the create
a Docker container as follows.

```
sudo docker run -i --net="host" --name abs_popt_container -t jacopomauro/abs_optimizer:v1 /bin/bash
```

The Docker container called `abs_popt_container` will start and the bash
prompt will be given.

### Defining the ABS model and the parameters

We assume the existence of an ABS model. This model exposes parameters
that can be tuned. Since ABS main execution does not support the setting
of external parameters, for ABS POPT the ABS code need to contained for all
parameters named `X` a unique line as follows.

```
def Int X() = 0;
```

For the time being parameters can only be integers.

Once the model defining the parameters is defined, the location of the model
can be given to ABS POPT by setting the correct variables in the `settings.py`
present in the main directory of the repository. The model will need to be 
copied in the container previously created (e.g., by using `scp` or the volume
sharing capabilities of Docker).

In particular we assume that all the parameters are defined in a single
ABS program which location needs to be given using the variable `MODEL`.
For instance, the following line states that the ABS code defining the
parameters is defined in `Settings.abs` file save in the `abs_model` folder.

```
MODEL = "./abs_model/Settings.abs"
```

In case to run the model other files need to be compiled, then the list
of the ABS files can be given using the variable `ADDITIONAL_ABS_FILES`.
These will be the list of the files that with the one defined in the variable
`MODEL` will be pass to the ABS compiler.

The `settings.py` contains other useful variables to set to control the execution
of the simulation:
- the variable `CLOCK_LIMIT` allows the possibility to set the upper bound
	of the clock of the ABS simulation (i.e., when the run is invoked with the
	`-l' option). In case no upper bound is needed this value can be set to -1,
- the variable `TIMEOUT` sets the timeout of the simulation in seconds. When
	the timeout is triggered the model simulation is interrupted and evaluated
	considering the output so far generated,
- the variable `OUTPUT_TIMEOUT` allows the stop of a simulation in case the
	program is bugged and does not output anything in less than `OUTPUT_TIMEOUT`
	seconds. If the program is reliable and correct then this value can be set to
  -1.
- the variable `ERROR_NUMBER` allows to define how many times the program is
	executed in case it was stop because it did not output anything in less then
	`OUTPUT_TIMEOUT` seconds (assuming that this feature is used).

### Defining the metric

ABS POPT tries to find out the best parameters maximizing the quality of
a simulation.  It is vital therefore to provide a function that associates
the output of a simulation with its quality.  This function is defined by
means of a python program in the program `parse_abs_output.py`.

In this program ABS POPT expects the user to define the python code to parse
the output of the simulation and then return a real number representing
the quality of the simulation. The function that performs that is called
`compute_quality` that takes in input the string of the output generated by
running the ABS model. SMAC will try to find configuration where the number
returned by the `compute_quality` is smaller (minimization problem). 

In case of errors due to the model execution is is possible to avoid the
computation of the quality raising an exception instead or returning a
real value.

### Defining the range of the parameters

ABS POPT automatically selects possible domain values for the parameters but
requires to understand what are the parameter considered and what are their
possible ranges. The parameters are defined in the file `params.pcs`. The
format to use is exactly the one used by the SMAC tool.  Please see
[SMAC manual](http://www.cs.ubc.ca/labs/beta/Projects/SMAC/v2.10.03/manual.pdf)
for more details.

As an example, let us assume to have a parameter X that can range in the
domain [1,10] and a parameter Y that can take the values {4,6,9}. Assume
that the default value for X is 5 and that the default value for Y is 9.

This parameters can be defined as follows.

```
X integer [1,10] [5]
Y ordinal {4,6,9} [9]
```

It is also possible to require the satisfaction of some constraints between
the values taken by different parameters. For instance stating that the
parameter X should be greater then Y can be done as follows.

```
{ X <= Y }
```

### Additional parameters to control SMAC

SMAC offers a lot of parameters that can be configured. These parameters can be
set in the file `scenario.txt`.
In particular, there are two parameters that the user needs to be aware of:
- `numberOfRunsLimit` that limits the number of successful simulation runs
	on a single machine (when ABS POPT is executed in parallel every execution
	of SMAC can potentially perform up to `numberOfRunsLimit` simulations),
- `wallClockLimit` that limits the overall time ABS POPT runs. The bound is
	given in seconds.

For more parameters we invite the reader to consult the 
[SMAC manual](http://www.cs.ubc.ca/labs/beta/Projects/SMAC/v2.10.03/manual.pdf).

### Running ABS POPT

After the models has been copied, all the parameters setting fixed,
ABS POPT can be run by invoking the following command within the 
Docker container.

```
./run_smac.sh
```

By default this tool will invoke SMAC. Depending on the hardware, it is
possible to configure the parallel runs of SMAC by setting the bash variable
`PAR_PROC` to the number of desired parallel SMAC executions.

It is recommended to run this command by using the
`screen` utility. In this way it will be possible to monitor also the
resource consumption of the Docker container and visualize the logs of the
SMAC executions.

The output of the tool will be saved in the directory `smac-output/test`
following the output syntax of SMAC (the logs of the single execution of SMAC 
are save in the files `log_run_smacXXX.log`).
In case of parallel execution, it is possible to merge all the states to find
the best combination of parameters tried so far.
This can be done by running the following command.

```
./merge_states.sh smac-output/test smac-output/merge
```

This runs the SMAC utility to merge the parallel computation of SMAC creating
a new merge state in the directory `smac-output/merge`.  It will also output
the settings of the best simulation run so far and its quality.

Note that the `run_smac.sh` can be run also to resume a finish computation
to refine and possibly find better parameters.
Assuming that the previous `merge_states.sh` has been run, this can be done
by running the following command.

```
./run_smac.sh --warmup smac-output/merge
```

In case the computation of SMAC has been interrupted it can be resumed by running
the following command.

```
./run_smac.sh --restore
```

### Cleaning

To clean up the Docker installation, the following commands can be used.

```
sudo docker rm abs_popt_container
sudo docker rmi jacopomauro/abs_optimizer:v1
```

## Using the Numascale cluster

ABS POPT can be easily installed on Linux machines and therefore on the
Numascale cluster (please see the Dockerfile to check the list of the packages
required).

To run the ABS POPT, assuming it has been properly installed (in particular
please make sure that the absc compiler is available as a shell command),
the following command can be invoked.

```
./run_numascale.sh

```

As was happening with the `run_smac.sh` bash script, the number of 
times SMAC is run in parallel can be configured by setting the variable
`PAR_PROC`. By default, every execution of SMAC will use the computational 
resources of a node and therefore `PAR_PROC` should be less than the number of 
nodes of the cluster (note that every node can have more than one CPU that
are exploited to speed up the simulation of the ABS model by using Erlang).

We recommend to use the command screen to run the computation that can go on
even if the connections with the Numascale cluster is lost. Moreover, please pay
attention that due to the large number of processes, if the computation is 
started it may be difficult to create new connections to the Numascale cluster.
Hence, we recomment to keep the connection with the cluster alive during the 
running time of the computation.

## Limitations

- Categorical paramters are not yet supported
