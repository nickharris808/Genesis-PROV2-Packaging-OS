# Solver Architecture Overview (Non-Confidential)

This document describes the mathematical framework underlying the Genesis PROV 2 Packaging OS. It covers the governing equations, solution methods, and optimization architecture without disclosing implementation details or source code.

---

## 1. Kirchhoff-von Karman Plate Theory

### Governing Equations

The warpage of thin substrates under thermal and mechanical loading is governed by the Kirchhoff-von Karman nonlinear plate equations. For a plate of thickness h with flexural rigidity D, the coupled system is:

**Equilibrium equation (out-of-plane)**:
```
D * nabla^4(w) = q + h * [F_yy * w_xx - 2 * F_xy * w_xy + F_xx * w_yy]
```

**Compatibility equation (in-plane)**:
```
nabla^4(F) = E * [w_xy^2 - w_xx * w_yy]
```

where:
- w(x, y) is the out-of-plane deflection
- F(x, y) is the Airy stress function
- q(x, y) is the transverse load (including thermal contributions)
- D = E * h^3 / [12 * (1 - nu^2)] is the flexural rigidity
- E is Young's modulus, nu is Poisson's ratio

The nonlinear coupling between the two equations (through the w_xx, w_yy, w_xy terms) captures the geometric stiffening effect that becomes significant when deflections exceed approximately 0.3 * h.

### Linear Kirchhoff (Small Deflection)

For small deflections (w << h), the compatibility equation decouples and the equilibrium reduces to the linear biharmonic equation:

```
D * nabla^4(w) = q
```

**Circular plate (simply supported, uniform load)**:
```
w(r) = q / (64 * D) * (R^2 - r^2)^2 / (1 + nu) * [5 + nu) / (1 + nu) * R^2 - r^2]
```

Maximum deflection at center:
```
w_max = q * R^4 / (64 * D)
```

**Rectangular plate (simply supported, uniform load -- Navier solution)**:
```
w(x, y) = sum_m sum_n A_mn * sin(m * pi * x / a) * sin(n * pi * y / b)
```

where:
```
A_mn = 16 * q / [pi^6 * D * m * n * (m^2/a^2 + n^2/b^2)^2]
```

for odd m, n only. The key observation is that this solution is expressed entirely in Cartesian coordinates (x, y) with no angular variable theta. This is why azimuthal stiffness modulation, which introduces cos(n * theta) terms, has no effect on rectangular plate response.

### NLGEOM Implementation

The NLGEOM (nonlinear geometry) implementation solves the full coupled Kirchhoff-von Karman system using an iterative Newton-Raphson scheme. This is executed in the CalculiX open-source FEM solver with shell elements and thermal loading. The cloud FEM cases (~550 task IDs) use this formulation.

---

## 2. Classical Laminate Plate Theory (CLPT)

### Multi-Layer Substrate Modeling

Advanced packaging substrates are multi-layer composite structures: die (silicon), adhesive (polymer), interposer (silicon or glass), and substrate (organic or glass). CLPT provides the analytical framework for computing the effective mechanical response.

### ABD Stiffness Matrix

For a laminate of N layers, the constitutive relation is:

```
[N]     [A  B] [epsilon_0]
[M]  =  [B  D] [kappa    ]
```

where:
- N = force resultants (N_x, N_y, N_xy)
- M = moment resultants (M_x, M_y, M_xy)
- epsilon_0 = midplane strains
- kappa = midplane curvatures
- A = extensional stiffness matrix
- B = coupling stiffness matrix
- D = bending stiffness matrix

The matrices are computed by integrating through the thickness:

```
(A_ij, B_ij, D_ij) = integral from -h/2 to h/2 of Q_ij * (1, z, z^2) dz
```

where Q_ij is the transformed reduced stiffness of each layer.

### Thermal Moment Resultant

Under thermal loading with temperature change Delta_T, the thermal moment is:

```
M_T = sum over k layers of [ Q_bar_k * alpha_k * Delta_T * (z_k^2 - z_{k-1}^2) / 2 ]
```

where alpha_k is the CTE of layer k and z_k are the layer boundaries. The warpage (curvature) is then:

```
kappa = D^{-1} * (M_T - B * A^{-1} * N_T)
```

This provides an exact analytical solution for each combination of layer thicknesses, materials, and thermal loading -- enabling rapid generation of the 3,508 training samples for the AI compiler.

---

## 3. Cartesian Stiffness Formula

### The Core Result

The physics-correct stiffness distribution for rectangular substrates is:

