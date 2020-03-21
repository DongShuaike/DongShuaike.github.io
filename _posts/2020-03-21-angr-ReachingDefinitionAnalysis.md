---
layout:post
title: "How Reaching Definition Analysis is implemented in angr"
date:	2020-03-21 18:01:00 +0800
categories: angr
---
It has been a time since I got to know angr. As an opensource academic project, it does a quite good job. In the coming days (months), I will try to dive into angr source code, for the aim of knowing its internals, the structure of this awesome framework and applies my algorithms to it.

I selected the Reaching Definition Analysis (RDA) as the breakthrough.  As its self-definition says: 
> angr is a python framework for analyzing binaries. It combines both static and dynamic symbolic ("concolic") analysis, making it applicable to a variety of tasks. 

The most significant advantage of angr is its combination of different types of analysis, and a very good expansibility.

I used PyCharm to do code exploration, thanks to its convenient and powerful Forward/Backward navigation functionalities, I saved a lot of time. **The code I'm referring to is angr-dev version (8, 20, 1, 7)** (outputed by `python -c "import angr; print(angr.__version__)"`)


The most obvious class of RDA is `ReachingDefinitionAnalysis`, it is inherited from two parent classes `ForwardAnalysis` and `Analysis`. 

Actually `ForwardAnalysis` can be regarded as the most important class for you to implement your own analysis "pass" as it provides a foundamental structure for forward analysis.

> This is my very first attempt to build a static forward analysis framework that can serve as the base of multiple  
static analyses in angr, including CFG analysis, VFG analysis, DDG, etc.  
In short, ForwardAnalysis performs a forward data-flow analysis by traversing a graph, compute on abstract values,  
and store results in abstract states. The user can specify what graph to traverse, how a graph should be traversed,  
how abstract values and abstract states are defined, etc.
-- Fish

`ForwardAnalysis` provides basic building blocks for implementing a forward analysis. The most crutial function is `_analyze`, inside it we can see:

```
    def _analyze(self):
        """
        The main analysis routine.

        :return: None
        """
        self._pre_analysis()
        if self._graph_visitor is None:
            # There is no base graph that we can rely on. The analysis itself should generate successors for the
            # current job.
            # An example is the CFG recovery.

            self._analysis_core_baremetal()
        else:
            # We have a base graph to follow. Just handle the current job.
            self._analysis_core_graph()
        self._post_analysis()
```
We can do some self-defined pre-work in `_pre_analysis()`, after that, if we provide a `graph` object to ForwardAnalysis, the control will be given to `_analysis_core_graph()`. After analyzing each node of the graph, we can do some work by `_post_analysis()`.

Diving into `_analysis_core_graph` function, we can see, nodes are unstoppably fetched by `self._graph_visitor.next_node()`, on each node, `self._run_on_node()` is executed. It returns a new `state` named `output_state` and the `output_state` of the current node will be used as the `input_state` of the next node. 

Note that the `state` here is not what we refer to `SimState` when playing angr symbolic execution. It's just a type of program abstraction, you can even define yours.
```
   def _analysis_core_graph(self):

        while not self.should_abort:

            self._intra_analysis()

            n = self._graph_visitor.next_node()

            if n is None:
                break

            job_state = self._get_input_state(n)
            if job_state is None:
                job_state = self._initial_abstract_state(n)

            changed, output_state = self._run_on_node(n, job_state)

            # output state of node n is input state for successors to node n
            successors_to_visit = self._add_input_state(n, output_state)

            if changed is False:
                # no change is detected
                continue
            elif changed is True:
                # changes detected
                # revisit all its successors
                self._graph_visitor.revisit_successors(n, include_self=False)
            else:
                # the change of states are determined during state merging (_add_input_state()) instead of during
                # simulated execution (_run_on_node()).
                # revisit all successors in the `successors_to_visit` list
                for succ in successors_to_visit:
                    self._graph_visitor.revisit_node(succ)
```

