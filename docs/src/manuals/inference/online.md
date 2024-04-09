# [Online inference](@id manual-online-inference)

This guide explains how to use the [`infer`](@ref) function for dynamic datasets. We'll show how `RxInfer` can continuously update beliefs asynchronously whenever a new observation arrives. We'll use a simple Beta-Bernoulli model as an example, which has been covered in the [Getting Started](@ref user-guide-getting-started) section, 
but keep in mind that these techniques can apply to any model.

## [Model specification](@id manual-online-inference-model-spec)

In online inference, we want to continuously update our prior beliefs about certain hidden states. 
To achieve this, we include extra arguments in our model specification to allow for dynamic prior changes:

```@example manual-online-inference
using RxInfer
using Test #hide

@model function beta_bernoulli_online(y, a, b)
    θ ~ Beta(a, b)  
    y ~ Bernoulli(θ)
end
```

In this model, we assume we only have one observation y at a time, and the a and b parameters are not fixed to specific values but rather are arguments of the model itself.

## [Automatic prior update](@id manual-online-inference-autoupdates)

Next, we want to enable `RxInfer` to automatically update the `a` and `b` parameters as soon as a new posterior for `θ` is available. To accomplish this, we utilize the [`@autoupdates`](@ref) macro.

```@example manual-online-inference
beta_bernoulli_autoupdates = @autoupdates begin 
    # We want to update `a` and `b` to be equal to the parameters 
    # of the current posterior for `θ`
    a, b = params(q(θ))
end
```

This specification instructs `RxInfer` to update `a` and `b` parameters automatically as as soon as new posterior for `θ` is available.

```@docs
@autoupdates
```

## [Asynchronous data stream of observations](@id manual-online-inference-async-datastream)

For demonstration purposes, we will use hand crafted stream of observations using the `Rocket.jl` library:

```@example manual-online-inference
using Rocket, Distributions, StableRNGs

hidden_θ     = 1 / 3.1415
distribution = Bernoulli(hidden_θ)
rng          = StableRNG(43)
datastream   = RecentSubject(Bool)
```

The [`infer`](@ref) function expects the `datastream` to emit values in the form of the `NamedTuple`s. To simply this process, `Rocket` exports `labeled` function:

```@example manual-online-inference
# We convert the stream of `Bool` to the stream of `Float64`, because the `Bernoulli` node
# from `ReactiveMP` expects `Float64` inputs
observations = labeled(Val((:y, )), combineLatest(datastream |> map(Float64, float)))
```

Let's verify that our datastream, does indeed produce `NamedTuple`

```@example manual-online-inference
test_values = [] #hide
test_subscription = subscribe!(observations, (new_observation) -> push!(test_values, new_observation)) #hide
subscription = subscribe!(observations, 
    (new_observation) -> println("Got new observation `", new_observation, "` 🎉")
)
```

```@example manual-online-inference
for i in 1:5
    next!(datastream, rand(rng, distribution))
end
@test length(test_values) === 5 #hide
@test all(value -> haskey(value, :y) && (isone(value[:y]) || iszero(value[:y])), test_values) #hide 
nothing #hide
```

Nice! Our data stream produces events in a form of the `NamedTuple`s, which is compatible with the [`infer`](@ref) function.

```@example manual-online-inference
unsubscribe!(test_subscription) #hide
# It is important to keep track of the existing susbcriptions
# and unsubscribe to reduce the usage of computational resources
unsubscribe!(subscription)
```

## [Instantiating the reactive inference engine](@id manual-online-inference-inst-reactive-engine)

Now, we have everything ready to start running the inference with `RxInfer` on dynamic datasets with the [`infer`](@ref) function:

```@example manual-online-inference
engine = infer(
    model         = beta_bernoulli_online(),
    datastream    = observations,
    autoupdates   = beta_bernoulli_autoupdates,
    initmarginals = (θ = Beta(1, 1), ),
    autostart     = false
)
```

In the code above, there are several notable differences compared to running inference for static datasets. Firstly, we utilized the `autoupdates` argument as discussed [previously](@ref manual-online-inference-autoupdates). Secondly, we employed the `initmarginals` function to initialize the posterior over `θ`. This is necessary for the `@autoupdates` macro, as it needs to initialize the `a` and `b` parameters before the data becomes available. Thirdly, we set `autostart = false` to indicate that we do not want to immediately subscribe to the datastream, but rather do so manually later using the [`RxInfer.start`](@ref) function.

