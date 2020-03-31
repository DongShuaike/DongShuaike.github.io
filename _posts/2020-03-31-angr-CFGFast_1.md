---
layout: post
title:  "Diving into angr source code: how CFGFast() works"
date:   2020-03-31 17:15:09 +0800
categories: angr 
---

CFGs are most-commonly used objects in programming analysis. In this post, I will explore how CFGs are built in angr. Angr provides two types of CFGs -- CFGFast and CFGAccurate. To be precise, what I'm going to analyze is CFGFast().

All the things related to CFGFast are recorded in `angr.analyses.cfg.cfg_fast.py`. From it we can see `CFGFast` with two parent classes -- `ForwardAnalysis` and `CFGBase`.

As the developer says,
> CFGFast will only perform light-weight analyses combined with some heuristics, and with some strong assumptions.

we know the most priorized goal of `CFGFast` is the speed, then comes with the accuracy.

Same as all other classes inhereted from `ForwardAnalysis`, `CFGFast` starts its analysis from `_analyze()` function. However, since `CFG` is the building block of all other graph-based analysis, its `self._graph_visitor` is None and will then step into `self._analysis_core_baremetal()`. 

Before the analysis on baremetal, there is something done in `CFGFast._pre_analysis`.

`_pre_analysis` conducts some initialization work, including many useful variables during the analysis, including 
```

```
One of the mose important variables is `starting_points`. Without it, the CFGFast will not be able to start its analysis. The variable is defined as follows:
```
starting_points = set()

# clear all existing functions
self.kb.functions.clear()

if self._use_symbols:
    starting_points |= self._function_addresses_from_symbols

if self._extra_function_starts:
    starting_points |= set(self._extra_function_starts)

# Sort it
starting_points = sorted(list(starting_points), reverse=True)

if self._start_at_entry and self.project.entry is not None and self._inside_regions(self.project.entry) and \
        self.project.entry not in starting_points:
    # make sure self.project.entry is inserted
    starting_points += [ self.project.entry ]
```

From this piece of code, we can see `starting_points` has three sources:
1. function addresses obtained from the symbol table. Tracing `self._function_addresses_from_symbols`, I find the function `_func_addrs_from_symbols()` with the following definition:
```
return {sym.rebased_addr for sym in self._binary.symbols if sym.is_function}
```
indicating that all functions listed in the symbol table are counted into the `starting_points`.
2. Apart from those listed in the symbol table, angr also provides an optional argument called `function_starts` to let users provide a extra list of functions.
3. The third one is the entry point of each excutable file.

After determining all the starting points, angr creates `CFGJob`s for them and insert the CFGJobs into the job queue.
```
for sp in starting_points:
    job = CFGJob(sp, sp, 'Ijk_Boring')
    self._insert_job(job)
    # register the job to function `sp`
    self._register_analysis_job(sp, job)
```

After finishing the preanalysis work, the control is passed to `_analysis_core_baremetal`. Inside it, jobs are poped from the queue one by one, and the poped one is then passed to `_process_job_and_get_successors` to handle. Note that currently these two procedures are still defined in `ForwardAnalysis`.

```
def _process_job_and_get_successors(self, job_info):
    """
    Process a job, get all successors of this job, and call _handle_successor() to handle each successor.

    :param JobInfo job_info: The JobInfo instance
    :return: None
    """

    job = job_info.job

    successors = self._get_successors(job)

    all_new_jobs = [ ]

    for successor in successors:
        new_jobs = self._handle_successor(job, successor, successors)

        if new_jobs:
            all_new_jobs.extend(new_jobs)

            for new_job in new_jobs:
                self._insert_job(new_job)

    self._post_job_handling(job, all_new_jobs, successors)
```
In `_process_job_and_get_successors`, the successors of a Job is obtained through `self._get_successor(job)`, which is defined in `CFGFast`.

