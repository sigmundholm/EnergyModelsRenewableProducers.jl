# EnergyModelsRenewableProducers

<!--- [![Code Style: Blue](https://img.shields.io/badge/code%20style-blue-4495d1.svg)](https://github.com/invenia/BlueStyle)
--->
`EnergyModelsRenewableProducers` is a package to model renewable power generation
technologies. It extends the `EnergyModelsBase` package with non-dispatchable power generation from sources such as wind turbines.

> **Note**
> This is an internal pre-release not intended for distribution outside the project consortium. 

## Usage

Work with documentation using [Documenter](https://juliadocs.github.io/Documenter.jl/stable/) is in progress. See below for an example with generation from wind turbines:

```julia
using EnergyModelsBase
using EnergyModelsRenewableProducers
using HiGHS
using TimeStructures
using JuMP

const EMB = EnergyModelsBase
const RP = EnergyModelsRenewableProducers

function demo_res()
    NG = ResourceEmit("NG", 0.2)
    CO2 = ResourceEmit("CO2", 1.0)
    Power = ResourceCarrier("Power", 0.0)
    products = [NG, Power, CO2]

    # Create dictionary with entries of 0. for all resources
    𝒫₀ = Dict(k => 0 for k ∈ products)

    # Create dictionary with entries of 0. for all emission resources
    𝒫ᵉᵐ₀ = Dict(k => 0.0 for k ∈ products if typeof(k) == ResourceEmit{Float64})
    𝒫ᵉᵐ₀[CO2] = 0.0

    # Create source and sink module as well as the arrays used for nodes and links
    source = RefSource(
        2,
        FixedProfile(1),
        FixedProfile(30),
        FixedProfile(10),
        Dict(NG => 1),
        𝒫ᵉᵐ₀,
        Dict("" => EMB.EmptyData()),
    )
    sink = RefSink(
        3,
        FixedProfile(20),
        Dict(:Surplus => FixedProfile(0), :Deficit => FixedProfile(1e6)),
        Dict(Power => 1),
        𝒫ᵉᵐ₀,
    )
    nodes = [GenAvailability(1, 𝒫₀, 𝒫₀), source, sink]
    links = [
        Direct(21, nodes[2], nodes[1], Linear())
        Direct(13, nodes[1], nodes[3], Linear())
    ]

    # Create time structure and the used global data
    T = UniformTwoLevel(1, 4, 1, UniformTimes(1, 24, 1))
    global_data = EMB.GlobalData(
        Dict(CO2 => StrategicFixedProfile([450, 400, 350, 300]), NG => FixedProfile(1e6)),
    )

    # Create the case dictionary
    case = Dict(
        :nodes => nodes,
        :links => links,
        :products => products,
        :T => T,
        :global_data => global_data,
    )

    wind = RP.NonDisRES(
        "wind",
        FixedProfile(2),
        FixedProfile(0.9),
        FixedProfile(10),
        FixedProfile(10),
        Dict(Power => 1),
        Dict(CO2 => 0.1, NG => 0),
        Dict("" => EMB.EmptyData()),
    )

    # Update nodes and links
    push!(case[:nodes], wind)
    link = EMB.Direct(41, case[:nodes][4], case[:nodes][1], EMB.Linear())
    push!(case[:links], link)

    model = EMB.OperationalModel()
    m = EMB.create_model(case, model)

    # Create model and optimize
    optimizer = optimizer_with_attributes(HiGHS.Optimizer, MOI.Silent() => true)
    set_optimizer(m, optimizer)
    optimize!(m)

    # Display some results
    pretty_table(
        JuMP.Containers.rowtable(
            value,
            m[:curtailment];
            header = [:Node, :TimePeriod, :Curtailment],
        ),
    )
end

@time demo_res()
```

<!---
## Documentation

The documentation is built with [Documenter.jl](https://juliadocs.github.io/Documenter.jl/stable/) can be generated by running
```shell
$ cd docs
$ julia make.jl
```
--->

## Project Funding

`EnergyModelsRenewableProducers` was funded by the Norwegian Research Council in the project [Clean Export](https://www.sintef.no/en/projects/2020/cleanexport/), project number [308811](https://prosjektbanken.forskningsradet.no/project/FORISS/308811)