```@docs
RxInferenceEngine
RxInfer.start
RxInfer.stop
```

Given the `engine`, we now can subscribe on the posterior updates:

```@example manual-online-inference
θ_updates_for_testing_the_example  = [] #hide
θ_updates_for_testing_subscription = subscribe!(engine.posteriors[:θ], (new_posterior_for_θ) -> push!(θ_updates_for_testing_the_example, new_posterior_for_θ)) #hide
θ_updates_subscription = subscribe!(engine.posteriors[:θ], 
    (new_posterior_for_θ) -> println("A new posterior for θ is `", new_posterior_for_θ, "` 🤩")
)
nothing #hide
```

In this setting, we should get a message every time a new posterior is available for `θ`. Let's try to generate a new observation!

```@example manual-online-inference
next!(datastream, rand(rng, distribution))
@test isempty(θ_updates_for_testing_the_example) #hide
nothing #hide
```

Hmm, nothing happened...? Oh, we forgot to _start_ the engine with the [`RxInfer.start`](@ref) function. Let's do that now:

```@example manual-online-inference
RxInfer.start(engine)
@test length(θ_updates_for_testing_the_example) === 1 #hide
nothing #hide
```

Ah, as soon as we start our engine, we receive the posterior for `θ`. This occurred because we initialized our stream as `RecentSubject`, which retains the most recent value and emits it upon subscription. Our engine automatically subscribed to the observations and obtained the most recent value, initiating inference. Let's see if we can add more observations:

```@example manual-online-inference
next!(datastream, rand(rng, distribution))
@test length(θ_updates_for_testing_the_example) === 2 #hide
nothing #hide
```

Great! We got another posterior! Let's try a few more observations:

```@example manual-online-inference
for i in 1:5
    next!(datastream, rand(rng, distribution))
end
@test length(θ_updates_for_testing_the_example) === 7 #hide
nothing #hide
```

As demonstrated, the reactive engine reacts to new observations and performs inference as soon as a new observation is available. But what if we want to maintain a history of posteriors? The [`infer`](@ref) function supports the `historyvars` and `keephistory` arguments precisely for that purpose.
In the next section we reinstantiate our engine, with the `keephistory` argument enabled, but first, we must shutdown the previous engine and unsubscribe from its posteriors:

```@example manual-online-inference
RxInfer.stop(engine)
unsubscribe!(θ_updates_subscription)
unsubscribe!(θ_updates_for_testing_subscription) #hide
nothing #hide
```

## [Keeping the history of posteriors](@id manual-online-inference-history)

To retain the history of posteriors within the engine, we can utilize the `keephistory` and `historyvars` arguments.
The `keephistory` parameter specifies the length of the circular buffer for storing the history of posterior updates, while `historyvars` determines what variables to save in the history and how often to save them (e.g., every iteration or only at the end of iterations).

```@example manual-online-inference
engine = infer(
    model         = beta_bernoulli_online(),
    datastream    = observations,
    autoupdates   = beta_bernoulli_autoupdates,
    initmarginals = (θ = Beta(1, 1), ),
    keephistory   = 100,
    historyvars   = (θ = KeepLast(), ),
    autostart     = true
)
```

In the example above, we specified that we want to store at most `100` posteriors for `θ`, and `KeepLast()` indicates that we are only interested in the final value of `θ` and not in intermediate values during variational iterations. We also specified the `autostart = true` to start the engine automatically without need for [`RxInfer.start`](@ref) and [`RxInfer.stop`](@ref).

!!! note
    In this model, we do not utilize the `iterations` argument, indicating that we perform a single VMP iteration. If multiple iterations were employed, `engine.posteriors[:θ]` would emit every intermediate value.

Now, we can feed some more observations to the datastream:

```@example manual-online-inference
for i in 1:5
    next!(datastream, rand(rng, distribution))
end
```

And inspect the `engine.history[:θ]` buffer:

```@example manual-online-inference
@test length(engine.history[:θ]) === 6 #hide
engine.history[:θ]
```

As we can see the buffer correctly saved the posteriors in the `.history` buffer.

!!! note 
    We have `6` entries, despite having only `5` new observations. As mentioned earlier, this occurs because we initialized our datastream as a `RecentSubject`, which retains the most recent observation and emits it each time a new subscription occurs.

### [Visualizing the history of posterior estimation](@id manual-online-inference-history-visualization)

