---
title:  "CHTC for Statistics Crash Course"
date:   2021-10-07 15:56:24 -0500
description: "Using high throughput computing resources at UW Madison"
tags: [computing, cluster, UW Madison]
---

These notes collect some tips for using the [Center for High Throughput
Computing (CHTC) Cluster](https://chtc.cs.wisc.edu/) at UW Madison, especially
those related to statistics research. For reference, the full documentation is
available [here](https://htcondor.readthedocs.io/en/latest/); there are many
more possibilities than we discuss here. You can follow along with the code in
[this repository](https://github.com/krisrs1128/chtc_crash_course) -- let us
know if you find any mistakes or would like to share any other tricks!

### Why use a computer cluster?

There are three types of statistical computing problems where a cluster is
helpful,

* Parallel Simulations: It is standard in statistics research to understand how an algorithm behaves across a range of data or hyperparameter regimes (e.g., noise levels, regularization parameters). While in principle these experiments can all be run sequentially on a laptop, whenever there are a large number of configurations to consider, it will save time to run them in parallel. In theory, processes can be parallelized on a single laptop. Practically, however, only a few processes can be run at a time without overloading a typical machine. On a computer cluster, we can run many simultaneous jobs without any issues.
* Long-Running Jobs: Sometimes we only need to run one script, but it takes a long time to complete. There have been times where I've kept a laptop running continuously for a few days, but this is always a little nerve-wracking and can distract from other tasks. If the long-running job is run on a CHTC machine instead, we can check on it whenever it's complete without having it always visible in the background. Also, when more resources are available, the job will likely finish more quickly.
* GPU Jobs: Most of us don't have GPU cards on our personal computers. CHTC has a great GPU Lab, where we can checkout a GPU whenever we need one.
* Large Datasets: There are some datasets that we can't store on our laptops. A reasonable workflow is to first develop code on a subset of the data on a personal machine. Once we trust that our code works on the subset, we can attempt running on the full dataset using CHTC.

### Logging onto the cluster

After   [requesting](https://chtc.cs.wisc.edu/uw-research-computing/form) an
account from CHTC, you can login to a remote node using

{% highlight bash %}
ssh yournetid@submit2.wisc.edu
{% endhighlight %}

Note that if you are off campus, you will first need to login to the
[GlobalProtect VPN](https://kb.wisc.edu/helpdesk/page.php?id=105619). Otherwise,
the cluster machines will not be reachable.

#### Tip: Aliases

You can create an alias to login to the cluster more quickly. On your personal
machine, you should have a folder `~/.ssh/` containing a few public and private
key files. If you `scp` the `id_rsa.pub` public key into `~/.ssh/` folder on the
remote machine, then you will be able to login using an alias after you,

* Use `chmod 600` the remote copy to ensure that it is not too visible.
* Add the following lines to your `~/.ssh/config` on your local machine.

{% highlight bash %}
host your_alias
  HostName submit2.chtc.wisc.edu
  User your_netid
{% endhighlight %}

For example, I set my alias to `chtc`, which lets me login to the cluster using
`ssh chtc`.

Once you log in, you can browse around and view a few files -- you should have
access to create files in a folder like `/home/your_net_id` and
`/staging/your_net_id`. The first folder is where routine code can be kept. The
second is where data should be placed just before launching a job (and remove
shortly afterwards).

### Setting up a simple job

A CHTC job is specified by a submit file. It describes the compute resources,
code, data, and logging instructions needed to complete the task. It is
essentially a dictionary written as a text file, whose key-value pairs conform
to conventions set by CHTC. Next, we'll review some of the most commonly used
keys. Examples using these keys in real compute jobs are available
[here](https://github.com/krisrs1128/chtc_crash_course) and discussed further
below.

_Specifying resources_

Computer clusters allow us to checkout more resources than we would have on our
own. To keep it from becoming a free-for-all, it's necessary to estimate how
much resources we need to complete a task (the more we ask for, the longer we
have to wait before the job can start).

* `request_memory`: How much memory does the job need? This is usually determined by the size of the largest matrix or data.frame that needs to be loaded into R or python. Example values are `16GB` or `500MB`. You can request much more memory than we have on a laptop -- asking for `128GB` of memory would be a lot but not totally out of the ordinary.
* `request_disk`: How much disk space do we need for storing data and saving outputs? This is usually determined by the size of the dataset that is being processed, though it can also depend on the size of the saved output / logs. Note that a job can require a lot of disk but only a little memory -- it may work off of a large dataset, but only processes small batches at a time. Typical values are `64GB` or `12GB`.
* `request_cpus`: If you have a process that can utilize several cores, then it can help to make use of several cpus. By default, though, I just set this to 1, since most of the parallelization can happen at the cluster job level.
* `queue`: How many parallel jobs should we run? More on parallelization below.

_Specifying code_

Cluster code needs to be more or less automatically runnable; we can't rely on
interactively sending commands one after another like we might on our own
laptops. We also need to ensure that the environment on the cluster compute node
is close enough to our local environments; i.e., all packages and environmental
variables that we require locally need to be accessible remotely.

* `executable`: This gives the name of a shell script containing all the commands to be run in the CHTC job. If we weren't using a computer cluster, it would be the only script that needs to be run.
* `environment`: We may want to use environmental variables within the executable. This is especially useful when looping over parameter values in a parallel array of jobs.
* `docker_image`: If we need packages that are hard to directly install on CHTC, it can be convenient to build a docker image on your local machine and run the CHTC job on a public version of that image. Note that for this key to be read, we first need to specify `universe=docker`

_Specifying data_

Whatever data we work with locally needs to be accessible on the remote machine.

* `transfer_input_files`: The absolute or relative path to a file (or comma separated list of files) that should be transferred to the compute node when a job begins. Note that you can also transfer code repositories using this command. I typically zip everything I need into a single file (`tar -zcvf all_files.tar.gz all_files/`) and then point the argument to this file. Note that all files in the same directory as the submit file will automatically be transferred to the working directory of the compute job.
* `requirements = (Target.HasCHTCStaging == true)`: Instead of directly transferring input files, we can keep our files unzipped in a `/staging` directory and then copy them over within the executable script. For this approach to work, though, the compute node needs access to the `/staging` filesystem. This flag ensures this access is supported (not all compute nodes can access `/staging`). Note that several requirements can be changed together using `(requirement 1) && (requirement 2)`.

_Specifying logs_

Logs are helpful for debugging. They also tell how much resources a job computed after-the-fact, which can be helpful for updating resource requests in future jobs. Note the use of `$(Process)` in the filenames below -- this makes sure that log files are not overwritten in parallel runs.

* `error`: The name of the error file.
* `output`: The name of a file that will include any messages printed out by the executable script.
* `log`: This name of a file describing how many resources are used by the job.

Putting this all together, here is an example submit file,

{% highlight bash %}
universe=docker
docker_image=ubuntu/focal

executable=basic.sh
transfer_input_files=file1.csv,file2.csv

# requesting resources
request_memory=10MB
request_cpus=1
request_disk=1MB

# log files
error=basic-$(Process).err
output=basic-$(Process).out
log=basic-$(Process).log
queue
{% endhighlight %}

and this is the accompanying executable.

{%highlight bash %}
#!/bin/bash
ls -lsh
wc file*
sleep 30s
{% endhighlight %}

These files are downloadable
[here](https://github.com/krisrs1128/chtc_crash_course/tree/main/basic_job). The
job can be submitted to the cluster using the command,

{%highlight bash %}
condor_submit basic.submit
{% endhighlight %}

What do you think is output by the job?

Finally, we note that any of the parameters specified in the submit file can be
overwritten at submit time as an argument to `condor_submit`. This is useful
whenever several similar executables can be run using the same background
resources and input. For example, we can use

{%highlight bash %}
condor_submit train.submit executable=model1.sh
condor_submit train.submit executable=model2.sh
{% endhighlight %}

to run two separate models,`model1` and `model2`, using the same resources,
specified by `train.submit`.

### Debugging a Compute Job

It would be a miracle if a compute job worked on its first submission. More
commonly, we submit small preliminary versions of a job (requesting fewer
resources) until we are confident that the script works. The faster we can debug
errors, the sooner we can get results.

The first thing to do when debugging a failed cluster job is to view the log
files. Errors that stopped the compute process will be saved in the `error` log
specified in the submit script, but useful hints are often available in the
messages saved in the `output` file. If you are running a parallel job, it might
be useful to read the last few lines of many error logs. For example, the command

{% highlight bash %}
tail -n 20 *.err
{% endhighlight %}

will print the last twenty lines of every error log in a directory.

Besides analyzing the logs and resubmitting a cluster job, here are some
utilities that streamline the process,

* `condor_q`: This shows all the current jobs, their IDs, and whether or not they are running.
* `condor_submit -i`: This runs an interactive version of a submit file. Once resources are checked out, you will have interactive access to an environment that looks exactly like what the original job's environment would look like. If every line of the executable works in an interactive job, then it will also work in a non-interactive submission.
* `condor_ssh_to_job`: This will let you log into a currently running compute node and watch the directory as the process is running. This is especially useful if a job doesn't fail right away but you know there are certain outputs that indicate the job will eventually fail. This way, you can check the output and kill a doomed job without waiting for the process to officially fail.
* `condor_rm`: If we realize a bug and want to remove a job, we can call `condor_rm job_id`, using the `job_id` printed by `condor_q`. If we want to remove all the jobs, we can use `condor_rm --all`.
* `condor_q -hold -af HoldReason`: If our job is being held for some reason (e.g., we have a typo in the input files or ran out of memory), we can inquire into the reason using this command, which takes the job's ID as its argument.

Finally, if we had asked CHTC to run using a docker environment, we can simulate
the CHTC environment using on our local machine. This works because the Docker
VM is  identical in both computers. However, CHTC doesn't use the ordinary
Docker run command, so the filesystem and permissions will be slightly
different. To account for the discrepancy, we can use

{% highlight bash %}
docker run --user $(id -u):$(id -g) --rm=true -it   -v $(pwd):/scratch -w /scratch   YOUR_DOCKER_IMAGE_ID /bin/bash
{% endhighlight %}

locally to exactly mimic the CHTC environment.

### Pattern 1: Parallel Runs

One of the most common patterns in statistics research is to launch a large
number of relatively small jobs (15 - 120 minutes) to see how an algorithm
varies over different dataset or hyperparameter configurations (e.g., matrix
aspect ratios, latent dimensionalities, noise levels, or regularization
parameters). This is easily supported by the `queue` parameter in CHTC jobs. The
main idea is,
* Write code that takes data or algorithm parameters as arguments.
* Create a table with all the parameter configurations of interest.
* Request a number of jobs equal to the number of desired configurations.
* For the $i^{th}$ job, read in the $i^{th}$ row of the table and use it to provide arguments to the generic code above.

For example, suppose we have a table with 5 configurations,

```
A,B,C
1,2,3
2,3,4
5,6,7
8,9,10
11,12,13
```

then we can queue 5 jobs and pass each job's number in the queue as an
environmental variable. Specifically, the submit file, we add

{% highlight bash %}
transfer_input_files=configurations.csv
environment = "id=$(Step)"
queue 5
{% endhighlight %}

and we refer to the ID environmental variable in the executable,

{% highlight bash %}
Rscript -e "rmarkdown::render('parallel.Rmd', params=list(id=$id))"
{% endhighlight %}

which is designed to read the associated row from the `configurations.csv`. This
command used an R process, but the same idea works in python or julia with the
appropriate argument parsers.

A full example of this pattern can be found
[here](https://github.com/krisrs1128/chtc_crash_course/tree/main/parallel_job).
Note that the submit script pulls from a [rocker
image](https://hub.docker.com/r/rocker/tidyverse), because the executable needed
to run R with the `rmarkdown` package installed.

## Pattern 2: GPU Jobs

GPU jobs are almost exactly like ordinary jobs, except that they need to
explicitly request nodes with GPUs. This is done using the `request_gpu` flag.
We usually set `request_gpu=1`, though it's also possible to run a job with
multiple GPUs (only do this if your code actually knows how to load data into
several GPUs, though). Some other useful commands are,
* `+GPUJobLength`: Will the job be done in a couple of hours? Will it need more than 24? This can be set to either `short`, `medium`, or `long` to ensure that the job won't be kicked out too early.
* `+WantGPULab`: This is a shared GPU resource that isn't included by default in CHTC jobs. This is almost always set to `true`, since it helps jobs launch more quickly.
* `+wantFlocking`: This is another shared GPU resources, like the GPU Lab. This is also usually set to `true`.

Some code will work on some GPUs and not others (e.g., a run with large batch
size might need extra GPU memory). We can restrict the GPUs that are considered
for a job by using the same `requirements` argument described above. For
example,

{% highlight bash %}
requirements = (CUDAGlobalMemoryMb > 4000)
{% endhighlight %}

will only run jobs on compute nodes with at least 4GB GPU memory. To see whether
a request has a chance of being fulfilled in the near future (or ever...if you
are like me and request resources that do not exist), we can review the
cluster's GPU resources,

{% highlight bash %}
condor_status -compact -constraint 'TotalGpus > 0'
{% endhighlight %}

Additional constraints can be used to filter this list. For example, to see
nodes with at least CUDA 6.0  installed and 12GB of memory, we could use

{% highlight bash %}
condor_status -compact -constraint 'TotalGpus > 0' -constraint 'CUDACapability > 6' -constraint 'CUDAGlobalMemoryMb > 12000'
{% endhighlight %}

A full example of this pattern can be found
[here](https://github.com/krisrs1128/chtc_crash_course/tree/main/gpu_job). The
script runs a file called `gpu.py`, which is only a few lines,

{% highlight python %}
import torch

device = torch.device("cuda")
x = torch.zeros((10, 10)).to(device)
print(x)
{% endhighlight %}

but this is enough to see that we can put the tensor `x` onto a GPU. Indeed, the
`output` file confirms that `x` had been on the GPU,

{% highlight python %}
tensor([[0., 0., 0., 0., 0., 0., 0., 0., 0., 0.],
        [0., 0., 0., 0., 0., 0., 0., 0., 0., 0.],
        [0., 0., 0., 0., 0., 0., 0., 0., 0., 0.],
        [0., 0., 0., 0., 0., 0., 0., 0., 0., 0.],
        [0., 0., 0., 0., 0., 0., 0., 0., 0., 0.],
        [0., 0., 0., 0., 0., 0., 0., 0., 0., 0.],
        [0., 0., 0., 0., 0., 0., 0., 0., 0., 0.],
        [0., 0., 0., 0., 0., 0., 0., 0., 0., 0.],
        [0., 0., 0., 0., 0., 0., 0., 0., 0., 0.],
        [0., 0., 0., 0., 0., 0., 0., 0., 0., 0.]], device='cuda:0')
{% endhighlight %}
