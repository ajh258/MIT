## Software & Packages
This code was ran in Jupyter using Julia v.1.7.2

Required Julia packages: GFlops, LeastSquaresOptim, Printf, Test

To install Julia packages (e.g. GFlops) run the following code first:
```julia
julia> using Pkg
julia> Pkg.add("GFlops")
```


## File Description
finalcode - this is the optimization code 

.csv - this is the sparse matrix used for testing 



## GFlops Counter Note

GFlops.jl does not see what happens outside the realm of Julia code. 

It does not see operations performed in external libraries such as BLAS calls: 

```julia
julia> using GFlops

julia> using LinearAlgebra

julia> x = rand(1000)
1000-element Vector{Float64}:
 0.5776651858964748
 0.6946644664794772
 0.3875659617393591
 0.3607964326251861
 0.878597803663574
 0.49578477434890644
 0.34572482864736054
 0.47060317011038777
 0.3907418152947162
 0.49669411346439796
 ⋮
 0.6868543025983905
 0.8609730536488172
 0.25835857412032026
 0.633615787488576
 0.43785626016340684
 0.3394714234751539
 0.9715263132599209
 0.01046989353197414
 0.5717838360554347

julia> @count_ops dot($x, $x)
Flop Counter: 0 flop
```




## Simple Syntax
The symple syntax mirrors the `Optim.jl` syntax

```julia
using LeastSquaresOptim
function rosenbrock(x)
	[1 - x[1], 100 * (x[2]-x[1]^2)]
end
x0 = zeros(2)
optimize(rosenbrock, x0, Dogleg())
optimize(rosenbrock, x0, LevenbergMarquardt())
```
You can also add the options : `x_tol`, `f_tol`, `g_tol`, `iterations`, `Δ` (initial radius), `autodiff` (`:central` to use finite difference method or `:forward` to use ForwardDiff package) as well as `lower` / `upper` arguments to impose boundary constraints.


## Choice of Optimizer / Least Square Solver
- You can specify two least squares optimizers, `Dogleg()` and `LevenbergMarquardt()`
- You can specify three least squares solvers (used within the optimizer)
	- `LeastSquaresOptim.QR()` or  `LeastSquaresOptim.Cholesky()` for dense jacobians
	- `LeastSquaresOptim.LSMR()`. A conjugate gradient method ([LSMR]([http://web.stanford.edu/group/SOL/software/lsmr/) with diagonal preconditioner). Beyond `Matrix` and `SparseMatrixCSC`, the jacobian can be any type that defines the following interface:

		- `mul!(y, A, x, α::Number, β::Number)` updates y to αAx + βy
		- `mul!(x, A', y, α::Number, β::Number)` updates x to αA'y + βx
		- `colsumabs2!(x, A)` updates x to the sum of squared elements of each column
		- `size(A, d)` returns the nominal dimensions along the dth axis in the equivalent matrix representation of A.
		- `eltype(A)` returns the element type implicit in the equivalent matrix representation of A.

		Similarly, `x` or `f(x)` may be custom types. An example of the interface can be found in the package [SparseFactorModels.jl](https://github.com/matthieugomez/SparseFactorModels.jl).

		Moreover, you can optionally specifying a function `preconditioner!` and a matrix `P` such that `preconditioner!(P, x, J, λ)` updates `P` as a preconditioner for `J'J + λ`. The preconditioner can be any type that supports `A_ldiv_B!(x, P, y)`. By default, the preconditioner is chosen as the diagonal of the matrix `J'J + λ`. 


The optimizers and solvers are presented in more depth in the [Ceres documentation](http://ceres-solver.org/nnls_solving.html). For dense jacobians, the default option is `Doglel(QR())`. For sparse jacobians, the default option is  `LevenbergMarquardt(LSMR())`

```julia
optimize(rosenbrock, x0, Dogleg(LeastSquaresOptim.QR()))
optimize(rosenbrock, x0, LevenbergMarquardt(LeastSquaresOptim.LSMR()))
```


## Alternative in-place Syntax
The alternative syntax is useful when specifying in-place functions or the jacobian. Pass `optimize` a `LeastSquaresProblem` object with:
 - `x` an initial set of parameters.
 - `f!(out, x)` that writes `f(x)` in `out`.
 - the option `output_length` to specify the length of the output vector. 
 - Optionally, `g!` a function such that `g!(out, x)` writes the jacobian at x in `out`. Otherwise, the jacobian will be computed following the `:autodiff` argument.

```julia
using LeastSquaresOptim
function rosenbrock_f!(out, x)
 out[1] = 1 - x[1]
 out[2] = 100 * (x[2]-x[1]^2)
end
optimize!(LeastSquaresProblem(x = zeros(2), f! = rosenbrock_f!, output_length = 2, autodiff = :central), Dogleg())

# if you want to use gradient
function rosenbrock_g!(J, x)
    J[1, 1] = -1
    J[1, 2] = 0
    J[2, 1] = -200 * x[1]
    J[2, 2] = 100
end
optimize!(LeastSquaresProblem(x = zeros(2), f! = rosenbrock_f!, g! = rosenbrock_g!, output_length = 2), Dogleg())
```
