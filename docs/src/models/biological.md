# Chemical Reaction Models

The biological models functionality is provided by `DiffEqBiological.jl` and helps
the user to build discrete stochastic and differential equation based systems
biological models. These tools allow one to define the models at a high level
by specifying reactions and rate constants, and the creation of the actual problems
is then handled by the modelling package.

## The Reaction DSL - Basic 
This section covers some of the basic syntax for building chemical reaction network models.
#### Basic syntax

The `@reaction_network` macro allows you to define reaction networks in a more
scientific format. Its input is a set of chemical reactions and from them it
generates a reaction network object which can be used as input to `ODEProblem`,
`SDEProblem` and `JumpProblem` constructors.

The basic syntax is:

```julia
rn = @reaction_network begin
  2.0, X + Y --> XY               
  1.0, XY --> Z1 + Z2            
end
```

where each line corresponds to a chemical reaction. Each reaction consists of a reaction rate (the expression on the left hand side of  `,`), a set of substrates (the expression in-between `,` and `-->`), and a set of products (the expression on the right hand side of `-->`). The substrates and the products may contain one or more reactants, separated by `+`.  The naming convention for these are the same as for normal variables in Julia.

The chemical reaction model is generated by the `@reaction_network` macro and stored in the `rn` variable (a normal variable, do not need to be called `rn`). The macro generates a differential equation model according to the law of mass action, in the above example the ODEs become:

```math
\frac{d[X]}{dt} = -2.0[X]\cdot [Y]\\
\frac{d[Y]}{dt} = -2.0[X]\cdot [Y]\\
\frac{d[XY]}{dt} = 2.0[X]\cdot [Y] - 1.0[XY]\\
\frac{d[Z1]}{dt}= 1.0[XY]\\
\frac{d[Z2]}{dt} = 1.0[XY]\\
```

#### Arrow variants 
Several types of arrows are accepted by the DSL and works instead of `-->`. All of these works:  `>`, `→` `↣`, `↦`, `⇾`, `⟶`, `⟼`, `⥟`, `⥟`, `⇀`, `⇁`. Backwards arrows can also be used to write the reaction in the opposite direction. Hence all of these three reactions are equivalent:

```julia
rn = @reaction_network begin
  1.0, X + Y --> XY               
  1.0, X + Y → XY      
  1.0, XY ← X + Y      
end
```
(note that due to technical reasons `<--` cannot be used)

#### Using bi-directional arrows
Bi-directional arrows can be used to designate a reaction that goes two ways. These two models are equivalent:
```julia
rn = @reaction_network begin
  2.0, X + Y → XY             
  2.0, X + Y ← XY          
end
rn = @reaction_network begin
  2.0, X + Y ↔ XY               
end
```
If the reaction rate in the backwards and forwards directions are different they can be designated in the following way:
```julia
rn = @reaction_network begin
  (2.0,1.0) X + Y ↔ XY               
end
```
which is identical to
```julia
rn = @reaction_network begin
  2.0, X + Y → XY             
  1.0, X + Y ← XY          
end
```

#### Combining several reactions in one line
Several similar reactions can be combined in one line by providing a tuple of reaction rates and/or substrates and/or products. If several tuples are provided they much all be of identical length. These pairs of reaction networks are all identical:
```julia
rn1 = @reaction_network begin
  1.0, S → (P1,P2)               
end
rn2 = @reaction_network begin
  1.0, S → P1     
  1.0, S → P2 
end
```
```julia
rn1 = @reaction_network begin
  (1.0,2.0), (S1,S2) → P             
end
rn2 = @reaction_network begin
  1.0, S1 → P     
  2.0, S2 → P
end
```
```julia
rn1 = @reaction_network begin
  (1.0,2.0,3.0), (S1,S2,S3) → (P1,P2,P3)        
end
rn2 = @reaction_network begin
  1.0, S1 → P1
  2.0, S2 → P2   
  3.0, S3 → P3  
end
```
This can also be combined with bi-directional arrows in which case separate tuples can be provided for the backward and forward reaction rates separately. These reaction networks are identical
```julia
rn1 = @reaction_network begin
 (1.0,(1.0,2.0)), S ↔ (P1,P2)  
end
rn2 = @reaction_network begin
  1.0, S → P1
  1.0, S → P2
  1.0, P1 → S   
  2.0, P2 → S 
end
```

