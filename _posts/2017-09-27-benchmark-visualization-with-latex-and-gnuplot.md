---
layout:     post
title:      Benchmark Visualization with LaTeX and gnuplot
date:       2017-09-27 18:00:00 +0200
tags:       [benchmark, gnuplot, latex, perl, picotm, visualization]
og-image:   benchmark.png
---

Visualizing benchmark results is an important step in performance testing.
In this blog post, we're going to use [LaTeX][latex], [gnuplot][gnuplot]
and [Perl][perl] to convert raw performance data into a nice-looking PDF
document.

<!-- excerpt -->

We're going to produce a script named `run_benchmark.pl`, which executes
a set of tests and produces a PDF document with a set of nicely formated
diagrams that illustrate the test results. Of course we cannot go through
the script line-by-line, we all the major points are explained.

The produced PDF looks like in [this example][site:results_pdf]. The full
source code is available in picotm's [picotm-perf][github:picotm-perf]
repository.

#### Turning Raw Numbers into Actionable Data

During September your author was busy with the implementation of a new testing
infrastructure for [picotm][picotm]'s unit tests and continuous integration.

Part of this infrastructure is a set of performance tests. Each test run a
predefined workloads and generate a test result that gives information about
the performance of the system. Picotm is a manager for system-level
transactions, which means that the most interesting performance information
is the number of transactions it's able to perform per unit of time.

The first interesting information is the number of completed transactions
per time unit. Our test constantly begins and commits transactions for
a certain amount of time and measures the number of successfully committed
transactions. This is the *throughput* measured in *commits per second.*
The single-thread throughput mostly depends on the transaction manager's
overhead, so this information is relevant during optimization.

Occationally (or maybe frequently) the transaction manager stops, rolls
back and restarts a transaction. This happens for error handling and
conflict resolution. Restarting transactions can have quite a bit of
overhead and minimizing the number of restarts will improve the system's
performance. Therefore the second interesting information is the number
of restarted transactions per time, measured in *restarts per second.*
Minimizing the number of restarts should have positive impact on
throughput.

Our test program `picotm-perf` executes a number of load and store
transactions in the main memory. The resulting output on the console
looks like this:

```
0 60000 10000 0
```

These numbers are

 1. the thread id. *0* as there's only one thread in this example.
 2. The number of milliseconds that the test ran. The equivalent of 60
    seconds in this case.
 3. The number of commited transactions on this thread. 10000.
 4. And finally the number of restarts. There are no restarts because a
    single transaction cannot conflict.

This test program can be executed for various combinations of load and
store operations and I/O patterns. This only depends on the command-line
parameters.

Running all these combinations result is an endless list of such single-line
performance results. This is unintuitive, so we have to feed the data into
further processing stages.

The tools your author has picked are

 * Perl for processing the test-program's information,
 * gnuplot for converting raw data into diagrams, and
 * LaTeX for assembling the individual graphs into a PDF document.

You can find the source code of the script in the
[picotm-perf][github:picotm-perf] repository. If you want to use it for
other performance visualization, all you have to replace is the test
program, and adopt the data parser in the Perl script and the gnuplot
instructions. The rest should be independent from the actual performance
tests.

#### Generating the Raw Data

As mentioned before, test results are generated by `picotm-perf`. To
generate larger amounts of test data, the benchmark script
`run_bechmark.pl` calls `picotm-perf` with different combinations of
parameters for the number of concurrent transactions, the number of
load operations per transaction, the number of store operations per
transaction, and the I/O pattern. It's a set of nested loops in Perl.

~~~ perl
foreach my $pattern ('random', 'sequential') {

    foreach my $nloads (0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100) {

        my $nstores = 100 - $nloads;

        foreach my $ncores (1 .. 4) {

            run_picotm_perf($pattern, $nloads, $nstores, $ncores);

        }
    }
}

sub run_picotm_perf() {

    my ($pattern, $nloads, $nstores, $ncores) = @_;

    return `picotm-perf -T 60000 -P $pattern -L $nloads -S $nstores -t $ncores`;
}
~~~

The example code is mostly self-explaining. It executes the function
`run_picotm_perf()` for all different combinations of I/O patterns, numbers
of loads and stores, and number of concurrent transactions / processor
cores.

The function `run_picotm_perf()` simply runs the external test program
with the given parameters and returns the resulting output string of
the format we've seen before.

#### Generating the Diagram Data

With the raw data at hand, we can now generate the information required
by gnuplot to create the diagrams. The question is what our diagrams shall
tell us.

We are interested in the number of commits and restarts per second for
each combination of I/O pattern and loads and stores. For each of these
combinations, we're also interested in how the values change if we add
or remove concurrent transactions in the workload.

So we want a single diagram for each combination of I/O pattern, loads
and stores, and have the number of transactions change along each
diagram's x axis. The y axis' display the result.

We can build this by replacing the inner-most loop of the previous
example by the function `generate_dat()`, which is shown below.

