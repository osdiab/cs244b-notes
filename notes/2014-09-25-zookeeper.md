# Zookeeper

## side note

* check out Spanner for a mathematically rigorous system

## Paper-writing thoughts

* This is typical for a lot of systems papers
  * 80% simple, 20% really complex
  * Hard to reason about the complexities of implementation that are present - what happens if a
    leader dies and a client dies at the same time after making a request... etc - not enough space
    to go into detail on these things
* this paper was probably cut down to be very compact - lots of effort into it
  * ends right on the last line of the last page
  * very few orphaned words on last lines of paragraphs
* this paper also is from industry, not academia
  * lots of examples of large-scale systems in use already, not just theory
  * api shown on the paper, very concrete - no shame in that
  * but this paper also happens to be relatively theoretically rigorous
* this paper was peer reviewed for a top conference
  * conferences decimate the vast majority of papers - very few make it through
  * selected very strictly by industry experts
  * There's politics in getting accepted by conferences - don't want to make enemies or not pay
    lip-service to others. Can be very arbitrary
* Compared to other fields like physics, english...
  * No math formulas or integrals or closed form statements - run synthetic experiments on actual
    practice instead.
    * Definitely synthetic - can't vary percent reads and writes that precisely
    * hard to generalize experiments, just like these synthetic ones
    * Why synthetic experiments?
      * legality with industry data
      * one company's real workload doesn't capture how all companies would use it
      * no error bars, no system configuration - can't be about reproducibility...
  *

## The paper itself

* wait-free?
  * not clearly defined
  * basically, the API is asynchronous
* is this "good"?
  * sacrifices quality/consistency by not always giving you the correct version of the data
  * you can watch for version numbers or sync to make up for it
    * but that's not really wait-free anymore
* comparison with chubby
  * seems like it is not a fair comparison - different promises about behavior in each system
  * but maybe that's ok, so long as the paper can prove that it's "good enough" to do what you need
    to do

* api insights
  * no open/close
    * in nfs, you'd open a file handle, and then operate on that. You're going to read teh original
      value
      * in zk, you need to include the version number with a path to get a stable version of a file
  * no partial reads/writes - can't read/write line by line
    * easier to coordinate
    * no partial writes = consistent since versions can't change partially and interfere with one
      another
    * don't do this with big files!
  * would you use this for very frequent writes? hell naw. operations on the order of minutes or
    hours
* what happens if you have two writes from different clients at teh same time?
  * data per-client will be processed in the correct order
  * writes are serializable