After knowing the whole structure of ForwardAnalysis, let's jump back to class `ReachingDefinitionsAnalysis`. Tracing from `_analyze()`, we will then reach `self._run_on_node`. Inside it we can find two temporary variables -- `block` and `engine`, the `block` refers to the current code block which is being processed and `engine`, in our case, is an instance of the type `SimEngineRDVEX`, which will be dealt with later.
```
self._engine_vex = SimEngineRDVEX(self.project, self._current_local_call_depth, self._maximum_local_call_depth,
                                          self._function_handler)
...
block = self.project.factory.block(node.addr, node.size, opt_level=0)
block_key = node.addr
engine = self._engine_vex
```
After that we find a key statement `self.node_observe(node.addr, state, OP_BEFORE)`. angr RDA, and other similar analysis, provide a concept of `observation`, we can set as many observation points as we want so that we can see what is happening there. The observation points can be "instructions" or "blocks", the time to observe can be "OP_BEFORE" or "OP_AFTER". For those states already reaching the observation points, they will be stored in a python-dictionary object like the following shows:

```
key = 'node', node_addr, op_type
...
self.observed_results[key] = state
```
Following it, here comes to the very important sentence
```
state, self._visited_blocks = engine.process(
    state,
    block=block,
    fail_fast=self._fail_fast,
    visited_blocks=self._visited_blocks
)
```
To know how that engine `process` a block, we need dive into the code of class `SimEngineRDVEX`.
From `class SimEngineRDVEX(SimEngineRDVEXMinin, SimEngineLight)` we can see, `SimEngineRDVEX` has inherited something form `SimEngineLight` and `SimEngineRDVEXMinin`.

```
def process(self, state, *args, **kwargs):
    self._def_use_graph = kwargs.pop('def_use_graph', None)
    self._visited_blocks = kwargs.pop('visited_blocks', None)

    # we are using a completely different state. Therefore, we directly call our _process() method before
    # SimEngine becomes flexible enough.
    try:
        self._process(
            state,
            None,
            block=kwargs.pop('block', None),
        )
    except SimEngineError as e:
        if kwargs.pop('fail_fast', False) is True:
            raise e
        l.error(e)
    return self.state, self._visited_blocks
```
The private function `_process` in `SimEngineRDVEXMinin` leads us to `_proccess_Stmt` function.

```
def _process_Stmt(self, whitelist=None):
  ...
  for stmt_idx, stmt in enumerate(self.block.vex.statements):
      if whitelist is not None and stmt_idx not in whitelist:
          continue
      self.stmt_idx = stmt_idx

      if type(stmt) is pyvex.IRStmt.IMark:
          # Note that we cannot skip IMarks as they are used later to trigger observation events
          self.ins_addr = stmt.addr + stmt.delta

      self._handle_Stmt(stmt)
```
Here comes to the exciting part `_handle_Stmt`:
```
def _handle_Stmt(self, stmt):
    handler = "_handle_%s" % type(stmt).__name__
    if hasattr(self, handler):
        getattr(self, handler)(stmt)
```
We can see it uses `hasattr` and `getattr` to invoke specitif "Stmt handlers" (like what `Java reflection` does), if a developer have their own ways of dealing with different Stmts, he/she can overwrites these and write his/her own.

Back into `SimEngineRDVEX` we find there exist a lot of `_handle_xxx` routines, take `_handle_WrTmp` as an example,

```
def _handle_WrTmp(self, stmt):
    super()._handle_WrTmp(stmt)
    self.state.kill_and_add_definition(Tmp(stmt.tmp), self._codeloc(), self.tmps[stmt.tmp])
```
After running the parent's `_handle_WrTmp` function, `kill_and_add_definition` function is invoked, since "writing a new tmp" will absolutely generate a new definition, and whether some old definitions will be killed depends.

As a totally static analysis, RDA traverses each VEX (valgrind IR) statements and stores their effects into `state`s. One other thing to mention, since angr works on VEX level, their atoms are different from those low-level assembly. If you want to know what an atom (in our case, `Tmp`), you need to check how those atoms are defined, let's say `angr.analyses.reaching_definitions.atoms` module in RDA.


