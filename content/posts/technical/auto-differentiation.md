---
title: "How Computers Do Differentiation?"
date: 2025-04-24T17:29:51+07:00
tags: ["AI", "math", "computer science"]
---

Differentiation is a key concept in machine learning, especially when optimizing functions like loss functions in neural networks. It helps us find the minimum of these functions, which is crucial for tasks like training a model. But have you ever wondered how popular libraries like **TensorFlow** and **PyTorch** perform differentiation? Let’s break it down!

## 1. Manual Differentiation: The Old-School Method

In school, we learn how to manually compute derivatives using calculus. You apply a set of rules to functions to find how they change with respect to their inputs. For example, given a simple function like:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;">
    <img src="https://latex.codecogs.com/svg.latex?\color{white}f(x,y) = x^{2}y + y + 2" title="f(x,y) = x^{2}\cdot{y} + y + 2" />
</div>

We could compute the partial derivatives with respect to x and y as follows:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;">
    <img src="https://latex.codecogs.com/svg.latex?\color{white}\frac{\partial f}{\partial x} = 2xy\indent\indent and\indent\indent\frac{\partial f}{\partial y} = 2xy + 1" title="\frac{\partial f}{\partial x} = 2\cdot{x}\cdot{y}" />
</div>

This method works well for simple functions, but as functions become more complex, the process of differentiation becomes tedious and prone to errors. **It’s not scalable for large models,** especially those used in machine learning.

## 2. Finite Difference Approximation: A Simpler, But Less Accurate Method

Finite difference approximation is a numerical method for calculating derivatives without explicit formulas. It approximates the derivative as the slope of a secant line between two points on the function. The derivative at point `x0` is defined as:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;">
    <img src="https://latex.codecogs.com/svg.latex?\color{white}h'(x_0)%20\approx%20\frac{h(x_0%20+%20\epsilon)%20-%20h(x_0)}{\epsilon}" title="h'(x_0) ≈ (h(x_0 + ε) - h(x_0)) / ε" />
</div>

Where `ε` is a small value (e.g., `10^(-5)`)

### Example with Python Code

```python
# f(x, y) = x^2 y + y + 2 with (x,y)=(3,4)
def f(x, y):
    return x**2 * y + y + 2

def derivative(f, x, y, x_eps, y_eps):
    return (f(x + x_eps, y + y_eps) - f(x, y)) / (x_eps + y_eps)

df_dx = derivative(f, 3, 4, 0.00001, 0)
df_dy = derivative(f, 3, 4, 0, 0.00001)

print(f"df_dx: {df_dx}") # 24.000039999805264
print(f"df_dy: {df_dy}") # 10.000000000331966
```

### Limitations

While simple, finite difference approximation is **imprecise** and becomes inefficient with many parameters. If there were `1000` parameters, we would need to call `f()` at least `1001` times. When you are dealing with large neural networks, this makes finite difference approximation way too inefficient.

## 3. Forward-Mode Autodiff

The figure below demonstrates forward-mode autodiff applied to the simple function `𝑔(𝑥,𝑦)=5+𝑥𝑦`. The left graph shows the function, while the right graph represents the partial derivative `∂𝑔/∂x=0+(0⋅𝑥+𝑦⋅1)=𝑦`. A similar process can be used to obtain the partial derivative with respect to `𝑦`