```
K(x, y) proportional to |nabla^2 M_T(x, y)|
```

where nabla^2 M_T is the Laplacian of the thermal moment field. This maps support stiffness to the spatial curvature of the thermal loading, providing maximum warpage compensation where the thermal moment changes most rapidly.

### Physical Motivation

The thermal moment M_T(x, y) represents the through-thickness asymmetry of thermal stress. Where M_T is spatially uniform, the plate curves uniformly and a uniform support suffices. Where M_T has strong spatial gradients (near die edges, material boundaries, thermal vias), the plate develops localized curvature that requires spatially-varying stiffness compensation.

The Laplacian |nabla^2 M_T| identifies these high-curvature regions, directing support stiffness precisely where it is needed.

### Why Azimuthal Fails

The azimuthal formulation K(r, theta) = K_0 * [1 + k_azi * cos(n * theta)] couples to the hoop stress sigma_theta in circular plates. For rectangular plates, the stress decomposition into (sigma_xx, sigma_yy, tau_xy) has no theta-periodic component. The inner product:

```
integral over domain of K_azi(r, theta) * stress_field_rect(x, y) dA = 0
```

is identically zero because the basis functions are orthogonal. This is the mathematical content of the Rectangular Immunity Theorem.

---

## 4. Bayesian Optimization

### Optimization Framework

The design optimization uses Bayesian optimization with Gaussian Process (GP) surrogate and Expected Improvement (EI) acquisition function:

```
EI(x) = (mu(x) - f_best) * Phi(z) + sigma(x) * phi(z)
```

where:
- mu(x), sigma(x) are the GP posterior mean and standard deviation
- f_best is the best warpage found so far
- z = (mu(x) - f_best) / sigma(x)
- Phi, phi are the standard normal CDF and PDF

### Optimization Loop

1. Initialize with Latin Hypercube Sample (LHS) of the design space
2. Evaluate each point using CLPT (fast) or FEM (expensive, high-fidelity)
3. Fit GP surrogate to all evaluated points
4. Maximize EI to select next evaluation point
5. Repeat until convergence or budget exhaustion

The best warpage of 4.53 micrometers was achieved in 14 FEM evaluations, demonstrating the efficiency of this approach for expensive-to-evaluate design spaces.

---

## 5. AI Compiler (Reduced-Order Model)

### Architecture

The ROM uses GradientBoosting regression (scikit-learn implementation) with the following pipeline:

1. **Feature space**: 8 design parameters (layer thicknesses, material indices, thermal loading)
2. **Training data**: 3,508 CLPT analytical evaluations via smart Latin Hypercube sweep
3. **Model**: GradientBoostingRegressor with hyperparameter tuning
4. **Validation**: 5-fold cross-validation, R-squared = 0.9982 +/- 0.0001
5. **Test accuracy**: R-squared = 0.9977 on held-out test set

### Use in Inverse Design

The ROM enables millisecond-scale warpage prediction, which is used for:

- **Design-space exploration**: Evaluate millions of candidate designs in seconds
- **Sensitivity analysis**: Sobol indices to identify dominant parameters
- **Multi-objective Pareto search**: Trade off warpage, cost, and manufacturability
- **APR integration**: Real-time warpage check during automatic place-and-route

### Domain Limitations

The ROM is trained on CLPT (linear theory) data. It does not capture:
- Geometric nonlinearity (large deflections captured by NLGEOM FEM)
- Contact mechanics (die-substrate interaction)
- Time-dependent effects (creep, viscoelasticity)

Final designs should be validated with full NLGEOM FEM simulation.

---

## 6. Verification Methods

### Method Confidence Levels

| Method | Physics Fidelity | Computational Cost | Confidence |
|:-------|:----------------|:-------------------|:-----------|
| NLGEOM FEM (CalculiX) | Full nonlinear | Hours (cloud) | High |
| Von Karman nonlinear | Geometric nonlinearity | Minutes (local) | High |
| CLPT analytical | Linear, exact composite | Milliseconds | Medium |
| ROM (GradientBoosting) | Learned from CLPT | Sub-millisecond | Medium (within domain) |

### Validation Hierarchy

1. **Analytical verification**: NLGEOM FEM matches Kirchhoff closed-form for simple cases
2. **Mesh convergence**: FEM results independent of mesh density
3. **Cross-solver**: Von Karman local solver confirms NLGEOM cloud results
4. **Statistical**: Monte Carlo over 100 cases confirms stability

---

*Genesis Platform -- PROV 2 Solver Architecture Overview*
*February 2026*
