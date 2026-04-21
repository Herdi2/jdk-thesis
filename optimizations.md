# Compiler optimizations of interest

## Control flow

### ifnode.cpp
IfNode :: Ideal
    IfNode :: Ideal_common
        Node ::remove_dead_region
            * If the current control flow node is dead (unreachable), remove it and any subgraph below depending on it
            * Seems to transform a step: if the current node's input is a region, remove it an instead go from Region's
              non-null input to this node. Don't fully grasp this part.
        static idealize_test
            * Canonicalize test, i.e. swap boolean comparisons from eq, gt, ge to ne, le, lt or swap iftrue/iffalse
              branches.
        static split_if
            * A bit complex, should ask someone to understand
    static remove_useless_bool
        * As name suggests, removes useless bools e.g. (x < y ? true : false) => (x < y)
    IfNode :: fold_compares
        * Seems to fold integer comparisons, e.g. two consequent comparisons turn into one
        * How can I trigger it?
    IfNode :: dominated_by
        * Seems to replace an if-node with an identical one where possible
    IfNode :: simple_subsuming
        * Removes if-node/check if it is dominated by an if-node that subsumes it

### cfgnode.cpp
RegionNode :: Ideal
    redundant_region (unnamed)
        * If a region node has no phi users and both inputs come from either arm of the same If node, then the control-flow split is useless and removed.
    one_path (unnamed)
        * If Region node has only one path, remove it
        * Removal happens after parsing. In parsing, region nodes with one input are left in.
    static merge_region
        * If a region flows into another, they may be merged into one region
    RegionNode :: optimize_trichotomy
        * Optimizes comparisons with regions that use the idea of trichotomy, i.e. for any two numbers
          a and b, a > b, a == b or a < b.

PhiNode :: Ideal
    one_input (unnamed)
        * If a Phi node has only one valid input, we replace the phi node with this input
    * Don't understand the whole casting optimizations
    Optimization on diamond phis
        static is_x2logic
            * Converts phi to convIB (What is convIB?)
        static is_absolute
            * No clue
        static is_cond_add
            * Test-and-branch of (P < Q) ? X+Y : X => (sgn(P - Q) & Y) + X
        static split_flow_path
            * Seems to be Phi optimizations to allow for reducible loops, can ignore?
    ConvertNode :: create_convert
        * Performs the optimization: Phi(Region, Conv(X), Conv(Y)) => Conv(Phi(Region, X, Y)) when possible
    Rest of the optimizations seem focused on looping and special flags

### callnode.cpp
ReturnNode :: Ideal
    Node :: remove_dead_region
        * Removes unreachable control flow nodes

## Memory
### memnode.cpp
LoadNode :: Ideal
    MemNode :: Ideal_common
        *


Points of interest:
* PhaseGVN::transform(Node* n) where we delay arithmetic optimizations is supposed to optimize the given node to a
  semantically equivalent node that computes the results faster/cheaper.
