# fire-parallel

The program `fire-parallel` was introduced into ldmx-sw in v4.5.7 and can be used to run multiple copies of the same `fire config.py` across a handful of local cores on your computer.
Under-the-hood `fire-parallel` uses [GNU parallel](https://www.gnu.org/software/parallel/sphinx.html) which has a
specific structure for defining the different command-line arguments that spawn the different copies of the process.

The most basic example is running the same simulation job but with different run numbers to seed the random number generation.
If `fire config.py 1` creates `events-1.root`, then you can
```
fire-parallel config.py ::: 1 2 3 4 5
```
to do the five different runs all in parallel on the available cores on the machine.
You should look at the GNU parallel documentation linked above for all of the details, it is a good resource and you can get much fancier than this basic example!

~~~admonish note title="Note on Performance" collapsible=true
Since all of these jobs are attempting to write to files on the same disk, we do not see a direct N-fold speed-up when using N cores, it seems like we roughly see a limit of a 2x speed-up from our basic testing.

This is okay as a first attempt to parallelize ldmx-sw processing and lines up with the number of cores assigned worker nodes at a few of the clusters that LDMX collaborators have access to.
~~~

~~~admonish warning title="Careful!"
The `fire-parallel` script does not check to make sure the arguments you provided or the configuration script being run are distinct jobs. Be careful to make sure you don't repeat run numbers and/or write data from different jobs to the same file. This will lead to confusing results that are not physical!
~~~

### Passing Options Directly to GNU Parallel
The options (starting with `-`) provided to `fire-parallel` are assumed to be options to `fire-parallel` or
the configuration script. If you want to pass options to GNU `parallel` itself (for example, using `-j` to limit
the number of cores it should use), you should store them in the `PARALLEL` environment variable.
```
denv config env copy PARALLEL="-j2"
denv fire-parallel config.py ::: 1 2 3 4 5 # only does 2 at a time
```

### Combining Results
Often the first thing you want to do after running multiple jobs in parallel is to combine the results into
a single output file to look at.
Assuming the output file is small enough (for example, its just histograms or very few events), you can
use ROOT's `hadd` program to merge several ROOT files together.
```
denv hadd output.root input-1.root input-2.root input-3.root ...
```
You may need to enter the `denv` interactively if you have too many ROOT files to pass on the command line.
```
denv
hadd output.root input-*.root
```

### Past ldmx-sw Versions
`fire-parallel` is just a shell script, so you can use it as inspiration to write your own specific one!
It will work with prior versions of ldmx-sw if you copy it into your usage space. For example,
```
# using a version of ldmx-sw that did not have fire-parallel
denv init ldmx/pro:v4.4.7
# download fire-parallel into this workspace
wget https://raw.githubusercontent.com/LDMX-Software/ldmx-sw/refs/heads/trunk/Framework/app/fire-parallel
# run fire-parallel from here
denv ./fire-parallel ...
```