Let's feed more observation and visualize how the posterior changes over time:

```@example manual-online-inference
for i in 1:94
    next!(datastream, rand(rng, distribution))
end
@test length(engine.history[:θ]) === 100 #hide
nothing #hide
```

To visualize the history of posteriors we use the `@gif` macro from the `Plots` package:
```@example manual-online-inference
using Plots

@gif for posterior in engine.history[:θ]
    rθ = range(0, 1, length = 1000)
    pθ = plot(rθ, (x) -> pdf(posterior, x), fillalpha=0.3, fillrange = 0, label="P(θ|y)", c=3)
    pθ = vline!(pθ, [ hidden_θ ], label = "Real value of θ")

    plot(pθ)
end
```

We can keep feeding data to our datastream, but only last `100` posteriors will be saved in the `history` buffer:

```@example manual-online-inference
for i in 1:200
    next!(datastream, rand(rng, distribution))
end

@test length(engine.history[:θ]) === 100 #hide
@gif for posterior in engine.history[:θ]
    rθ = range(0, 1, length = 1000)
    pθ = plot(rθ, (x) -> pdf(posterior, x), fillalpha=0.3, fillrange = 0, label="P(θ|y)", c=3)
    pθ = vline!(pθ, [ hidden_θ ], label = "Real value of θ")

    plot(pθ)
end
```

!!! note
    It is also possible to visualize the inference estimation continously with manual subscription to `engine.posteriors[:θ]`.


As previously it is important to shutdown the inference engine when it becomes unnecessary:
```@example manual-online-inference
RxInfer.stop(engine)
```

## [Subscribing on the stream of free energy](@id manual-online-inference-free-energy)


To obtain a continuous stream of updates for the [Bethe Free Energy](@ref lib-bethe-free-energy), we need to initialize the engine with the `free_energy` argument set to `true`:

```@example manual-online-inference
engine = infer(
    model         = beta_bernoulli_online(),
    datastream    = observations,
    autoupdates   = beta_bernoulli_autoupdates,
    initmarginals = (θ = Beta(1, 1), ),
    keephistory   = 5,
    autostart     = true,
    free_energy   = true
)
```

!!! note 
    It's important to use the `keephistory` argument alongside the `free_energy` argument because setting `free_energy = true` also maintains an internal circular buffer to track its previous updates.

```@example manual-online-inference
free_energy_for_testing = [] #hide
free_energy_for_testing_subscription = subscribe!(engine.free_energy, (v) -> push!(free_energy_for_testing, v)) #hide
free_energy_subscription = subscribe!(engine.free_energy, 
    (bfe_value) -> println("New value of Bethe Free Energy has been computed `", bfe_value, "` 👩‍🔬")
)
@test length(free_energy_for_testing) === 1 #hide
nothing #hide
```

Let's emit more observations:

```@example manual-online-inference
for i in 1:5
    next!(datastream, rand(rng, distribution))
end
@test length(free_energy_for_testing) === 6 #hide
nothing #hide
```

In this particular example, we do not perform any variational iterations and do not use any variational constraints, hence, the inference is exact.
In this case the BFE values are equal to the minus log-evidence of the model given new observation. 
We can also track history of Bethe Free Energy values with the following fields of the `engine`:
- `free_energy_history`: free energy history, averaged across variational iterations value for all observations  
- `free_energy_raw_history`: free energy history, returns returns computed values of all variational iterations for each data event (if available)
- `free_energy_final_only_history`: free energy history, returns computed values of final variational iteration for each data event (if available)

```@example manual-online-inference
@test length(engine.free_energy_history) === 1 #hide
engine.free_energy_history
```

```@example manual-online-inference
@test length(engine.free_energy_raw_history) === 5 #hide
engine.free_energy_raw_history
```

```@example manual-online-inference
@test length(engine.free_energy_final_only_history) === 5 #hide
engine.free_energy_final_only_history
```

```@example manual-online-inference
unsubscribe!(free_energy_for_testing_subscription) #hide
# Stop the engine when not needed as usual
RxInfer.stop(engine)
unsubscribe!(free_energy_subscription)
```

As has been mentioned, in this particular example we do not perform variational iterations, hence, there is little different between different representations of the BFE history buffers. However, when performing variational inference with the `iterations` argument, those buffers will be different. To demonstrate this difference let's build a slightly more complex model with variational constraints:

