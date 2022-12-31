# julia-notes

- expr && action_if_true
- expr || action_if_false

- ⊆ (issubset())

- Don't use variable names that are same with function names, even if the variables are locally scoped (e.g. inside a function)
- Concrete function implementations are called methods in Julia

- BenchmarkTools: @btime and @benchmark
- JET: watch_and_record_file()

- use @edit to quickly find the source code implementation: @edit ⊆([1], [1, 2])
- There's also @code_lowered, @code_typed, @code_warntype, @code_llvm, @code_native (see https://docs.julialang.org/en/v1/devdocs/reflection/#Intermediate-and-compiled-representations)
- There's also Meta.@dump and Meta.@lower

- use generators (different from Python's, based on iterators instead of coroutines): Dict(base => 0 for base in "ACGT")

- Bounds checks are turned off with @inbounds, this removes some hidden branches (normally every array access in Julia has a little check that the index is valid first)
- Iterate a vector of bytes instead of a string (keep tabs on unicode-safety). Iterating characters from a String involves doing extra work every iteration to establish if the current byte is the start of a multibyte sequence (and, if so to reassemble that sequence into its proper 32 bit character)
- The loop body is extremely simple and fast on x86 processors (load a value, increment a value at an offset*) and branchless (there are no conditionals within the loop body). Branchless code is often faster because branching hinders CPU instruction level parallelism and stops compilers using "Single Instruction Multiple Data" (SIMD) instructions

- Instead of globals, pass things as function arguments
- Avoid globals. Specially for parameters. Just use a Dict, or create your own struct and pass it along (there some package solutions too). I use globals. I have a global timer in my module, so I can reset it, run a bunch of my methods, and then get how much was spent in each method (and each method called inside them). It is the best solution? No. But it is good enough and used only for debug, not for anything that will get in a paper.
- Considering that you will use globals. If you may need to change the entire object including its type: do not use const (and take the performance hit). If the type never changes, but you may need to change the whole object, not just a field, then use something like const var = Ref(10) for storing an Int, this creates a const binding to a reference (you need to use [] to access, or set, the Int value), so you get type-stability and gets to change the value safely. If you have a lot of these globals (type-fixed but with an object that may need to be wholly replaced), then instead of using Ref you may also declare your own mutable struct and use it with const and without the Ref (because the fields are mutable and can be changed safely). If you are completely sure you will never need to replace the entire object but only mutate a fields of it (or call mutating methods over it), then you use const (directly over the value, not wrapping inside a Ref).
- I do not think the main point of using structs is efficiency, but just that you are following (1) by doing this. Unless you are talking about using a global struct to store the values, I have addressed this in the point above. Basically, if const was used (one way or another) then the performance of the global variable will not be so much different from the passed-along parameters (but I welcome someone showing me wrong).
- If you have a grouping of fields that never change (or change very rarely), or they change but almost all fields in the same point (so it is almost the same cost of creating the struct again), then immutable structs are a good idea. If you need to consistently change the values of single fields, not so good. If you actually have a loop that needs to change a single field in that structure multiple times, then surely use a mutable struct.
- Minimize the number of parameters defined outside a function (basically (1) again).

- Separate kernel functions (aka, function barriers) https://docs.julialang.org/en/v1/manual/performance-tips/#kernel-functions (modularity, multiple dispatch feels natural, function barriers)
- Code is usually easier to write using a mutating function, because you don't have to recreate the structure of the object you're dealing with. However, users might not want to use a mutating function, because that would mean they wouldn't be able to recover the original object. To circumvent this, you can provide the users with a non-mutating function, by doing:
  ```julia
  function foo(ob)
    ob2 = deepcopy(ob)
    foo!(ob2)
    return ob2
  end
  ```
- `isa Type` checks are usually an anti-pattern; use dispatch instead

- use sizehint!
- use push! for unsorted collections, append! for sorted (TODO: confirm)

- Use UInts or Gandalf's strings if you want to edit strings because unicode

- views??

Assume that you have something like the following:
Depending on the type of an argument (here y), I want to call a function with every argument staying the same except the one where I pass `y` (`arg_that_differs1`, vs `arg_that_differs2`).
The thing is that these two arguments (`arg_that_differs1` and `arg_that_differs2`) have different names, so I end up having to essentially repeat almost all of the code.
Are there ways around this?

```julia
function myfun(x, y::TypeA)
    plot(
        x; 
        arg_that_differs1 = y
        arg_that_is_same1 = "foo"
        arg_that_is_same2 = "bar"
        ...
    )
end

function myfun(x, y::TypeB)
    plot(
        x; 
        arg_that_differs2 = y
        arg_that_is_same1 = "foo"
        arg_that_is_same2 = "bar"
        ...
    )
end
```

The original piece of code was something akin to:

```julia
function vismetabolism(mappath, reaction_edge_color)
    function _escherplot!(reaction_edge_color::Symbol)
        escherplot!(
            mappath; 
            reaction_show_text = true,
            reaction_edge_color = reaction_edge_color,
            metabolite_show_text = true,
            metabolite_node_colors = Dict("glc__D_e" => :red),
            metabolite_node_color = :lightskyblue,
        )

        return nothing
    end

    function _escherplot!(reaction_edge_colors::AbstractDict)
        escherplot!(
            mappath; 
            reaction_show_text = true,
            reaction_edge_colors = reaction_edge_colors,
            metabolite_show_text = true,
            metabolite_node_colors = Dict("glc__D_e" => :red),
            metabolite_node_color = :lightskyblue,
        )

        return nothing
    end

    ... # do stuff
end
```

---

Attempt 1:

```julia
function _vismetabolism(mappath; args...)
    _escherplot!(reaction_edge_color)
    escherplot!(
        mappath; 
        reaction_show_text = true,
        metabolite_show_text = true,
        metabolite_node_colors = Dict("glc__D_e" => :red),
        metabolite_node_color = :lightskyblue,
        args...,
    )

    ... # do stuff
end

function vismetabolism(mappath; reaction_edge_color::Symbol)
    return _vismetabolism(mappath, reaction_edge_color = reaction_edge_color)
end

function vismetabolism(mappath; reaction_edge_colors::AbstractDict)
    return _vismetabolism(mappath, reaction_edge_colors = reaction_edge_colors)
end
```

---

Attempt 2:

```julia
function vismetabolism(mappath, reaction_edge_color::T) where {T<:Union{Symbol, AbstractDict}}
    insert::Expr
    if reaction_edge_color isa Symbol
        insert = :(reaction_edge_color = $reaction_edge_color,)
    elseif reaction_edge_color isa AbstractDict
        insert = :(reaction_edge_colors = $reaction_edge_color,)
    else
        insert = :()
    end

    eval(quote
        escherplot!(
            mappath;
            $(insert),
            reaction_show_text = true,
            metabolite_show_text = true,
            metabolite_node_colors = Dict("glc__D_e" => :red),
            metabolite_node_color = :lightskyblue
        )
    end)

    ... # do stuff
end
```

---

Attempt 3:

```julia
function vismetabolism(mappath, reaction_edge_color::T) where {T<:Union{Symbol, AbstractDict}}
    if T <: Symbol
        kwargs = (reaction_edge_color = reaction_edge_color,)
    elseif T <: AbstractDict
        kwargs = (reaction_edge_colors = reaction_edge_color,)
    else
        kwargs = ()
    end

    escherplot!(
        mappath;
        kwargs...,
        reaction_show_text = true,
        metabolite_show_text = true,
        metabolite_node_colors = Dict("glc__D_e" => :red),
        metabolite_node_color = :lightskyblue
    )

    ... # do stuff
end
```

---

* `show(stdout, "text/plain", x)` to print `x` in it's entirety (e.g. if it's a vector with a lot of elements)

---

So there's a lot of (some very, very good) packages for currying, partial application, functors, lenses, etc.
If you want to define a partial function apparently you can also simply do:
```julia
partial_func = (x) -> other_func(x, y)
```
for example:
```julia
approx = (x) -> round(x; digits = abs(Int(round(log(10, tol)))))
```

Of course you can have more than one arguments, e.g.
```julia
(a,c) -> f(a,b,c)
```

---

To have the RNG pull a number from a set other than the default $[1, 0]$, say from the interval `[a, b]`:
```julia
using Distributions
rand(Uniform(a, b))
```

---

Supported Pluto Markdown Admonitions:
- `danger`
- `warning`
- `info`/`note`/`hint`
- `correct`/`tip`
