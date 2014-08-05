Make-based RNA-seq Analysis Workflow
====================================================

Prerequisites
--------------------------------------

This guide assumes the reader has some basic unix knowledge.
At a  minimum the user should have a conceptual understanding of
the unix command line, unix file system, and secure shell.

Intro
---------------------------------------

Traditional bioinformatic workflows are implemented using
either scripting (e.g. bash, python, etc.) or a GUI program
such as Galaxy to compose different analysis tools together.

While galaxy works well for simple analysis, it requires
a high amount of expertise to integrate custom and unsupported 
analysis programs into the galaxy workflows--meaning that it
is often simpler to script or to just execute a workflow by
hand. However, scripting or executing a workflow manually 
is difficult since it requires the developer to imperatively
encode the workflow from start to finish; iteratively
developing such a pipeline is difficult since it creates
an organizational and computational 
bottleneck. For larger datasets, scripting is further
complicated by the need to use high performance computing
infrastructure.

We were able to simplify the construction of custom
bioinformatic workflows by keeping track of input files 
and output files of each step of the analysis to implicitly 
determine the order of execution. We use a common program 
called GNU Make (henceforth called simply Make) which is found
on nearly every unix computer. 

Make is:

* _Reproducible_, Make ensures that the workflows output
is actually a representative by the most up-to-date code.
In addition, make file supports start-to-finish execution
of the complete workflow.
* _Simple_, Make determines the workflows execution order 
on it's own, meaning that the developer does not need to
worry about complicated for loops.
* _Efficient_, Make only performs computations that are
required to produce the desired outputs. In addition
make can execute the pipeline in parallel automatically.
* _Robust_, Make can usually detect when a program crashed
and stop downstream execution.

However Make is designed for software compilation, not
bioinformatic pipelines. At times, Make using make will be a
bit awkward. 

Make Basics
--------------------------------------

Make at it's core is very simple. Simply by tracking 
dependencies a lot of workflow programming becomes implicit.

### Terminology

Make is pretty simple. 

<dl>
<dt>Dependency (a.k.a Prerequisite)</dt>
<dd>A dependency is an input file needed to construct a target.
Dependencies can, themselves, be targets.
</dd>
<dt>Target</dt>
<dd>A target is the file that will be generated by a rule. Make
can only generated one target per rule.</dd>
<dt>Command</dt>
<dd>The shell command used to construct the target. Commands must
be offset with a tab.</dd>
<dt>Rule</dt>
<dd>A rule is a single step in a workflow. It consists of a list of 
dependencies, a target, and a command. </dd>
<dt>Makefile</dt>
<dd>A file named `Makefile` that stores multiple rules. Makefiles
must be saved as `makefile` or `Makefile`.</dd>
<dt>Make Directory</dt>
<dd>The directory where the Makefile lives. The `make` command can
only be called from this directory.</dd>
</dl>

### A simple make file