#### Production and Destruction and Stoichiometry
Sometimes reactants are produced/destroyed from/to nothing. This can be designated using either `0` or `∅`:
```julia
rn = @reaction_network begin
  2.0, 0 → X
  1.0, X → ∅
end
```
Sometimes several molecules of the same reactant is involved in a reaction, the stoichiometry of a reactant in a reaction can be set using a number. Here two species of `X` forms the dimer `X2`:
```julia
rn = @reaction_network begin
  1.0, 2X → X2
end
```
this corresponds to the differential equation:

```math
d[X]/dt = -1.0\*[X\]^2/2!
d[X2]/dt = 1.0\*[X\]^2/2!
```

Other numbers than 2 can be used and parenthesises can be used to use the same stoichiometry for several reactants:
```julia
rn = @reaction_network begin
  1.0, X + 2(Y + Z) → XY2Z2
end
```

#### Variable reaction rates
Reaction rates do not need to be constant, but can also depend on the current concentration of the various reactants (when e.g. one reactant activate the production of another one). E.g. this is a valid notation:
```julia
rn = @reaction_network begin
  X, Y → ∅
end
```
and will have `Y` degraded at rate 

```math
d[Y]/dt = -[X]\*[Y]
```

Note that this is actually equivalent to the reaction

```julia
rn = @reaction_network begin
  1.0, X + Y → X
end
```
Most expressions and functions are valid reaction rates, e.g:
```julia
rn = @reaction_network begin
  2.0*X^2, 0 → X + Y
  gamma(Y)/5, X → ∅
  pi*X/Y, Y → ∅
end
```

please note that user defined functions cannot be used directly (see later section "User defined functions in reaction rates").

#### Defining parameters
Just as when defining normal differential equations using `DifferentialEquations` parameter values does not need to be set when the model is created. Components can be designated as parameters by declaring them at the end:
```julia
rn = @reaction_network begin
  p, ∅ → X
  d, X → ∅
end p d
```
Parameters can only exist in the reaction rates (where they can be mixed with reactants). All variables not declared at the end will be considered a reactant.

#### Pre-defined functions
The hill function and the michaelis-menten function, both which are common in biochemical reaction networks, are pre-defined and can be used as expected. These pairs of reactions are all equivalent:
```julia
rn1 = @reaction_network begin
  hill(X,v,K,n), ∅ → X
  v*X^n/(X^n+K^n), ∅ → X
end v K n
rn2 = @reaction_network begin
  mm(X,v,K), ∅ → X
  v*X/(X+K), ∅ → X
end v K
```

## Model Simulation

Once created, a reaction network can be used as input to various problem types and the solved by `DifferentialEquations.jl`.

#### Deterministic simulations using ODEs
A reaction network can be used as input to a `ODEProblem` instead of a function, using
```probODE = ODEProblem(rn, args...; kwargs...) ```
E.g. a model can be created and simulated using:
```julia
rn = @reaction_network begin
  p, ∅ → X
  d, X → ∅
end p d
p = [1.0,2.0]
u0 = [0.1]
tspan = (0.,1.)
prob = ODEProblem(rn,u0,tspan,p)
sol = solve(prob)
```
(if no parameters are given `p` does not need to be provided)

#### Stochastic simulations using SDEs
In a similar way a SDE can be created using `probSDE = SDEProblem(rn, args...; kwargs...)`. 
In this case the chemical Langevin equations (as derived in Gillespie 2000) will 
be used to generate stochastic differential equations.

#### Stochastic simulations using discrete stochastic simulation algorithm
Instead of solving SDEs one can make stochastic simulations of the model using real copy numbers and a discrete stochastic simulation algorithm. This can be done using:
```julia
rn = @reaction_network begin
  p, ∅ → X
  d, X → ∅
end p d
p = [1.0,2.0]
u0 = [10]
tspan = (0.,1.)
discrete_prob = DiscreteProblem(u0,tspan,p)
jump_prob = JumpProblem(discrete_prob,Direct(),rn)
sol = solve(jump_prob,FunctionMap())
```


## The Reaction DSL - Advanced 
This section covers some of the more advanced syntax for building chemical reaction network models (still not very complicated!).

