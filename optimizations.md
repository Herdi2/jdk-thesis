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
        * No explicit/interesting optimizations?
    MemNode :: optimize_memory_chain
        MemNode :: optimize_simple_memory_chain
            * Optimizations seem to focus on Calls, Allocations and Initialization etc. not related to pure memory stores.
        * Some Phi optimizations I can't wrap my head around

StoreNode :: Ideal
    MemNode :: Ideal_common
        * Same as LoadNode
    back_to_back_stores (unnamed)
        * Folds back-to-back stores to the same address (presumably same instance too)
    hoist_store (unnamed)
        * Related to initialization, not relevant to us
    fold_cast (unnamed)
        * Not sure about reinterpret casts
    merge_primitive_stores (unnamed)
        * Irrelevant, we do not touch primitive memory

StoreNode :: Identity
    * Removes redundant stores
    * Store(m, p, Load(m, p)) changes to m.
    * Store(, p, x) -> Store(m, p, x) changes to Store(m, p, x).


# Preliminary Bugs to examine
## Control flow
* Bug 1: Incorrect if-guard subsuming (if-clause incorrectly marked as dead due to dominating if), ifnode.cpp#1690
    * Git blame history uninteresting, written in one go
    * Seems to not have any tests?
    * Bugs: Change values in `short_circuit_map`
* Bug 2: Incorrect canonicalization of an if-guard, ifnode.cpp#1872 `IfNode::idealize_test`
    * Git blame history uninteresting, written in one go, minor updates with range check
    * Seems to have no tests as well >:|
    * Bugs: Canonicalize without switching IfTrue/IfFalse branch
* Bug 3: IfNode::fold_compares, fold 2 CmpI into one CmpU
    * Git blame 197ecf9bc10a bug: x <= 0 || x > 0 wrongly folded as (x-1) >u -1
    * Corresponding test to use, found in `TestBadFoldCompare.java`
    * Bugs: Reintroduce JDK-8346420
* Bug 4: Incorrect if-node equality leading to incorrect reuse of if nodes (?), ifnode.cpp#1513
    * Git blame found JDK-8347365, however related to div commoning
    * Functions of interest: `search_identical` and `dominated_by`
    * (dom->Opcode() != op ||  // Not same opcode?
         !same_condition(dom, igvn) ||  // Not same input 1?
         prev_dom->in(0) != dom)
    * Bugs: Change above condition to introduce bug, most interesting might be dominator
* Bug 4: Incorrect Phi-node elimination, due to assuming it only has one valid input, cfgnode.cpp#2196
    * Git blame has nothing interesting
    * Loops over region input to find TOP, meaning invalid paths are already marked
    * Bugs: Incorrectly kill control flow into region node (e.g. CmpINode::Ideal return nullptr on some operations?)
* Bug 5: Something with diamond phi pattern? (CMove?)
    * Git blame finds nothing
    * Actual CMove creation seems to happen in `conditional_move` in `loopopts.cpp`, replaces phi's with CMove
    * Bugs: Explicit FP bug found at `movenode.cpp@123`
* Bug 6: Incorrect Rangecheck CMove application (?), ifnode.cpp#1928


## Memory
** More bugs not relying on aliasing
* Bug 1: Back-to-back store folding on different instances (f1.x = 10; f2.x = 11; treated as f1.x = 10; f1.x = 11 => f1.x = 11), memnode.cpp#699
* Bug 2: Reusing load nodes from different instances (e.g. f1.x + f2.x treated as f1.x + f1.x), memnode.cpp#1959
* Bug 3: Removing stores from data flow (e.g. f1.x = 20; return f2.x treated as f1.x = 20; return f1.x => return 20), memnode.cpp#3563
Can potentially include memory bugs that affect control flow:
* Bug 4: Incorrect if-guard subsuming based on aliasing (if (f1.x > 0) { if (f2.x < 0) { ... } }, second if-clause incorrectly marked as dead)
* Bug 5: Incorrect phi-node elimination due to assuming it has one valid input (Once again, incorrect aliasing in the guards)

# Examining Git blame
* StoreNode::Identity
    * Found nothing interesting, mostly small rewrites

# Examining tests


Points of interest:
* PhaseGVN::transform(Node* n) where we delay arithmetic optimizations is supposed to optimize the given node to a
  semantically equivalent node that computes the results faster/cheaper.
* `x > 0 ? f1.x + 10 : f1.x + 11` will not use the same AddP for the loads, may lead to false-positives
* Doesn't memnode.cpp#711 confirm that a memnode cannot take top?