```
def _get_successors(self, job):  # pylint:disable=arguments-differ

    # current_function_addr = job.func_addr
    # addr = job.addr

    # if current_function_addr != -1:
    #    l.debug("Tracing new exit %#x in function %#x", addr, current_function_addr)
    # else:
    #    l.debug("Tracing new exit %#x", addr)

    jobs = self._scan_block(job)

    # l.debug("... got %d jobs: %s", len(jobs), jobs)

    for job_ in jobs:  # type: CFGJob
        # register those jobs
        self._register_analysis_job(job_.func_addr, job_)

    return jobs
```
Then Dive into `self._scan_block(job)`
```
def _scan_block(self, cfg_job):
    """
    Scan a basic block starting at a specific address

    :param CFGJob cfg_job: The CFGJob instance.
    :return: a list of successors
    :rtype: list
    """

    addr = cfg_job.addr
    current_func_addr = cfg_job.func_addr

    # Fix the function address
    # This is for rare cases where we cannot successfully determine the end boundary of a previous function, and
    # as a consequence, our analysis mistakenly thinks the previous function goes all the way across the boundary,
    # resulting the missing of the second function in function manager.
    if addr in self._function_addresses_from_symbols:
        current_func_addr = addr

    if self._addr_hooked_or_syscall(addr):
        entries = self._scan_procedure(cfg_job, current_func_addr)

    else:
        entries = self._scan_irsb(cfg_job, current_func_addr)

    return entries
```
Based on the type of current address, `_scan_irsb` and `_scan_procedure` will be individually invoked. So what's happening in `_scan_irsb`?
IRSB stands for ***Intermediate Representation Super-Block***. Since all analyses in angr work on valgrind VEX IR, and therefore an IRSB is a single-entry, multiple-exit code block.

The first statement in `_scan_irsb` is `addr, function_addr, cfg_node, irsb = self._generate_cfgnode(cfg_job, current_func_addr)`. Step into `_generate_cfgnode` we will see how a CFG node is generated. To conclude, `_generate_cfgnode` returns 4 important objects `addr`, `current_function_addr`, `cfg_node` and `irsb`. Since the cfg node should have a non-overlapped boundary, `_generate_cfgnode` performs several checks:
1. if the section containing current block executable or not? If not, return None since non-executable sections must not contain any code.
2. Does the current block exceeds the scope of the current function and spans into another? If it is, the size of that block should be `min(block size, distance to the nearest func)`
3. Does the current spans across the closest occupied region in segment list? **I'm not sure what is this check used for, hope to know it later.**

Now go back to `_scan_irsb`, at line 1516 of `cfg_fast.py`, there is a statement 
```
self._process_block_arch_specific(addr, irsb, function_addr)
```
Step into this function, in its comments I see
> According to arch types ['ARMEL', 'ARMHF', 'MIPS32'] does different fixes.
> 
> For ARM deals with link register on the stack
  (see _arm_track_lr_on_stack)
  For MIPS32 simulates a new state where the global pointer is 0xffffffff
  from current address after three steps if the first successor does not
  adjust this value updates this function address (in function manager)
  to use a conrete global pointer

Since mostly I'm working on MIPS binary analysis, let's go directly to the part dealing MIPS.
**TODO**
```

```

As we know, a control flow graph is comprised of multiple control flow nodes linked with each other by edges. How does `CFGFast` gets edges? In our case, how `successors` and `predecessors` of a CFG node are summarized?

From line 1528, we have 
```
if irsb.statements:
    for i, stmt in enumerate(irsb.statements):
        if isinstance(stmt, pyvex.IRStmt.Exit):
            successors.append((i,
                               last_ins_addr if self.project.arch.branch_delay_slot else ins_addr,
                               stmt.dst,
                               stmt.jumpkind
                               )
                              )
        elif isinstance(stmt, pyvex.IRStmt.IMark):
            last_ins_addr = ins_addr
            ins_addr = stmt.addr + stmt.delta
else:
    for ins_addr, stmt_idx, exit_stmt in irsb.exit_statements:
        successors.append((
            stmt_idx,
            last_ins_addr if self.project.arch.branch_delay_slot else ins_addr,
            exit_stmt.dst,
            exit_stmt.jumpkind
        ))

successors.append((DEFAULT_STATEMENT,
                   last_ins_addr if self.project.arch.branch_delay_slot else ins_addr, irsb_next, jumpkind)
                  )
```
From it we know, when the current VEX statement being analyzed is of the type `Exit`, the destination of that statement will be added to the list `successors`. 