~~~ perl
sub generate_dat {

    my ($pattern, $nloads, $nstores) = @_;

    foreach my $ncores (1 .. 4) {

        my $result_string = run_picotm_perf($pattern, $nloads, $nstores, $ncores);

        my dat_string

        my $ncommits = 0;
        my $nretries = 0;

        my @lines = split /\n/, $result_string;

        foreach my $line (@lines) {

            # <thread id> <nmsecs> <ncommits> <nretries>
            $line =~ m/^(\d+)\s+(\d+)\s+(\d+)\s+(\d+)\s*$/ or die;

            # Sum up normalized results
            $ncommits += $3 * 1000 / $2;
            $nretries += $4 * 1000 / $2;
        }

        dat_string .= "\n" . "$ncores $ncommits $nretries";
    }

    return dat_string;
}
~~~

This code produces the data for a single diagram, but it's probably
non-obvious what happens.

First of all the function runs the test program 4 times: with 1, 2, 3, or
4 concurrent  transactions. Each run produces a result that is stored in
`result_string`.

For each test run, the result string is parsed line by line for the number
of seconds that the test ran, the number of committed transactions and the
number of restarted transactions. The result numbers are normalized by time
and added up. The result is appended to the string `dat_string` in the
following format.

```
1 123400 0
2 12300  100
3 1200   1200
4 100    12300
```

Each line in `dat_string` now contains the number of concurrent transactions,
the number of committed transactions per second, and the number of restarted
transactions per second. This is what we want to display in each diagram
and it's in a format that gnuplot understands.

#### Generating a LaTeX document

At this point we have performed a large number of tests for different
combinations of I/O patterns, loads and stores, *and* for each combination
we know how the results change when the number of concurrent transactions
increases.

In principle the hardest work is done and all that's left to do is to put
the data into a nice format.[^1] As mentioned before, we use LaTeX and
gnuplot for the data visualization. It's fair to say that both programs
are fantastic in terms of features, but less than optimal in terms of
usability. Both have a file format that is rather inelegant and breaks
easily. It's something most people write and debug once, and then try to
leave alone.

The input for gnuplot is a format that describes the resulting diagram. It
allows to connect the input data, which we generated in the previous step,
to the individual axes, and allows for configuring colors and style of the
graph. The x axis on each of our diagrams shows the number of concurrent
transactions. The y axis on the left shows the number of commits per second;
the y axis on the right shows the number of restarts per second. Running
`gnuplot` on each set of input data produces the corresponding image of
the diagram.

We won't go through the details of the LaTeX document source code here,
but your author likes to highlight the use of LaTeX' gnuplot package. The
gnuplot tool typically gets it input from an external file. One can run
gnuplot to generate an image file from such an external file and then
reference the image's filename in the LaTeX source code.

The gnuplot package for LaTeX simplifies this significantly. It allows for
storing the gnuplot diagram description directly within the LaTeX document.
The gnuplot tool is run automatically by the package while processing the
LaTeX document and the resulting image is included where the description
was located before. This does away with all the file-name juggling. Your
author appreciated the gnuplot package a lot while working on the benchmark
script.

The rest of the LaTeX document is set in the standard format, which is
widely documented on the web. The end result is a file named `results.tex`,
which can be converted into a PDF document.

#### Generating a PDF Document

The PDF output is produced by the LaTeX tool `pdflatex`. Running LaTeX
tools seems just as arkward as writing LaTeX documents. We have to run
`pfdlatex` 3 times on `results.tex` to get a correct PDF document. The
benchmark script does this internally.

#### Ideas for Future Improvements

Finally we have a [PDF document][site:results_pdf] named `results.pdf`. With
picotm's current implementation the performance numbers don't have much
meaning yet, but from here we can easily change and optimize and compare
different versions against each other.

For the benchmark script itself, your author has a number of ideas.

Of course, more combinations of our test parameters are always possible.
Adding these is a low-hanging fruit.

Since we already have a lot of performance data, we could archive it for
future use. Then a user could point the script to a certain directory with
test results and have it generate performance diagrams from these.

With the old results stored away, the script could also compute the
differences between different test runs, visualize them, and automatically
warn about performance regressions above a certain threshold.

#### Summary

In this blog post, we've developed a script for creating and visualizing
performance data.

 - The actual test utility is an external program that generates performance
   data for different simulated workloads.
 - The test result is returned in the test utility's output.
 - The benchmark script is written in Perl, which is well-suited for
   text processing.
 - The benchmark script runs the test program with various combinations
   of workload parameters and converts the resulting performance data
   to a format that is compatible with gnuplot.
 - The gnuplot tool generates images of diagrams from raw data.
 - The benchmark script assembles all gnuplot-generated diagrams into
   a LaTeX source file.
 - The LaTeX tool `pdflatex` generates the final PDF document from the
   LaTeX sources.

If you like this blog post, please subscribe to the RSS feed, follow on
Twitter or share on social networks.

#### Footnotes

[^1]:   One could argue that collecting the data is easy, but writing
        LaTeX source files is hard. :p

[github:picotm-perf]:   http://github.com/picotm/picotm-perf
[gnuplot]:              http://www.gnuplot.info/
[latex]:                http://www.latex-project.org/
[perl]:                 http://www.perl.org/
[picotm]:               http://picotm.org/
[site:results_pdf]:     {{ site.baseurl }}/assets/pdf/results.pdf