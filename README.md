The Bingo Interactive Alarm Prioritization System
=================================================

This is the public release of the Bingo interactive alarm prioritization system described in the paper
presented at PLDI 2018: [User-Guided Program Reasoning Using Bayesian Inference](https://dl.acm.org/citation.cfm?id=3192417).
This readme file describes the system workflow, its constituent scripts, and instructions to use them.

**NOTE 1:** The Bingo system is agnostic of the underlying static analysis. Per our description in the PLDI 2018 paper,
we nominally assume that the analysis is expressed in Datalog. However, the code we provide here can more generally work
with any analysis that can be conceptualized as consisting of derivation rules which are repeatedly instantiated until
fixpoint. In particular, we require that the analysis produces a set of output conclusions (a subset of which are
reported to the user as warnings) and a derivation graph which connects them together. In the terminology that follows,
the alarms are reported in a file named `base_queries.txt`, the analysis rules are listed in a file called
`rule_dict.txt`, and the derivation graph is contained in a file called `named_cons_all.txt`.

**NOTE 2:** The PLDI 2018 paper describes the operation of Bingo with two backend static analyses: a datarace analysis
for Java programs and a taint analysis for Android apps. Running these specific analyses is a somewhat involved process.
Furthermore, this intial analysis run is unimportant to anyone wishing to port Bingo to a new analysis of their choice.
In this code distribution, we therefore only include the portion of the workflow after the analysis run has been
completed, the alarms have been reported, and the derivation graph has been extracted.

Building Bingo
--------------

Bingo is mostly written in Python. A few small performance-critical pieces of code have been written in C++. The core of
Bingo crucially depends on the LibDAI inference library, which the build script will clone itself. Ensure that you have
the Boost C++ libraries and the gmpxx wrapper to the GMP library installed. On Ubuntu, the appropriate dependencies can
be installed by running:
```
sudo apt install libboost-dev libboost-program-options-dev libboost-test-dev libgmp-dev
```
Now, build Bingo by running:
```
./scripts/build.sh
```
from the main Bingo directory.

System Workflow
---------------

Bingo is architected as a sequence of small scripts intended to be run one after the other. We assume that all data
related to the program being analyzed are stored in a directory named `PROBLEM_DIR`. These scripts comprising Bingo
communicate by creating several files in `PROBLEM_DIR` whose purpose we will now describe. We have provided an example
`PROBLEM_DIR` directory for the `lusearch` benchmark, available by navigating to the path `bingo/examples/lusearch/`:
these are the warnings generated by a static datarace detector when applied to a program from the Dacapo benchmark
suite.

0. **Analysis Run:** After the analysis run is complete, we assume that the `PROBLEM_DIR` directory contains the
   following files:

   1. `base_queries.txt`. This file contains the list of alarms reported by the analysis. Note that each alarm is
      listed in the form of a tuple, rather than as the warning message string presented to the user. Each tuple is of
      the form `RelName(v1,v2,v3,...,vk)`, for some relation name `RelName`, and where `v1`, `v2`, ..., `vk` are the
      fields of the tuple. However, note that this format is merely conventional: the only concrete requirements that
      Bingo makes are that each tuple is represented by a globally unique string, and that there are no spaces in this
      string.

      See the `base_queries.txt` file included with the `lusearch` example. This file contains a list of tuples of the
      form `racePairs_cs(x,y)`. Each of these tuples indicates a possible datarace between source location `x` and
      source location `y`. For example, the alarm tuple `racePairs_cs(15562,15645)` may be decoded into the following
      human-readable warning message: "There may be a datarace between the field accesses at line 40 of the file
      `org/apache/lucene/analysis/LowerCaseFilter.java` and at line 685 of the file
      `org/apache/lucene/analysis/Token.java`." The file `racePairs_cs.txt` contains the corresponding human-readable
      warning messages. In general, how the alarm tuples map to the actual warning message is an analysis-specific
      design decision, and irrelevant to Bingo's ranking algorithm.

      Of the 237 alarms in the `lusearch` example, only the four alarms listed in `oracle_queries.txt` are real bugs. Of
      course, when analyzing a new program, the user is unaware of which alarms are real bugs and which are false
      positives: we can only provide `oracle_queries.txt` for the example problem because we have laboriously triaged
      each of the alarms.

   2. `rule_dict.txt`. Each derivation rule of the analysis is assigned a name, conventionally of the form `Rn`, for
      some number `n`. The `rule_dict.txt` file contains a mapping between the rule name and the underlying rule. Each
      line of this file contains an element of this mapping in the form `Rn: rule description`. This file is for human
      consumption only, and is not strictly required, and the format is not strictly regulated.

   3. `named_cons_all.txt`. This file contains the derivation graph. Each line of the form
      `Rn: NOT h1, NOT h2, ..., NOT hk, t`. Here `Rn` is the name of the rule, which can be instantiated to produce the
      concrete constraint (not `h1` or not `h2` or ... or not `hk` or `t`). This is equivalent to writing (`h1` and `h2`
      and ... and `hk`) => `t`. In other words, the rule `Rn` can be instantiated to produce the output conclusion `t`
      from the input hypotheses `h1`, `h2`, ..., `hk`. We assume that tuples which are not the conclusion of any rule
      instantiation are facts about the program which have been supplied as input to the analysis (also called the EDB).

   4. `rule-prob.txt`. This file contains the mapping from rule names to their probability values. Each rule is given a
      a line of the form `Rn: pn`, where `Rn` is the name of the rule and `pn` is its probability value. The next
      `build-bnet.sh` step will, by default, assign the probability value 0.999 to rules which do not appear in
      `rule-prob.txt`. As a result, an empty `rule-prob.txt` is completely acceptable:
      `cat /dev/null > $PROBLEM_DIR/rule-prob.txt`. In fact, our experiments in the PLDI 2018 paper were conducted with
      an empty `rule-prob.txt` file.

1. **Building the Bayesian Network (`build-bnet.sh`):** The `build-bnet.sh` script converts the constraints in
   `named_cons_all.txt` into a Bayesian network. We provide two versions of this script. Either run:
   ```
   ./scripts/bnet/compressed/build-bnet.sh $PROBLEM_DIR noaugment_base $PROBLEM_DIR/rule-prob.txt
   ```
   or run
   ```
   ./scripts/bnet/build-bnet.sh $PROBLEM_DIR noaugment_base $PROBLEM_DIR/rule-prob.txt
   ```
   The first script applies the chain compression procedure described in Algorithm 3 of our PLDI 2018 paper, but the two
   two commands are otherwise interchangeable. We recommend the use of the compressed version unless it is specifically
   inapplicable, for example when learning rule probabilities with the Expectation Maximization algorithm.

   The meaning of the arguments are as follows:
   1. `PROBLEM_DIR`. The location of the problem-specific files.
   2. `noaugment_base`. Augmentation was one of the options we experimented with while developing Bingo. Our hope was
      that it would significantly improve accuracy at a small cost in inference time. In practice, we found no
      perceptible increase in accuracy but dramatically higher costs for constraint pruning and for inference. We
      therefore recommend that this option be turned off.
   3. `$PROBLEM_DIR/rule-prob.txt`. Location of the file containing the rule probabilities.

   The script will place the Bayesian network in the directory `$PROBLEM_DIR/bnet/noaugment_base/`. The most important
   files in this directory are:
   1. `named-cons-all.txt.pruned`. This contains the output of the cycle elimination procedure described in Algorithm 2
      of the paper.
   2. `named-bnet.out`. Contains the constraint graph as a Bayesian network with deterministic or and probabilistic and
      nodes. The essential difference between this file and `named-cons-all.txt.pruned` is that the alternate
      derivations of tuples are made explicit in the form of disjunctions.
   3. `bnet-dict.out`. The nodes of the Bayesian network in `named-bnet.out` are given numerical names to simplify
      processing. The `bnet-dict.out` file contains a mapping between these numbers and the tuples of
      `named-cons-all.txt`.
   4. `factor-graph.fg`. The underlying marginal inference library, LibDAI, works with factor graphs rather than
      Bayesian networks. This file contains the same Bayesian network as `named-bnet.out` in the form of a factor graph.

   In addition, the version of the script with compression produces the intermediate files `named-cons-all.txt.ep`,
   `named-cons-all.txt.cep` and `new-rule-prob.txt`. These files contain versions of `named-cons-all.txt.pruned` with
   various semantics preserving optimizations applied.

2. **Running the Bingo Interaction Loop (`driver.py`):** We are now ready to run the main Bingo interaction loop. Run
   the command:
   ```
   ./scripts/bnet/driver.py $PROBLEM_DIR/bnet/noaugment_base/bnet-dict.out \
                            $PROBLEM_DIR/bnet/noaugment_base/factor-graph.fg \
                            $PROBLEM_DIR/base_queries.txt \
                            $PROBLEM_DIR/oracle_queries.txt
   ```
   The arguments are as follows:
   1. `$PROBLEM_DIR/bnet/noaugment_base/bnet-dict.out` and `$PROBLEM_DIR/bnet/noaugment_base/factor-graph.fg`. This is
      the Bayesian network over which marginal inference is to be performed.
   2. `$PROBLEM_DIR/base_queries.txt`. This is the list of alarms which have been ultimately produced by the analysis
      tool. This is therefore the set of alarms which have to be ranked by marginal probabilities, rather than _all_
      tuples present in the Bayesian network.
   3. `$PROBLEM_DIR/oracle_queries.txt`. This contains the subset of alarms which are actually true bugs. Note that this
      argument was solely used for our experiments while developing the system: in practical deployments, the user is
      unaware of which alarms are real bugs until after triaging them. In these cases, it is completely acceptable to
      use an empty file, `cat /dev/null > $PROBLEM_DIR/oracle_queries.txt`, and it will leave the interaction unchanged.

   This script is an interactive driver that provides an interface between the user and the the LibDAI inference
   library. It repeatedly reads user commands from stdin and prints output to stdout and stderr. The most user-friendly
   command is called `MAC` (short for manual alarm carousel):
   ```
   MAC 1e-6 500 1000 100 stats.txt combined out
   ```
   The first four arguments set various properties of the belief propagation loop, respectively the tolerance at which
   to declare convergence, the minimum and maximum number of iterations of belief propagation, and the width of the
   low-pass filter in case of non-convergence. The arguments we have specified in the example invocation worked well for
   the benchmarks in the paper. The next argument, `stats.txt`, is the name of the file in which to print aggregate
   statistics from the interaction loop. The last two arguments, `combined` and `out`, indicate the format of the
   filenames in which to print the ranked list of alarms after each iteration.

Operating the Bingo Interaction Loop (`driver.py`)
--------------------------------------------------

As specified in the previous section, the easiest way to operate `driver.py` is to invoke the `MAC` command:
```
MAC tolerance minIters maxIters histLength stats.txt combined out
```
with `tolerance` = 1e-6, `minIters` = 500, `maxIters` = 1000 and `histLength` = 100. Users desiring greater flexibility
in the interaction loop may find the following commands useful:
1. `BP tolerance minIters maxIters histLength`. Run the loopy belief propagation algorithm given the evidence provided
   so far.
2. `HA`. Print the top ranked alarm and its confidence value.
3. `Q t`. Given a tuple `t` (for example `racePairs_cs(15974,16487)`), print its marginal probability.
4. `O t ground`. Specify the ground truth (`ground` = `true` or `ground` = `false`) of the tuple `t`.
5. `P filename`. Print the ranked list of alarms to a file. If the provided `oracle_queries.txt` is empty, then it
    automatically labels each alarm as `FalseGround` in the output file, but this is merely labelling and does not
    affect anything the actual ranking.
6. `AC tolerance minIters maxIters histLength stats.txt combined out`. Run the interaction loop, but assuming that the
   provided `oracle_queries.txt` file is the ground truth. This was the command we used to test the system while running
   the experiments in the paper.

Note that `driver.py` is very sensitive to the order in which these commands are issued. Please make sure that the
sequence of commands is from the following regular language: `MAC | (BP (HA | Q | P)* O)* | AC`. All commands except
`MAC` and `AC` are idempotent, and `HA`, `Q` and `P` do not change the state of the system.
