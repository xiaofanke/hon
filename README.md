# HON
Code for generating Higher-order Network (HON) from data with higher-order dependencies.
See details in paper [Representing higher-order dependencies in networks](http://advances.sciencemag.org/content/2/5/e1600028).
* Input: Trajectories / sequential data, such as ship movements among ports, a person's clickstream of websites, and so on.
* Output: HON edges in triplets [FromNode] [ToNode] [weight]

## Compatibility
The code is written in Common Lisp. Tested with [SBCL](http://www.sbcl.org/), but should run on any modern Common Lisp implementation.

Tested on Linux, Windows and Mac.

_A python implementation is under development. If you are interested, please let me know._

## Setting up environment
### Recommended environment
On *Linux*, follow [this tutorial](http://www.jonathanfischer.net/modern-common-lisp-on-linux/) to set up Emacs, SBCL, Quicklisp and SLIME.

After installing [Quicklisp](http://www.quicklisp.org/), call ql:quickloads :split-sequence.

For other platforms, follow [this tutorial](http://cliki.net/Getting+Started)

### Minimal environment
Install SBCL and QuickLisp. Call ql:quickloads :split-sequence.

## Workflow
### 1. Rule extraction

This corresponds to Algorithm 1 in paper.

Using Emacs and SLIME: open build-rules.lisp, press CTRL+C twice to compile, run (main).

Using Minimal environment: run sbcl, run twice (load "build-rules.lisp"), run (main).

#### Input file
Trajectories / sequential data. See test-trace.csv or traces-simulated-mesh-v100000-t100-mo4.csv for example.

In the context of ship movements, every line is a ship's trajectory, in the format of [ShipID] [Port1] [Port2] [Port3] ... 
> Notice that the first element of every line is the ship's ID.

ShipID and PortID can be any integer. 
> Non-integers can be used if function _parse-lists-for-integer_ is removed from (main).

Other types of trajectories or sequential data can be used, such as a person's clickstream of websites, music playing history, sequences of check-ins, and so on.

#### Output file
Variable orders of "rules" extracted from the sequential data. See rules-simulated-mesh-v100000-t100-mo4-kl.csv for example.

Every line of record represents a "rule", which is the (normalized) probability of going to [TargetPort] from [PreviousPorts], in the format of ... [PrevPrevPort] [PrevPort] [CurrPort] => [TargetPort] [Probability]. 
> If you want to output the number of observations instead of the normalized probability, in function (add-to-rules), change the dictionary of *distributions* to the length of the source nodes's value in *observations*, and remove (clrhash *observations*) in (build-distributions)

#### Parameters
Parameters are at the beginning of the file build-rules.lisp, starting with *defparameter*.

##### max-order
For each path, the rule extraction process attempts to increase the order until the maximum order is reached. The default value of 5 should be sufficient for most applications. Setting this value as 1 will yield a conventional first-order network. Discussion of this parameter (how it influences the accuracy of representation and the size of the network) is given in the supporting information of the paper.

> Setting this parameter too large may result in (1) long running time and/or (2) exhausting the heap and die silently (run sbcl with larger dynamic-space-size). Try increasing the value of this parameter progressively: if max-order is 5 but the rules extracted show at most 3rd order, then there is no need further increasing max-order.

##### min-support
Observations that are less than min-support are discarded during preprocessing. 

For example, if the patter [Shanghai, Singapore] -> [Tokyo] appears 500 times and [Zhenjiang, Shanghai, Singapore] -> [Tokyo] happened only 3 times, and min-support is 10, then [Zhenjiang, Shanghai, Singapore] -> [Tokyo] will not be considered as a higher-order rule.

This parameter is useful for filtering out infrequent patterns that might be considered as higher-order rules. Setting a reasonable min-support can (1) considerably decrease the network size and the time needed for computation and (2) potentially improving representation accuracy by filtering out noise. Discussion of this parameter (how it influences the accuracy of representation and the size of the network) is given in the supporting information of the paper. 

>If do not want to use this filter, set the value as 1.

##### min-length-of-trajectory
Trajectories shorter than the given value are discarded during preprocessing. Useful for filtering out inactive individuals.
> If do not want to use this filter, set the value as 1.

##### digits-for-testing
The last given steps in each trajectory are not used to for network construction (reserved for testing). 
> If want to use full trajectories, set this value as 0.
> max-order + digits-for-testing should be smaller or equal to min-length-of-trajectory.

### 2. Network wiring

This corresponds to Algorithm 1 in paper.

Using Emacs and SLIME: open build-network.lisp, press CTRL+C twice to compile, run (main).

Using Minimal environment: run sbcl, run twice (load "build-network.lisp"), run (main).

#### Input file
The file containing rules produced by the last step.

#### Output file
HON edges in triplets [FromNode],[ToNode],[weight]
> Default weight is the probability of a random walker going from [FromNode] to [ToNode]. If you want to output the number of observations instead of the normalized probability, follow previous instructions.

Every node can be a higher-order node, in the format of [CurrNode]|PrevNode.PrevPrevNode.PrevPrevPrevNode

This representation (as comma deliminated network edges file) is directly compatible with the conventional network representation and analysis tools. The only difference is edge labels.

## Synthetic data
> It is not feasible to add rules beyond third order using the same setting in paper, because movements matching multiple previous steps will unlikely happen. However you can use a different setting to generate even higher orders.

### Script
The script synthesize-trace-mesh.py (run with python or python3) synthesizes trajectories of vessels going certain steps on a 10x10 mesh (i.e. grid).
10,000,000 movements will be generated by default.
Normally a vessel will go either up/down/left/right with the same probability (see function NextStep)
(vessel movements are generated almost randomly with the following exceptions)

We add a few higher order rules to control the generation of vessel movements:
2nd order rule: if vessel goes right from X0 to node X1 (X means a number 0-9),
then the next step will go right with probability 70%, go down 30% (see function BiasedNextStep).
3rd order rule: if vessel goes right to node X7 then X8, then the next step will go right with probability 70%, go down 30%.
4th order rule: if vessel goes down from 1X to 2X to 3X to 4X,
then the next step will go right with probability 70%, go down 30%.
There are ten 2nd order rules, ten 3rd order rules, ten 4th order rules, no other rules.

### Data
Data generated using the aforementioned script is provided as traces-simulated-mesh-v100000-t100-mo4.csv

### Rule extraction with HON
Use HON with a default setting of MaxOrder = 5, MinSupport = 5
It should be able to detect all 30 of these rules of various orders,
And will not detect "false" dependencies such as 5th order rules.
The result is provided as rules-simulated-mesh-v100000-t100-mo5-ms5-t01.csv, note that for illustration here (since we are not going to build a network now), in this file when higher order rules are detected, corresponding lower order rules are not added recursively.
If all preceding rules are added, following the algorithm in the paper, the expected result should be rules-simulated-mesh-v100000-t100-mo4-kl.csv.
The whole time to process these 10,000,000 movements should be under 5 seconds (unoptimized code, on single core, excluding disk IO).



## Contact
Please contact Jian Xu (jxu5 at nd dot edu) for any question. 
_A python implementation is under development. If you are interested, please let me know._