```@example manual-online-inference
@model function iid_normal(y, mean_μ, var_μ, shape_τ, rate_τ)
    μ ~ Normal(mean = mean_μ, var = var_μ)
    τ ~ Gamma(shape = shape_τ, rate = rate_τ)
    y ~ Normal(mean = μ, precision = τ)
end

iid_normal_constraints = @constraints begin
    q(μ, τ) = q(μ)q(τ)
end

iid_normal_autoupdates = @autoupdates begin 
    mean_μ  = mean(q(μ))
    var_μ   = var(q(μ))
    shape_τ = shape(q(τ))
    rate_τ  = rate(q(τ))
end

iid_normal_hidden_μ     = 3.1415
iid_normal_hidden_τ     = 0.0271
iid_normal_distribution = NormalMeanPrecision(iid_normal_hidden_μ, iid_normal_hidden_τ)
iid_normal_rng          = StableRNG(123)
iid_normal_datastream   = RecentSubject(Float64)
iid_normal_observations = labeled(Val((:y, )), combineLatest(iid_normal_datastream))

iid_normal_engine = infer(
    model         = iid_normal(),
    datastream    = iid_normal_observations,
    autoupdates   = iid_normal_autoupdates,
    constraints   = iid_normal_constraints,
    initmarginals = (
        μ = NormalMeanPrecision(0.0, 0.001),
        τ = GammaShapeRate(10.0, 10.0)
    ),
    historyvars   = (
        μ = KeepLast(),
        τ = KeepLast(),
    ),
    keephistory   = 100,
    iterations    = 10,
    free_energy   = true,
    autostart     = true
)
```

The notable differences with the previous example is the use of the `constraints` and `iterations` arguments. Read more about constraints in the [Constraints Specification](@ref user-guide-constraints-specification) section of the documentation. We have also indicated in the `historyvars` that we want to keep track of posteriors only from the last variational iteration in the history buffer.

Now we can feed some observations to the datastream:
```@example manual-online-inference
for i in 1:100
    next!(iid_normal_datastream, rand(iid_normal_rng, iid_normal_distribution))
end
```

Let's inspect the differences in the `free_energy` buffers:

```@example manual-online-inference
@test length(iid_normal_engine.free_energy_history) === 10 #hide
iid_normal_engine.free_energy_history
```

```@example manual-online-inference
@test length(iid_normal_engine.free_energy_raw_history) === 1000 #hide
iid_normal_engine.free_energy_raw_history
```

```@example manual-online-inference
@test length(iid_normal_engine.free_energy_final_only_history) === 100 #hide
iid_normal_engine.free_energy_final_only_history
```

We can also visualize different representations:
```@example manual-online-inference
plot(iid_normal_engine.free_energy_history, label = "Bethe Free Energy (averaged)")
```

```@example manual-online-inference
plot(iid_normal_engine.free_energy_raw_history, label = "Bethe Free Energy (raw)")
```

```@example manual-online-inference
plot(iid_normal_engine.free_energy_final_only_history, label = "Bethe Free Energy (last per observation)")
```

As we can see, in the case of the variational iterations those buffers are quite different and represent different representations
of the same Bethe Free Energy stream. As a sanity check, we could also visualize the history of our posterior estimations in the same way 
as we did for a simpler previous example:

```@example manual-online-inference
@test length(iid_normal_engine.history[:μ]) === 100 #hide
@test length(iid_normal_engine.history[:τ]) === 100 #hide
@gif for (μ_posterior, τ_posterior) in zip(iid_normal_engine.history[:μ], iid_normal_engine.history[:τ])
    rμ = range(0, 10, length = 1000)
    rτ = range(0, 1, length = 1000)

    pμ = plot(rμ, (x) -> pdf(μ_posterior, x), fillalpha=0.3, fillrange = 0, label="P(μ|y)", c=3)
    pμ = vline!(pμ, [ iid_normal_hidden_μ ], label = "Real value of μ")

    pτ = plot(rτ, (x) -> pdf(τ_posterior, x), fillalpha=0.3, fillrange = 0, label="P(τ|y)", c=3)
    pτ = vline!(pτ, [ iid_normal_hidden_τ ], label = "Real value of τ")

    plot(pμ, pτ, layout = @layout([ a; b ]))
end
```

Nice, the history of the estimated posteriors aligns well with the real (hidden) values of the underlying parameters.

## [Callbacks and the event loop](@id manual-online-inference-event-loop)

```@example manual-online-inference
@test false
```