Do we finish all the work yet? No! from line 1558 we have 
```
for suc in successors:
    stmt_idx, ins_addr, target, jumpkind = suc

    entries += self._create_jobs(target, jumpkind, function_addr, irsb, addr, cfg_node, ins_addr,
                                 stmt_idx
                                 )                                  
```
From it we know each one of the collected successors will be given a created job. Step into `self._create_jobs` to see what happens:

```
def _create_jobs(self, target, jumpkind, current_function_addr, irsb, addr, cfg_node, ins_addr, stmt_idx):
        """
        Given a node and details of a successor, makes a list of CFGJobs
        and if it is a call or exit marks it appropriately so in the CFG

        :param int target:          Destination of the resultant job
        :param str jumpkind:        The jumpkind of the edge going to this node
        :param int current_function_addr: Address of the current function
        :param pyvex.IRSB irsb:     IRSB of the predecessor node
        :param int addr:            The predecessor address
        :param CFGNode cfg_node:    The CFGNode of the predecessor node
        :param int ins_addr:        Address of the source instruction.
        :param int stmt_idx:        ID of the source statement.
        :return:                    a list of CFGJobs
        :rtype:                     list
        """
```
As we know, except from the `Exit` statement, a normal `Call` also indicates the change of control flow. Let's see what measurements angr take to handle such cases.

To put it in a word, angr takes different approaches dealing with `jump`s of concrete target address and non-concrete target address.

First let's see what happens when the jump target is not concrete.
```
if target_addr is None:
    # The target address is not a concrete value

    if jumpkind == "Ijk_Ret":
        # This block ends with a return instruction.
        if current_function_addr != -1:
            self._function_exits[current_function_addr].add(addr)
            self._function_add_return_site(addr, current_function_addr)
            self.functions[current_function_addr].returning = True
            self._pending_jobs.add_returning_function(current_function_addr)

        cfg_node.has_return = True

    elif self._resolve_indirect_jumps and \
            (jumpkind in ('Ijk_Boring', 'Ijk_Call', 'Ijk_InvalICache') or jumpkind.startswith('Ijk_Sys')):
        # This is an indirect jump. Try to resolve it.
        # FIXME: in some cases, a statementless irsb will be missing its instr addresses
        # and this next part will fail. Use the real IRSB instead
        irsb = cfg_node.block.vex
        cfg_node.instruction_addrs = irsb.instruction_addresses
        resolved, resolved_targets, ij = self._indirect_jump_encountered(addr, cfg_node, irsb,
                                                                         current_function_addr, stmt_idx)
        if resolved:
            for resolved_target in resolved_targets:
                if jumpkind == 'Ijk_Call':
                    jobs += self._create_job_call(cfg_node.addr, irsb, cfg_node, stmt_idx, ins_addr,
                                                  current_function_addr, resolved_target, jumpkind)
                else:
                    to_outside, target_func_addr = self._is_branching_to_outside(addr, resolved_target,
                                                                                 current_function_addr)
                    edge = FunctionTransitionEdge(cfg_node, resolved_target, current_function_addr,
                                                  to_outside=to_outside, stmt_idx=stmt_idx, ins_addr=ins_addr,
                                                  dst_func_addr=target_func_addr,
                                                  )
                    ce = CFGJob(resolved_target, target_func_addr, jumpkind,
                                last_addr=resolved_target, src_node=cfg_node, src_stmt_idx=stmt_idx,
                                src_ins_addr=ins_addr, func_edges=[ edge ],
                                )
                    jobs.append(ce)
            return jobs

        if ij is None:
            # this is not a valid indirect jump. maybe it failed sanity checks.
            # for example, `jr $v0` might show up in a MIPS binary without a following instruction (because
            # decoding failed). in this case, `jr $v0` shouldn't be a valid instruction, either.
            return [ ]

        if jumpkind in ("Ijk_Boring", 'Ijk_InvalICache'):
            resolved_as_plt = False

            if irsb and self._heuristic_plt_resolving:
                # Test it on the initial state. Does it jump to a valid location?
                # It will be resolved only if this is a .plt entry
                resolved_as_plt = self._resolve_plt(addr, irsb, ij)

                if resolved_as_plt:
                    jump_target = next(iter(ij.resolved_targets))
                    target_func_addr = jump_target  # TODO: FIX THIS

                    edge = FunctionTransitionEdge(cfg_node, jump_target, current_function_addr,
                                                  to_outside=True, dst_func_addr=jump_target,
                                                  stmt_idx=stmt_idx, ins_addr=ins_addr,
                                                  )
                    ce = CFGJob(jump_target, target_func_addr, jumpkind, last_addr=jump_target,
                                src_node=cfg_node, src_stmt_idx=stmt_idx, src_ins_addr=ins_addr,
                                func_edges=[edge],
                                )
                    jobs.append(ce)

            if resolved_as_plt:
                # has been resolved as a PLT entry. Remove it from indirect_jumps_to_resolve
                if ij.addr in self._indirect_jumps_to_resolve:
                    self._indirect_jumps_to_resolve.remove(ij.addr)
                    self._deregister_analysis_job(current_function_addr, ij)
            else:
                # add it to indirect_jumps_to_resolve
                self._indirect_jumps_to_resolve.add(ij)

                # register it as a job for the current function
                self._register_analysis_job(current_function_addr, ij)

        else:  # jumpkind == "Ijk_Call" or jumpkind.startswith('Ijk_Sys')
            self._indirect_jumps_to_resolve.add(ij)
            self._register_analysis_job(current_function_addr, ij)

            jobs += self._create_job_call(addr, irsb, cfg_node, stmt_idx, ins_addr, current_function_addr, None,
                                          jumpkind, is_syscall=is_syscall
                                          )
```
The first case it handles is the statement with an normal `return`. The `cfg_node.has_return` will be set `True`.