#### User defined functions in reaction rates
The reaction network DSL cannot "see" user defined functions. E.g. this is not correct syntax:

```julia
myHill(x) = 2.0*x^3/(x^3+1.5^3)
rn = @reaction_network begin
  myHill(X), ∅ → X
end
```
However, it is possible to define functions in such a way that the DSL can see them 
using the `@reaction_func` macro:

```julia
@reaction_func myHill(x) = 2.0*x^3/(x^3+1.5^3)
rn = @reaction_network begin
  myHill(X), ∅ → X
end
```

#### Defining a custom reaction network type
While the default type of a reaction network is `reaction_network` (which inherits 
from `AbstractReactionNetwork`) it is possible to define a custom type (which also 
will inherit from `AbstractReactionNetwork`) by adding the type name as a first 
argument to the `@reaction_network` macro:

```julia
rn = @reaction_network my_custom_type begin
  1.0, ∅ → X
end
```

#### Scaling noise in the chemical Langevin equations
When making stochastic simulations using SDEs it is possible to scale the amount of noise 
in the simulations by declaring a noise scaling parameter. This parameter is declared as a 
second argument to the `@reaction_network` macro (when scaling the noise one have to 
declare a custom type).

```julia
rn = @reaction_network my_custom_type ns begin
  1.0, ∅ → X
end
```

The noise scaling parameter is automatically added as a last argument to the parameter array 
(even if not declared at the end). E.g. this is correct syntax:

```julia
rn = @reaction_network my_custom_type ns begin
  1.0, ∅ → X
end
p = [0.1,]
u0 = [0.1]
tspan = (0.,1.)
prob = SDEproblem(rn,u0,tspan,p)
sol = solve(prob)
```
Here the amount of noise in the stochastic simulation will be reduced by a factor 10.

#### Ignoring mass kinetics
While one in almost all cases want the reaction rate to take the law of mass action into account, so the reaction
```julia
rn = @reaction_network my_custom_type ns begin
  k, X → ∅
end k
```
occur at the rate ``d[X]/dt = -k\*[X]``, it is possible to ignore this by using any of the following 
non-filled arrows when declaring the reaction: `⇐`, `⟽`, `⇒`, `⟾`, `⇔`, `⟺`. This means that 
the reaction 

```julia
rn = @reaction_network my_custom_type ns begin
  k, X ⇒ ∅
end k
```

will occur at rate ``d[X]/dt = -k`` (which might become a problem since ``[X]`` will be degraded 
at a constant rate even when very small or equal to 0.

## The Reaction Network Object
The `@reaction_network` macro generate a `reaction_network` object which have several fields 
which can be accessed.

* `rn.syms` is a vector containing symbols corresponding to all the reactants of the network.
* `rn.params` is a vector containing symbols corresponding to all the parameters of the network.
* `rn.f_func` is a vector containing expression corresponding to the equations in the ODE corresponding to the model.
* `rn.g_func` is a vector containing expressions corresponding the noise terms used when creating the SDEs (n\*m element where there are n reactants and m reactions. The first m elements contains the noise term for the first reactant and each reaction correspondingly, next for the second reactant and so on).
* `rn.symjac` is the symbolically calculated Jacobian of the ODE corresponding to the model.

 

## Examples

#### Example: Birth-Death Process

```julia
rs = @reaction_network begin
  c1, X --> 2X
  c2, X --> 0
  c3, 0 --> X
end c1 c2 c3
p = (2.0,1.0,0.5)
prob = DiscreteProblem([5], (0.0, 4.0), p)
jump_prob = JumpProblem(prob, Direct(), rs)
sol = solve(jump_prob, Discrete())
```

#### Example: Michaelis-Menten Enzyme Kinetics

```julia
rs = @reaction_network begin
  c1, S + E --> SE
  c2, SE --> S + E
  c3, SE --> P + E
end c1 c2 c3
p = (0.00166,0.0001,0.1)
# S = 301, E = 100, SE = 0, P = 0
prob = DiscreteProblem([301, 100, 0, 0], (0.0, 100.0), p)
jump_prob = JumpProblem(prob, Direct(), rs)
sol = solve(jump_prob, Discrete())
```
