---
title: GHC Development - Adding a TickTicky counter.
tags: Haskell, GHC
---

In this post I will summarize the steps required to add a counter which will be used with [`-ticky` profiling](https://gitlab.haskell.org/ghc/ghc/wikis/debugging/ticky-ticky).

We will then use this to try and estimate the impact a partial fix for [#8905](https://gitlab.haskell.org/ghc/ghc/issues/8905) would have.

The short summary of the issue is that when GHC checks if a value is already evaluated, it *unconditionally* sets up the stack for *potentially* evaluating
the value.  
There are good reasons for this, but when dealing with the only argument of a function they do not apply. So we want
to estimate the impact of special casing that particular pattern.

# What is ticky profiling.

It's a profiling method in which GHC annotates the **optimized** program with runtime counters.

This means core and stg passes are not affected by the profiling annotations and the final executable will be
closer to a release build than the case with [cost center profiling.](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/profiling.html)

The large downside is of course that we don't get "stacks". Knowing that `myExpensiveFunction` was called 5 Billion times is no doubt useful.
But even more useful is it if we know that it's called by `thisFunctionWithARedundantBang`.

Despite the drawbacks I still found it quite usefull when investigating performance hotspots in the past. I've recently also
used it to add custom counters to characterize the possible performance impact of this optimization for a custom branch.

Either way this post is not about WHAT ticky-ticky is, the wiki has most of the info on that.  
It's about how we can add new counters!

# Adding the counter

If you want to add a new ticky counter your first stop should be [`StgCmmTicky.hs`](https://gitlab.haskell.org/ghc/ghc/blob/master/compiler/codeGen/StgCmmTicky.hs) which has a
helpful comment `OVERVIEW: ticky ticky profiling`.

I won't reproduce the content here but go give it a skim.

## The problem

First we need to think of something we want to measure. Ticky works best (or arguably really only when) we want to measure how often
a certain code path was taken.

Let's say we look at [#8905](https://gitlab.haskell.org/ghc/ghc/issues/8905) and wonder if it is worthwhile to
solve this problem at least for functions with a single argument.

So for that we want to figure out how often we actually evaluate a function with a single evaluated argument!

## Step 1: Adding a counter

Adding a new ticky counter involves copy pasting a little code in a few places. `OVERVIEW: ticky ticky profiling` is helpful here as it lists all the use sites for counters.

* `includes/stg/Ticky.h` has the list of all counters.
* `rts/RtsSymbols.c` also needs to mention the new counter
* `rts/Ticky.c` needs to print the new counter

The easiest way to do so is to pick an existing counter, check where it occurs and add your counter in the same places.
So we could for example grep for `TICK_ENT_VIA_NODE`.

```
Andi@Horzube MINGW64 /e/ghc_head
$ grep "ENT_VIA_NODE_ctr" . -r -I
./compiler/codeGen/StgCmmTicky.hs:tickyEnterViaNode     = ifTicky $ bumpTickyCounter (fsLit "ENT_VIA_NODE_ctr")
./includes/Cmm.h:#define TICK_ENT_VIA_NODE()             TICK_BUMP(ENT_VIA_NODE_ctr)
./includes/stg/Ticky.h:EXTERN StgInt ENT_VIA_NODE_ctr INIT(0);
./rts/RtsSymbols.c:      SymI_HasProto(ENT_VIA_NODE_ctr)                   \
./rts/Ticky.c:        tot_enters - ENT_VIA_NODE_ctr;
./rts/Ticky.c:  PR_CTR(ENT_VIA_NODE_ctr);
```

First we add the new counter `ENT_SINGLE_ARG_ctr` to `Ticky.h`:

```C
...
EXTERN StgInt ENT_SINGLE_ARG_ctr INIT(0);
...
```

Then `RtsSymbols.c`

```
      ...
      SymI_HasProto(ENT_SINGLE_ARG_ctr)                 \
      ...
```

We skip `Cmm.h` as we won't expose our counter to handwritten Cmm. It's only temporary. 
But if you plan to keep the counter around in GHC please also add it there.


Then `StgCmmTicky.hs`, don't forget to also export the definition.

```haskell
tickySingleEvalArg :: FCode ()
tickySingleEvalArg         = ifTicky $ bumpTickyCounter (fsLit "ENT_SINGLE_ARG_ctr")
```

Important to note is that tickySingleEvalArg bumps a **runtime** counter when the code
generated by tickySingleEvalArg is executed.

As a last step we want the counter to show up in the output, so we add it also to `Ticky.c`.

```C
void
PrintTickyInfo(void)
{
  ...
  //We add it to the end so it's easier to find. But it's not a requirement.
  PR_CTR(ENT_SINGLE_EVALD_ARG_ctr);
}
```

## Step 2: Actually using the counter

This is also the ["Draw the rest of the owl"](https://www.reddit.com/r/restofthefuckingowl/) step.

Once you found the right place to insert the counter, it's as simple as adding a call to `tickySingleEvalArg` in the right place.

In our case we know we want to deal with single argument closures.  
Sadly it's not quite trivial as we have to figure out where we generate the code we want to investigate.

### Sidequest: Figuring out where to call the ticky function

We know we want to modify the generation of entry code for arguments of closures.

With some digging we can find out that `emitEnter` is responsible for generating the entry code.  
However neither inside, nor at the callsite of `emitEnter` we know if we actually enter an argument
or a "regular" id.

However we CAN hack that contextext into GHC. The Cmm codegen is setup as a Statemonad. We emit Cmm code storing it as we go, and carry around a bit of state
needed for context. So we add the list of arguments of the current closure to this state:

```haskell
data CgInfoDownwards        -- information only passed *downwards* by the monad
  = MkCgInfoDown {
        cgd_dflags    :: DynFlags,
        cgd_mod       :: Module,            -- Module being compiled
        cgd_updfr_off :: UpdFrameOffset,    -- Size of current update frame
        cgd_ticky     :: CLabel,            -- Current destination for ticky counts
        cgd_sequel    :: Sequel,            -- What to do at end of basic block
        cgd_self_loop :: Maybe SelfLoopInfo,-- Which tail calls can be compiled
                                            -- as local jumps? See Note
                                            -- [Self-recursive tail calls] in
                                            -- StgCmmExpr
        cgd_closure_args :: Maybe [NonVoid Id], -- ^ Arguments to the closure
                                                -- we are compiling.
        cgd_tick_scope:: CmmTickScope       -- Tick scope for new blocks & ticks
  }
```

`cgd_closure_args` is new and will allow us to figure out if the id we are looking at is an/one of the arguments.
We can look at the usage sites of `cgd_self_loop` for a template of how and where to adjust the state.

I've decided to call `tickySingleEvalArg` at the call site of emitEnter instead of inside it.
The only reason is that inside emiteEnter we only have a Cmm representation of what we enter,
so there is no way to compare it to an argument. Even better would be to pass the ID and do it
inside, but not worth it for a temporary thing.

The use site inside `cgIdApp` then looks like this:
```haskell
  EnterIt -> do MASSERT( null args )  -- Discarding arguments
                args <- getClosureArgs
                let evalSingleArg
                    | Just [arg] <- args
                    , fromNonVoid arg == fun_id
                    = True
                    | otherwise
                    = False
                when (evalSingleArg) $ do
                  tickySingleEvalArg
                  pprTraceM "evalSingleArg" (ppr fun_id)
                emitEnter fun
```

If the current closure has only a single argument we emit additional code before the tag-check&enter code
which will bump our counter.

In other words whenever we check if we have to evaluate the argument of a single-argument closure we will also bump our counter.
Let's put it to the test!

## Step 3: Wait a long time while GHC builds

## Step 4: Let's ticky some code

The easiest way is arguably to test it on [nofib](https://gitlab.haskell.org/ghc/ghc/wikis/building/running-no-fib).

We simply pass `-ticky` on compilation and `+RTS -r -RTS` when running the benchmarks.
The full invocation then is:

`make clean && make boot && make EXTRA_HC_OPTS="-ticky" SRC_RUNTEST_OPTS="+RTS -r -RTS"`

Running the benchmarks then will create `<executableName>.ticky` files in the folder we started them in.
We will just grep for our counters name across all the files to see if this is a super rare occurence or
pretty common.

```
find nofib/ -name "*.ticky" -exec cat {} + | less | grep ENT_SINGLE_EVALD_ARG_ctr
        357 ENT_SINGLE_EVALD_ARG_ctr
          0 ENT_SINGLE_EVALD_ARG_ctr
          0 ENT_SINGLE_EVALD_ARG_ctr
      19684 ENT_SINGLE_EVALD_ARG_ctr
   11350500 ENT_SINGLE_EVALD_ARG_ctr
    2000000 ENT_SINGLE_EVALD_ARG_ctr
          0 ENT_SINGLE_EVALD_ARG_ctr
   44083000 ENT_SINGLE_EVALD_ARG_ctr
   51841700 ENT_SINGLE_EVALD_ARG_ctr
          0 ENT_SINGLE_EVALD_ARG_ctr
          0 ENT_SINGLE_EVALD_ARG_ctr
          0 ENT_SINGLE_EVALD_ARG_ctr
    3207600 ENT_SINGLE_EVALD_ARG_ctr
   38860600 ENT_SINGLE_EVALD_ARG_ctr
          0 ENT_SINGLE_EVALD_ARG_ctr
   28898018 ENT_SINGLE_EVALD_ARG_ctr
       7501 ENT_SINGLE_EVALD_ARG_ctr
   29307600 ENT_SINGLE_EVALD_ARG_ctr
    1715001 ENT_SINGLE_EVALD_ARG_ctr
    9181600 ENT_SINGLE_EVALD_ARG_ctr
     292433 ENT_SINGLE_EVALD_ARG_ctr
     910000 ENT_SINGLE_EVALD_ARG_ctr
    2852392 ENT_SINGLE_EVALD_ARG_ctr
   19317744 ENT_SINGLE_EVALD_ARG_ctr
   53611457 ENT_SINGLE_EVALD_ARG_ctr
   20488320 ENT_SINGLE_EVALD_ARG_ctr
   24227544 ENT_SINGLE_EVALD_ARG_ctr
   14727300 ENT_SINGLE_EVALD_ARG_ctr
   36950000 ENT_SINGLE_EVALD_ARG_ctr
    6956726 ENT_SINGLE_EVALD_ARG_ctr
   19065839 ENT_SINGLE_EVALD_ARG_ctr
     367599 ENT_SINGLE_EVALD_ARG_ctr
    3860002 ENT_SINGLE_EVALD_ARG_ctr
    6817064 ENT_SINGLE_EVALD_ARG_ctr
    6113200 ENT_SINGLE_EVALD_ARG_ctr
    2894700 ENT_SINGLE_EVALD_ARG_ctr
    1443603 ENT_SINGLE_EVALD_ARG_ctr
    5859305 ENT_SINGLE_EVALD_ARG_ctr
      11400 ENT_SINGLE_EVALD_ARG_ctr
   18547800 ENT_SINGLE_EVALD_ARG_ctr
    4720001 ENT_SINGLE_EVALD_ARG_ctr
   17240002 ENT_SINGLE_EVALD_ARG_ctr
    1380925 ENT_SINGLE_EVALD_ARG_ctr
   61236387 ENT_SINGLE_EVALD_ARG_ctr
   14336200 ENT_SINGLE_EVALD_ARG_ctr
    9254291 ENT_SINGLE_EVALD_ARG_ctr
   10237993 ENT_SINGLE_EVALD_ARG_ctr
    4080001 ENT_SINGLE_EVALD_ARG_ctr
    3560000 ENT_SINGLE_EVALD_ARG_ctr
     818500 ENT_SINGLE_EVALD_ARG_ctr
....................................

```

Looks pretty common at a glance!

## Step5: Digging in

We now know:

* Functions with just one argument are called pretty often.
* If the argument is already evaluated we can improve on the performance.

But we have no idea if:

* Evaluated arguments are common in these cases.
* If the numbers above are actually high in the context of their benchmarks.

### Making sense of the numbers.

We will chose to make sense of the numbers with some napkin (calculator) math.

* parstof, the benchmark with the 60 Million entries for our counter takes ~1.5 seconds to run when optimized.
* One cycle on my cpu takes around 0.25 nanoseconds.

If each entry get's one cycle faster that's 15ms saved, or about 1%. That's not huge but
a decent speedup for a straight forward optimization.

Now maybe it would shave of 40 cycles, or maybe all arguments are lazy. Either way there is the chance that
this will be useful so it's worth digging deeper.

## Step6: A better counter.

Now what we want to do is the following:

* On each entry check if the argument is evaluated.
* Only if it's evaluated bump another counter.

Thankfully there is this helpful function:  
```haskell
bumpTickyCounterByE :: FastString -> CmmExpr -> FCode ()
bumpTickyCounterByE lbl = bumpTickyLblByE (mkCmmDataLabel rtsUnitId lbl)
```

It allows us to bump another counter by the value of a CmmExpr so we can contrast the evaluated and unevaluated cases.  
However since I am lazy I will just reuse the existing counter, but only bumping it when the argument is also evaluated.

This is fine as I can compare the results to earlier runs.

So we replace `tickySingleEvalArg` with `tickySingleEvalArgBy`

```haskell
tickySingleEvalArgBy :: CmmExpr -> FCode ()
tickySingleEvalArgBy = ifTicky $ bumpTickyCounterByE (fsLit "ENT_SINGLE_EVALD_ARG_ctr")
```

GHC is nice enough enough to translate boolean Cmm expressions to the values 0/1 when we
use such an expression where a value is expected. So we only have to come up with an Cmm expression
which tells us if the argument is already evaluated. Something we can use `(cmmIsTagged dflags fun)`
for.

So the use site doesn't change that much, we only pass an expression checking for a pointer
tag as the argument to our new ticky function.

```haskell
    args <- getClosureArgs
    let evalSingleArg
        | Just [arg] <- args
        , fromNonVoid arg == fun_id
        = True
        | otherwise
        = False
    when (evalSingleArg) $ do
      tickySingleEvalArgBy (cmmIsTagged dflags fun)
      pprTraceM "evalSingleArg" (ppr fun_id)
    emitEnter fun
```

## Step7: Final results

Running nofib we get similar, but still quite encouraging results.
In particular parstof still has very high entry counts for the single argument + evaluated combination.

Now ~52M instead of 60M entries. Sure fewer (which is expected) but more than I would have guessed.
So bottom line is it seems worth to do this optimization!

Overall even evaluated single arguments seem somewhat common. Here are the counter results for the first 10 benchmarks:

```
$ find nofib/ -name "*.ticky" -exec cat {} + | less | grep ENT_SINGLE_EVALD_ARG_ctr
         17 ENT_SINGLE_EVALD_ARG_ctr
          0 ENT_SINGLE_EVALD_ARG_ctr
          0 ENT_SINGLE_EVALD_ARG_ctr
          4 ENT_SINGLE_EVALD_ARG_ctr
    7183413 ENT_SINGLE_EVALD_ARG_ctr
    2000000 ENT_SINGLE_EVALD_ARG_ctr
          0 ENT_SINGLE_EVALD_ARG_ctr
   24495211 ENT_SINGLE_EVALD_ARG_ctr
   51741600 ENT_SINGLE_EVALD_ARG_ctr
          0 ENT_SINGLE_EVALD_ARG_ctr
            ...
```