The second case is the statement with an indirect target, angr  tries to recover the real target by `cfg_base._indirect_jump_encountered`. The comments on the top say angr tries to resolve the indirect jump using timeless (fast) indirect jump resolvers, and then `cfg_base._resolve_indirect_jump_timelessly` is executed, due to the page limitation, I will write down all my analyses of **angr jump resolvers** in a new post.

When the target address is concrete, angr does  following things:
```
elif target_addr is not None:
    # This is a direct jump with a concrete target.

    # pylint: disable=too-many-nested-blocks
    if jumpkind in ('Ijk_Boring', 'Ijk_InvalICache'):
        to_outside, target_func_addr = self._is_branching_to_outside(addr, target_addr, current_function_addr)
        edge = FunctionTransitionEdge(cfg_node, target_addr, current_function_addr,
                                      to_outside=to_outside,
                                      dst_func_addr=target_func_addr,
                                      ins_addr=ins_addr,
                                      stmt_idx=stmt_idx,
                                      )

        ce = CFGJob(target_addr, target_func_addr, jumpkind, last_addr=addr, src_node=cfg_node,
                    src_ins_addr=ins_addr, src_stmt_idx=stmt_idx, func_edges=[ edge ])
        jobs.append(ce)

    elif jumpkind == 'Ijk_Call' or jumpkind.startswith("Ijk_Sys"):
        jobs += self._create_job_call(addr, irsb, cfg_node, stmt_idx, ins_addr, current_function_addr,
                                      target_addr, jumpkind, is_syscall=is_syscall
                                      )

    else:
        # TODO: Support more jumpkinds
        l.debug("Unsupported jumpkind %s", jumpkind)
        l.debug("Instruction address: %#x", ins_addr)
```
If the jumpkind is of the type `Ijk_Boring` or `Ijk_InvalICache`, a `FunctionTransitionEdge` will be generated and then a new CFGJob will be produced and added to the queue.
**TODO: clarify Ijk_InvalICache**

If the jumpkind is `Ijk_Call` or something with the prefix `Ijk_Sys`, `_create_job_call` will be called. In it the target will be given a CFGJob, the analysis continues.

Now let's go back to function `_process_job_and_get_successors` in `forward_analysis`, we can see that after collecting all `successors`, angr starts to handle each of them.
```
for successor in successors:
  new_jobs = self._handle_successor(job, successor, successors)
```
Interestingly, I found the implementation of `_handle_successor` is quite simple, just one statement `return [successor]`, which means most of the work of generating a CFG has been done in `_get_successors` function.

Uptill now, we have almost finish analyzing the whole work flow of CFGFast, but there is still a question, how does angr deal with `indirect jump targets`? For example, what if the target is stored in a general-purpose register? I will take MIPS as the example to uncover this.