![plot](https://github.com/quachthetruong/quachthetruong.github.io/blob/main/assets/images/forward-mode-autodif.png?raw=true)

The forward-mode autodiff algorithm works by traversing the computation graph from inputs to outputs. It starts by computing the partial derivatives of the leaf nodes. For instance, the constant node `5` returns `0` (since the derivative of a constant is `0`), `𝑥` returns `1` (since `∂𝑥/∂𝑥=1`), and `𝑦` returns `0` (since `∂𝑦/∂𝑥=0`).

Next, we move to the multiplication node, where the product rule is applied: `∂(𝑢⋅𝑣)/∂𝑥=∂𝑣/∂𝑥⋅𝑢+𝑣⋅∂𝑢/∂𝑥`. This gives the expression `0⋅𝑥+𝑦⋅1`.

Finally, at the addition node, the derivative of a sum is the sum of the derivatives, so we combine the parts to get `∂𝑔/∂𝑥=0+(0⋅𝑥+𝑦⋅1)`.

Though the equation can be simplified further to `∂𝑔/∂𝑥=𝑦`, but imagine if our function had a variable `𝑧`— we would need to calculate the entire graph **once more**. In real-world scenarios, the function may have many more variables, making the differentiation process **much more complex and time-consuming**.

## 4. Reverse-Mode Autodiff

Reverse-mode autodiff is a powerful technique commonly used in machine learning. It involves two passes through the computation graph. The first pass computes the value of each node from inputs to output. The second pass works in reverse (from output to inputs) to compute all partial derivatives. This process is known as "reverse mode" because gradients flow backward through the graph.

The graph bellow illustrates the second pass. During the first pass, all node values are computed starting from `𝑥 = 3` and `𝑦 = 4`, with results shown at the bottom of each node (e.g., `𝑥 × 𝑥 = 9`). The output node, `𝑓(3,4)`, results in `42`.

![plot](https://github.com/quachthetruong/quachthetruong.github.io/blob/main/assets/images/reverse-mode-autodif.png?raw=true)

The reverse pass applies the chain rule to compute partial derivatives. **The chain rule** is given by:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> 
  <img src="https://latex.codecogs.com/svg.latex?\color{white}\frac{\partial%20f}{\partial%20x}%20=%20\frac{\partial%20f}{\partial%20n_i}%20\times%20\frac{\partial%20n_i}{\partial%20x}" 
  title="\frac{\partial f}{\partial x} = \frac{\partial f}{\partial n_i} \times \frac{\partial n_i}{\partial x}" />
</div>

where each `n𝑖` represents an intermediate node in the graph.

### Example of Calculating Partial Derivatives

Let’s go through the reverse pass step by step:

1. At `n7` (the output node):
 <div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> 
   <img src="https://latex.codecogs.com/svg.latex?\color{white}\frac{\partial%20f}{\partial%20n_7}%20=%201" 
   title="\frac{\partial f}{\partial n_7} = 1" />
 </div>

Since `n7` is the output node, `f=n7`, and the derivative with respect to `n7` is 1.

2. At `n5`:
 <div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> 
   <img src="https://latex.codecogs.com/svg.latex?\color{white}\frac{\partial%20f}{\partial%20n_5}%20=%20\frac{\partial%20f}{\partial%20n_7}%20\times%20\frac{\partial%20n_7}{\partial%20n_5}" 
   title="\frac{\partial f}{\partial n_5} = \frac{\partial f}{\partial n_7} \times \frac{\partial n_7}{\partial n_5} = 1 \times 1 = 1" />
 </div>

Since `n7 = n6 + n5`, we have:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> 
  <img src="https://latex.codecogs.com/svg.latex?\color{white}\frac{\partial%20n_7}{\partial%20n_5}=%201" 
  title="\frac{\partial f}{\partial n_5} = \frac{\partial f}{\partial n_7} \times \frac{\partial n_7}{\partial n_5} = 1 \times 1 = 1" />
</div>
Thus, the partial derivative is:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> 
  <img src="https://latex.codecogs.com/svg.latex?\color{white}\frac{\partial%20f}{\partial%20n_5}%20=%20\frac{\partial%20f}{\partial%20n_7}%20\times%20\frac{\partial%20n_7}{\partial%20n_5}%20=%201%20\times%201%20=%201" 
  title="\frac{\partial f}{\partial n_5} = \frac{\partial f}{\partial n_7} \times \frac{\partial n_7}{\partial n_5} = 1 \times 1 = 1" />
</div>

3. At `n4`:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> 
  <img src="https://latex.codecogs.com/svg.latex?\color{white}\frac{\partial%20f}{\partial%20n_4}%20=%20\frac{\partial%20f}{\partial%20n_5}%20\times%20\frac{\partial%20n_5}{\partial%20n_4}" 
  title="\frac{\partial f}{\partial n_5} = \frac{\partial f}{\partial n_7} \times \frac{\partial n_7}{\partial n_5} = 1 \times 1 = 1" />
</div>

Since `n5 = n4 * n2`, we have:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> 
  <img src="https://latex.codecogs.com/svg.latex?\color{white}\frac{\partial%20n_5}{\partial%20n_4}=%20 n_2" 
  title="\frac{\partial f}{\partial n_5} = \frac{\partial f}{\partial n_7} \times \frac{\partial n_7}{\partial n_5} = 1 \times 1 = 1" />
</div>

Thus, the partial derivative is:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> 
  <img src="https://latex.codecogs.com/svg.latex?\color{white}\frac{\partial%20f}{\partial%20n_4}%20=%201%20\times%20n_2%20=%204" 
  title="\frac{\partial f}{\partial n_5} = \frac{\partial f}{\partial n_7} \times \frac{\partial n_7}{\partial n_5} = 1 \times 1 = 1" />
</div>

This process continues for all the nodes in the graph. In the end, we will have computed all partial derivatives of `𝑓(𝑥,𝑦)` at `𝑥 = 3` and `𝑦 = 4`. In this example, we find:

<div style="display: inline-block; background-color: #6A6767; width: 25rem; height: 4rem;padding-left: 1rem; align-items: center; display:flex; border-radius: 8px;"> 
  <img src="https://latex.codecogs.com/svg.latex?\color{white}\frac{\partial%20f}{\partial%20x}%20=%2024 \indent\indent and \indent\indent \frac{\partial%20f}{\partial%20y}%20=%2010" 
  title="\frac{\partial f}{\partial x} = 24" />
</div>

### Key Benefits of Reverse-Mode Autodiff

Reverse-mode autodiff is particularly efficient when there are **many inputs and fewer outputs**, as it only requires one forward pass and one reverse pass per output. This makes it ideal for machine learning tasks, especially when training models like neural networks where there is typically only one output (e.g., the loss). This means **only two passes through the graph are needed to compute all the gradients with respect to the model parameters**.

Reverse-mode autodiff can also handle functions that are not entirely differentiable, as long as the partial derivatives can be computed at differentiable points.

## Conclusion

Having explored how computers perform differentiation, it’s clear why **reverse-mode autodiff is the dominant solution in machine learning**. The key advantage of reverse-mode is that it only requires two passes—one forward and one reverse—per output, regardless of the number of inputs. This makes it highly efficient, especially for large models where there are many parameters but typically only a single output, like the loss function in neural networks. Reverse-mode autodiff allows for fast, scalable optimization, making it the primary approach in modern machine learning frameworks.

### References

- https://github.com/ageron/handson-ml3
- https://www.youtube.com/watch?v=wG_nF1awSSY
- https://srijithr.gitlab.io/post/autodiff/
- https://www.tensorflow.org/guide/autodiff