Make a new directory and save the following code
in a file called `Makefile`:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~`

# this rule creates a file called Hello.
Hello : ;
echo Hello > Hello
a
# this rule creates a file called Word.
World : ;
echo World > World

# This rule combines the files Hello and World.
# n.b. Hello and World are dependencies for the rule below.
HelloWorld : Hello World ;
cat Hello World > HelloWorld

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The target `Hello` can now be constructed (or "maked") by executing the following command 
from the Makefiles directory:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ make Hello
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Make will output the commands used to construct `Hello`, i.e.:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
echo Hello > Hello
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The target `HelloWorld` can be constructed by the command:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ make HelloWorld
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Then Make executes two commands.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
echo World > World
cat Hello World > HelloWorld
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Notice how Make does not reconstruct the target `Hello`. 
Make is lazy and avoids redundent computions.

To demonstrate, let's make `HelloWorld` again.

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ make HelloWorld 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Make outputs:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
make: Target up-to-date
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Meaning that the target is younger than it's dependencies in terms 
of the dependency files modification timestamp. If the dependency
is a target, make ensures that the dependency is also up-to-date.

As an exercise, edit the file hello and make `HelloWorld` again.

Make only computes what it needs to compute. So that previous step
will only reconstruct the target `HelloWorld` again. On a larger 
scale, for example, consider how a Make based workflow would reaact
to switching to a more up-to-date genome. Make will see that the 
genome files are younger than the targets that depend on these files and
reconstruct only those targets. Make won't bother to run any read
level quality control operation since these targets are completely independent
of the genome data. Make knows exactly which steps of a workflow 
to run and not to rerun.

Make is able to declaretively encode a bioinformatic workflow quite
easily, simply by tracking the input and output files that a bioinformatic
program uses or produces.

### The three most important rules to bioinformatic Make files.

Make is powerful and flexible. From experience, it seems that make
work's best when you follow the commandments:

1. Though shalt not set targets to directories.
2. Though shalt not use recursion.
3. Though shalt not use some obscure Makefile extensions.
4. Though shalt not modify a dependency within rule's command.
5. Though shalt not modify a dependency's dependency within rule's command.
6. Though shalt not modify a dependency's dependency's dependency within rule's command.
7. etc.

Rules 4 through n can be summarized as: No interdependent relationships between
targets and dependencies. That is, in math lingo, Make works only for computations
that can be reduced to an acylic graph. 

### Make, High Performance Computing, Remote Execution

By using a scheduler such as `slurm` or a remote access tool 
such as `ssh` we are able to outsource work to other computers.

Some very expensive proprietary schedulers are even able to use makefiles as
job submission scripts.


The first command: 

As long as a command "blocks" i.e. does not terminate 
until it is finished executing it can be used within the workflow. 
This means we can't simply submit "batch scripts" to the scheduler,
instead we submit interactive jobs. Interactive jobs work identically
to connecting to a remote server via ssh. However, there are some
syntaxual caveats to be aware of.

Namely piped commands need to be enclosed in single quotes in order
for the entire sequence of computations to take place remotely.

Consider the following example: 

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ ssh -t myserver date > remote_time # example 1
$ ssh -t myserver 'date > my_time'   # example 2
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The first command:

1. Connects to `myserver` 
2. On myserver executes the command `date` 
3. Sends the output of `date` back to your computer.
4. Your computer sends the output to a file named `remote_time`.

I.e. the output is sent directly back at us.

While the second command:

1. Connects to `myserver`
2. Executes the command `date`
3. Saves the time as `my_time` on `myserver`'s filesystem.

I.e. the output of the command `date` never leaves the server.

But on an HPC cluster with a shared filesystem both my_time
and time will appear in your current directory. The first 
command is significantly less efficient since the data `date` produced gets
rerouted back to the local computer and then sent over the file system. In
the second command, `date` bypasses the local computer and gets sent directly to
data. In high throughput situations, the first example is disastrous and has 
resulted in the workflow crashing. It's best to avoid piping altogether.

Using raw ssh gives us the ability to have certain steps of a workflow
execute on very particular computers (e.g. your very laptop or
the labs web server). This is useful for less computational intense 
work that requires software that is difficult to get working on a 
HPC cluster; e.g., graphing and gene set analysis software. Note that,
you need to mount the remote file system on your local computer or
you must move input files need to be moved to the remote server
explicitly and output files need to be moved back. This is easier
said than done. It's probably simplest to keep two make files:
one Makefile for high throughput analysis and second Makefile for
the analysis that can take place on a personal computer.

Otherwise, for UWM's Avi cluster, we prepend `salloc <scheduler PARAMS>` to the command
to execute remotely. E.g. the following will reserve a node with 8 cores and 22gb
of memory and then execute `my_command`.

~~~~~~~~~~~~~~~~~~~
-alloc -c 8 -N 1 --mem 22000 srun my_command
~~~~~~~~~~~~~~~~~~~

Note this is a single "conjugate" command. `salloc` executes `srun` and `srun` executes `my_command` 
on the resources `salloc` requisitioned. 

### Organizing the experiment #########

Another potential caveat of the Makefile approach is that
it requires you to organize your experiments data files in a
wae that is--well--very organized. See the example Makefile
which goes in depth for a generic RNAseq design that analyses
multiple experiments in parallel.

In short, files need to be predictable places. As you move down
the directory tree you should move from more "general files" to
more specific files.

As a rule of thumb, if you're Makefile has a lot of experiment
specific rules or a lot of funky string manipulation going on, chances
are moving around a few files will make a simpler Makefile.

### Writing Rules #####################

Makefiles should be easy to read and well documented. The great
thing about using Makefile is that an entire workflow can be
encoded into a single file.

The order of rules doesn't matter, but they should appear in the
same order they would execute.

E.g.:

1. Rule for Downloading a Sample form the sequencer
2. Rule for generating a FastQC report
3. Rule for cleaning out adapters
4. Rule for generatinng a FastQC report for cleaned adapters.
5. Rule to align reads to genome.

But rules 2 and 4 are identical. It's best to move more general
rules to either the start or end of the Makefile. This means
certain bioinformatic operations become implicit based on the 
dependency requested. Here are some examples of general rules on
alignment files: 

TODO Replace working Makefile

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# Convert Serial (Sam) aligments to Binary (Bam) alignments
     *.sam : *.bam ;
     samtools view -b $< > $@ 

# Sort a bam file by locus
     *.sorted.bam : %.bam ;
     samtools sort $< $@ 

# Index a bam file
     *.sorted.bam.bai : *.sorted.bam
     samtools faidx $<
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The percent sign is a wild card that works similar to `*` wildcard in shell scripts.
`$<` is a macro for the first dependency. `$@` is a macro
for the current target. 

Suppose we have a file called `test.sam` then running:

~~~~~~~~~~~~~~~~~~~~~~~~
$ make test.sorted.bam.bai
~~~~~~~~~~~~~~~~~~~~~~~~

Will execute the following commands:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
samtools view -b test.sam > test.bam
samtools sort test.bam test.sorted.bam
samtools faidx test.sorted.bam
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This order of execution gets determined by searching for a
dependency's dependency until Make finds a rule where all
dependencies are satisfied. Make matches "wildcard" rules
as a last resort, that is make defaults to rules for targets
without wildcards whenever possible. 


Here is an example of a more complicated rule:

~~~~~~~~~~~~~~~~
#Align to genome via tophat via the script scripts/tophat.sh
samples/%/tophat/accepted_hits.bam : \
	$(genome_idx) \
	$(genome_gtf) \
	scripts/tophat.sh \
	samples/%/cleaned_reads/r1.fq.gz \
	samples/%/cleaned_reads/r2.fq.gz ;
	salloc -c 8 -N 1 --mem 21000 -J tophat srun \
	       scripts/tophat.sh $(@D)

~~~~~~~~~~~~~~~~~~~

Since there are a lot of dependencies, we keep the code readable by 
spanning the list of over multiple lines and escaping the new line
character with a `\`. The scheduler-related portion of the command
has it's own line. Finally, the actual command is offset by an additional 
tab. 


### Make Variables and String Operations


