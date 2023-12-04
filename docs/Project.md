# Automatic Differentiation
Automatic Differentiation (AD) identifies a sequence of instructions performed by a given (primal) program 
$$
P : {I_1; I_2; \cdots; I_p}
$$

with a composition of mathematical functions 
$$
F = f_p \circ f_{p-1} \circ f_{p-2} \circ \cdots f_1.
$$

AD uses chain rule to evaluate the derivative of the output with respect to all it's inputs
$$
F'(X_0) = f'_p(X_{p-1}) \times f'_{p-1}(X_{p-2}) \times f'_{p-2}(X_{p-3}) \times \cdots f'_1(X_0).
$$
where $X_0$ is the input and $X_k=f_k(X_{k-1})$ for $k=1, 2, \cdots p$.

Practically, codes produced by AD will compute the product of $F'$ with an appropriate seed vector, either on the left (Reverse Mode AD) or on the right (Forward Mode AD). This approach is more efficient for applications that only require Jacobian-vector products. The full $𝐹'(𝑋)$ can be obtained through repeated evaluation of the derivative code with all vectors of the Cartesian basis as seeds.

In Forward Mode AD, with a seed column vector $\dot{X}$ multiplied on the right, we get direction derivative of $F$ along the direction $\dot{X}$. The efficient computation order in this case is from right to left. This results in the derivative computation alonside the function evaluation (in the same order), retaining the control structure of the primal program. 
$$
\dot{Y} = F'(X_0) \times \dot{X} = f'_p(X_{p-1}) \times f'_{p-1}(X_{p-2}) \times f'_{p-2}(X_{p-3}) \times \cdots f'_1(X_0) \times \dot{X}.
$$

With seed row vector $\bar{Y}$ multiplied to the left, we get Reverse Mode AD which computes the gradient with respect to $𝑋$ of the scalar function $𝑌 \times 𝐹(𝑋)$. The efficient computation order is left to right. This results in an adjoint program that evaluates, in the inverse order of the primal program. This is certainly more complicated but has an enormous benefit when a program has many inputs and few or just one output. This will return the complete gradient of the function with respect to all of its inputs in
just one run
$$
\bar{X} = \bar{Y} \times F'(X_0) = \bar{Y} \times f'_p(X_{p-1}) \times f'_{p-1}(X_{p-2}) \times f'_{p-2}(X_{p-3}) \times \cdots f'_1(X_0).
$$

## Parallel Forward Mode Automatic Differentiation
### Introduction
The core idea behind the implementation of Forward Mode Automatic Differentiation comes from **Complex Step Finite Difference** where the value and the change is stored separately, i.e. *Dual Numbers*. From Taylor Series we know that $f(x+h) \approx f(x)+hf^{'}(x)$. This statement gives us a way to represent a function by it's value and derivative:

$$
f(x)\rightarrow f(x)+\epsilon f^{'}(x) \tag{1}
$$

To enable this idea, I can define a custom `struct` which stores both value and the derivative. This `Dual` number is a multidimensional number where senstivity (or derivative) of the function is propagated alongside the evaluation.
```julia
struct Dual{T}
    # Value
    val::T
    # Derivative
    grad::T
end
```

Using Equation 1, let's define two function $f(x)$ and $g(x)$:

$$
f(x)\rightarrow f(x)+\epsilon f^{'}(x) \tag{2}
$$

$$
g(x)\rightarrow g(x)+\epsilon g^{'}(x) \tag{3}
$$

Now using the two above defined equations, I can perform several arithmetic operations on them. For example, Adding Equation 2 and 3 gives 

$$
(f+g)(x)\rightarrow \big(f(x)+g(x)\big)+\epsilon \big[f^{'}(x)+g^{'}(x)\big]
$$

This can be coded easily by overriding `+` operator such that it collects gradients as well as the value inside the `Dual` type (this is known as Multiple Dispatch in Julia).
```julia
# Addition of two Dual Numbers
function Base.:+(f::Dual{T}, g::Dual{T}) where {T}
    return Dual{T}(f.val+g.val, f.grad+g.grad)
end

# (Pre) Addition of Dual Number with a Scalar
function Base.:+(f::Number, g::Dual{T}) where {T}
    return Dual{T}(f+g.val, g.grad)
end

# (Post) Addition of Dual Number with a Scalar
function Base.:+(f::Dual{T}, g::Number) where {T}
    return Dual{N,T}(f.val*g, f.grad)
end
```

Something similar can be done for other operations as well. To keep the project feasible, I have defined Subtraction, Multiplication, Division and Power operations on the `Dual` number. Note that, more operations can be added very similarly and everything else will work as usual.
```julia
# Subtraction of two Dual Numbers
function Base.:-(f::Dual{T}, g::Dual{T}) where {T}
    return Dual{T}(f.val-g.val, f.grad-g.grad)
end

# (Pre) Subtraction of Dual Number with a Scalar
function Base.:-(f::Number, g::Dual{T}) where {T}
    return Dual{T}(f-g.val, g.grad)
end

# (Post) Subtraction of Dual Number with a Scalar
function Base.:-(f::Dual{T}, g::Number) where {T}
    return Dual{T}(f.val-g, f.grad)
end
```

```julia
# Multiplication of two Dual Numbers
function Base.:*(f::Dual{T}, g::Dual{T}) where {T}
    return Dual{T}(f.val*g.val, f.val*g.grad+f.grad*g.val)
end

# (Pre) Multiplication of Dual Number with a Scalar
function Base.:*(f::Number, g::Dual{T}) where {T}
    return Dual{T}(f*g.val, f*g.grad)
end

# (Post) Multiplication of Dual Number with a Scalar
function Base.:*(f::Dual{T}, g::Number) where {T}
    return Dual{T}(f.val*g, f.grad*g)
end
```

```julia
# Division of two Dual Numbers
function Base.:/(f::Dual{T}, g::Dual{T}) where {T}
    return Dual{T}(f.val/g.val, (f.grad*g.val-f.val*g.grad)/g.val^2)
end

# Division of a Scalar by Dual Number
function Base.:/(f::Number, g::Dual{T}) where {T}
    return Dual{T}(f/g.val, (-f*g.grad)/g.val^2)
end

# Division of a Dual Number by Scalar
function Base.:/(f::Dual{T}, g::Number) where {T}
    return Dual{T}(f.val/g, (f.grad*g)/g^2)
end
```

We know that $x^3$ basically means $((x\times x)\times x)$, so I can exploit the already defined **multiplication rule** to get this.
```julia
# Taking power to the dual number
function Base.:^(f::Dual{T}, g::Number) where {T}
    return Base.power_by_squaring(f, g)
end
```

In Forward Mode AD, we can get speedups from parallelisation in two ways:
- Computing the derivative in parallel alongside the function evaluation. This is universal and does not depend on the problem statement.
- Exploiting the parallelism in the computation graph. This is problem dependent. 

Let's look at each of them in further detail. 

### Parallel Derivative Computation
Suppose we have a function $f(x)=x^2+2x$. From chain rule, I can write $\frac{\partial f}{\partial x}=\frac{\partial f}{\partial x} \times \frac{\partial x}{\partial x}$ where $\frac{\partial x}{\partial x}=1$. Hence to get the *automatic differentiation* to supply $\frac{\partial f}{\partial x}$, I need to define the input $x$ as a `Dual` number, i.e. `Dual(x, 1)`, where `x` is the value at which we want to compute gradient.

As an example, let's evaluate gradient at $x=2$.
```julia
# Defining Function
h(x) = x^2 + 2*x + (x*5)/x

# Defining x
x = Dual(2,1)

# Evaluating h(x)
y = h(x)

println("Function Value: ", y.val)
println("Function Gradient: ", y.grad)
```
```
Function Value: 13
Function Gradient: 6
```

Spawning threads to parallelise the derivative computation results in massive overhead which in fact makes the computations slower. I can mitigate this problem by instead using *Vectorisation*. In the definition of the operations on `Dual` numbers, I made sure to define the `val` and `grad` fields independently. This will ensure that the compiler pushes these calculations through in parallel by utilizing the Vector Registers in the CPU. This is not a unique thing, and is very common in other languages like C/C++ or NumPy in Python.

I've verified this by evalauting the runtime of a simple function evaluation against the version using `Dual` number to evaluate the function as well as the gradient.
```julia
# Normal Function eval
println("Just Evaluating value of the Function")
@btime _ = h(2);

# Dual Function eval
println("Evaluating Value and Derivative of the function")
@btime _ = h(Dual(2,1));
```
```
Just Evaluating value of the Function
  0.875 ns (0 allocations: 0 bytes)
Evaluating Value and Derivative of the function
  0.875 ns (0 allocations: 0 bytes)
```

This is all well and good but if we have multiple functions and need to evaluate the jacobian. For example, we have functions $f(x, y) = x^2 + xy$ and $g(x, y) = y^3 + x$ and want to compute jacobian 
$J = 
\begin{bmatrix} 
\frac{\partial f}{\partial x} & \frac{\partial f}{\partial y}\\
\frac{\partial g}{\partial x} & \frac{\partial g}{\partial y}
\end{bmatrix}
$ at $x=3, y=4$.

With the current setup, this can be done by evaluating all four elements separately as follows:
```julia
# Defining Functions
f(x, y) = x^2 + x*y
g(x, y) = y^3 + x

# Getting Derivatives
df_dx = f(Dual(3, 1), Dual(4, 0)).grad
df_dy = f(Dual(3, 0), Dual(4, 1)).grad

dg_dx = g(Dual(3, 1), Dual(4, 0)).grad
dg_dy = g(Dual(3, 0), Dual(4, 1)).grad


J = zeros(2, 2)
J[1,1] = df_dx
J[1,2] = df_dy
J[2,1] = dg_dx
J[2,2] = dg_dy

println("Jacobian: ")
show(stdout, "text/plain", J)
```
```
Jacobian: 
2×2 Matrix{Float64}:
 10.0   3.0
  1.0  48.0
```

Hence, for $f(x_1, x_2, \cdots, x_n)$, I'll have to do differentiation `n` times (which isn't very efficient)!. I know that 
$$
\nabla f = \left[\begin{array}{ccc}
\dfrac{\partial f(x,y)}{\partial x} & \dfrac{\partial f(x,y)}{\partial y}
\end{array}\right]
$$ 
and if I want efficiency I need something like 

`df = ff(X).grads where (X=[x y])` 

and 

`df=[ff(Dual(3, 1), Dual(4, 0)).grad, ff(Dual(3, 0), Dual(4, 1)).grad]`, 

in turn utilise the **vector instructions** to calculate each partial derivative in parallel. This can be done by storing the derivatives in a `Vector` type rather than a scalar `grad`.

### `MultiDual` for efficient Jacobian 
As discussed above, to utilise vectorisation I can store gradients in a `SVector` type. This is a Static Vector which lives on the stack for fast access (this makes sense because the dimensions for most real life problems can't be horribly large). In `MultiDual`, `N` defines the number of variables in the function and `T` defines the type (`Int`, `Float`, etc. etc.).

```julia
struct MultiDual{N,T}
	# Value
	val::T
	# SVector is static vector which lives on the stack
	grads::SVector{N,T} 
end
```

Various rules can be defined using Taylor Series (like before), i.e. if $x$ is a vector then Taylor Series is defined as 

$$f(x+\epsilon)=f(x)+\epsilon \nabla f(x)+O(\epsilon)$$

Only change is that $f^{'}$ is replaced by $\nabla f$ and the same thing goes for various differentiation rules.

- **Addition Rule**
    $$(f+g)(x)\rightarrow f(x)+g(x)+\epsilon \big[\nabla f(x)+\nabla g(x)\big]$$
```julia
function Base.:+(f::MultiDual{N,T}, g::MultiDual{N,T}) where {N,T}
    return MultiDual{N,T}(f.val+g.val, f.grads+g.grads)
end

function Base.:+(f::Number, g::MultiDual{N,T}) where {N,T}
    return MultiDual{N,T}(f+g.val, g.grads)
end

function Base.:+(f::MultiDual{N,T}, g::Number) where {N,T}
    return MultiDual{N,T}(f.val+g, f.grads)
end
```

- **Subtraction Rule**
    $$(f-g)(x)\rightarrow f(x)-g(x)+\epsilon \big[\nabla f(x)-\nabla g(x)\big]$$

```julia
function Base.:-(f::MultiDual{N,T}, g::MultiDual{N,T}) where {N,T}
    return MultiDual{N,T}(f.val-g.val, f.grads-g.grads)
end

function Base.:-(f::Number, g::MultiDual{N,T}) where {N,T}
    return MultiDual{N,T}(f-g.val, g.grads)
end

function Base.:-(f::MultiDual{N,T}, g::Number) where {N,T}
    return MultiDual{N,T}(f.val-g, f.grads)
end
```

- **Multiplication Rule**
    $$(f \cdot g)(x)\rightarrow f(x)g(x)+\epsilon \big[f(x)\nabla g(x)+\nabla f(x)g(x)\big]$$
```julia
function Base.:*(f::MultiDual{N,T}, g::MultiDual{N,T}) where {N,T}
    return MultiDual{N,T}(f.val*g.val, f.val*g.grads+f.grads*g.val)
end

function Base.:*(f::Number, g::MultiDual{N,T}) where {N,T}
    return MultiDual{N,T}(f*g.val, f*g.grads)
end

function Base.:*(f::MultiDual{N,T}, g::Number) where {N,T}
    return MultiDual{N,T}(f.val*g, f.grads*g)
end
```

- **Division Rule**
    $$(\frac{f}{g})(x)\rightarrow \frac{f(x)}{g(x)}+\epsilon \bigg[\frac{\nabla f(x)g(x)-f(x)\nabla g(x)}{g(x)^2}\bigg]$$
```julia
function Base.:/(f::MultiDual{N,T}, g::MultiDual{N,T}) where {N,T}
    return MultiDual{N,T}(f.val/g.val, (f.grads*g.val-f.val*g.grads)/g.val^2)
end

function Base.:/(f::Number, g::MultiDual{N,T}) where {N,T}
    return MultiDual{N,T}(f/g.val, (-f.val*g.grads)/g.val^2)
end

function Base.:/(f::MultiDual{N,T}, g::Number) where {N,T}
    return MultiDual{N,T}(f.val/g, (f.grads*g.val)/g^2)
end
```

- **Exponential operator**
```julia
function Base.:^(f::MultiDual{N,T}, g::Number) where {N,T}
    return Base.power_by_squaring(f,g)
end
```

Let's now compute the Jacobian using `MultiDual`. The two functions are 

$$f(x, y) = x^2 + xy$$ 

$$g(x, y) = y^3 + x$$

The next step is to define `MultiDual` functions for `x` and `y`. 
- Let a function $xx(x,y)=x + 0 \times y$. `MultiDual` `xx` can be written as `xx = MultiDual(x, SVector(1.,0.))`. `SVector(1.,0.)` is written because $\frac{\partial xx}{x} = 1$ and $\frac{\partial xx}{y} = 0$, hence these are set as the first and second elements of the vector.
- Similarly for $yy(x,y)=y + 0 \times x$, `yy=MultiDual(y, SVector(0.,1.))`.

Jacobian can then easily be calculated in parallel.
```julia
f(x, y) = x^2 + x*y
g(x, y) = y^3 + x

# Jacobian at x=3 and y=4
xx = MultiDual(3., SVector(1.,0.))
yy = MultiDual(4., SVector(0.,1.))

# Function and Jacobian computation
f_ = f(xx, yy)
g_ = g(xx, yy)

df_dx, df_dy = f_.grads 
dg_dx, dg_dy = g_.grads 

J = zeros(2, 2)
J[1,1] = df_dx
J[1,2] = df_dy
J[2,1] = dg_dx
J[2,2] = dg_dy

println("Jacobian: ")
show(stdout, "text/plain", J)
```
```
Jacobian: 
2×2 Matrix{Float64}:
 10.0   3.0
  1.0  48.0
```

### Parallel Stencil Computation