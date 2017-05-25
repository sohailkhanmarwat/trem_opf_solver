# Optimal Power Flow solver for Restricted Radial Networks

This repository contains MATLAB code for the optimal power flow problem on a special kind of networks, we call restricted radial networks, which have the following properties:

- The topology is a tree with non-zero edge impedances.
- Node 1 is the reference node.
- The leaves of the tree are  either PQ or PV constrained nodes with bounded intervals constraining voltage magnitude or reactive power.
- Internal nodes are PQ nodes, with arbitrary, possibly unbounded, intervals constraining voltage magnitude.

The precise definition is described in the paper [INSERT PAPER REFERENCE HERE].

# Getting Started

1. Make sure you have MATLAB2015b. Verify that the curve-fitting toolbox is installed by typing the following command inside MATLAB:

   ```matlab
   help curvefit
   ```

2. [Download](https://github.com/alexshtf/trm_opf_solver/archive/master.zip) the contents of this repository. Unzip the file to a directory of your choice.

3. Inside MATLAB, go to the directory with the unzipped contents.

4. Solve an OPF problem on one of the provided networks:

   ```matlab
   load('networks/13b.mat'); % load network data into the Z, PQ, PV, ref variables
   f = @(v, s) real(s(1, :)); % objective - real power generated by the reference node
   [v, s, optval] = solve_tree(f, Z, PQ, PV, ref)
   ```

Detailed reference documentation about using the solver is available by typing
```matlab
help solve_tree
```
# Running the solver on your own network

The following short MATLAB code defines a power network, feeds it to our solver, and produces an optimal solution. The code below can be found in the `example.m` script. 

```matlab
% Define the impedance matrix and network topology
Zdef = [1, 2, 0.0017+0.0003i;  % Z12 
        1, 3, 0.0006+0.0001i;  % Z13
        3, 4, 0.0007+0.0003i;  % Z34
        3, 5, 0.0009+0.0005i]; % Z35
Z = sparse(Zdef(:, 1), Zdef(:, 2), Zdef(:, 3), 5, 5);
Z = Z + Z.';

% Define PQ constraints. Each row is:
%   bus,    p+iq, vmin, vmax       [consumed power is negative!]
PQ = [3,   -5-1i,  0.9, 1.1  ;
      4, -4-0.2i, 0.92, 1.06];

% Define PV constraints. Each row is:
%   bus,   p,  |v|, qmin, qmax     [generated power is positive]
PV = [2, 4.9, 1.01,  -10, 10;
      5, 4.2,  1.0,   -5, 5];
      
% Define reference constraints: 
%      vmin, vmax, pmin, pmax, qmin, qmax      
ref = [0.93,  1.1,    0,    5,  -10, 10];

% Define objective function - real power at node 1 + stability at PQ nodes.
%    f(v, s) = Re(s_1) + ||v_3| - 1| + ||v_4| - 1|
% We must provide a function handle that applies the function above
% to every column of the given arguments.
f = @(v, s) real(s(1, :)) + abs(abs(v(3, :)) - 1) + abs(abs(v(4, :)) - 1);

% Solve the OPF problem and produce the optimal voltages, powers and objective value.
[v, s, fval] = solve_tree(f, Z, PQ, PV, ref)
```

Running the code above produces the following output:

```
v =

   1.0013 + 0.0000i
   1.0100 - 0.0012i
   0.9979 + 0.0023i
   0.9950 + 0.0012i
   1.0000 + 0.0074i


s =

   0.0083 + 3.0689i
   4.9000 + 1.5423i
  -5.0000 - 1.0000i
  -4.0000 - 0.2000i
   4.2000 - 3.3796i


fval =

    0.0153
```

# More on objective functions

Looking at the example above, we defined the objective function to operate on all columns at once. Namely, we wrote

```matlab
f = @(v, s) real(s(1, :)) + abs(abs(v(3, :)) - 1) + abs(abs(v(4, :)) - 1);
```

instead of using the, perhaps, more natural syntax:

```matlab
f = @(v, s) real(s(1)) + abs(abs(v(3)) - 1) + abs(abs(v(4)) - 1);
```

The reason is that the objective given to `solve_tree` must be able to evaluate the function value on a set of feasible solutions at once, instead of operating on a single feasible solution. This approach was chosen to let us, the users, provide an efficient 'vectorized' implementation, with a small sacrifice to code readability.

The function is given two *matrices* `v, s` of size `m x n`, where `m` is the number of nodes in the given network. Namely, for each `i`, the pair of vectors `v(:, i)` and `s(:, i)` is a feasible solution of the OPF problem. The function is expected to return a vector of `n` entries containing the value of the objective function for every feasible solution.

Here are more examples of objective functions:

```matlab
% The voltage stability function at all nodes - the sum of deviations of |v| from 1.
f = @(v, s) sum(abs(abs(v) - 1)); 

% The stability function, considering PQ nodes only. 
pq_indices = PQ(:, 1); % indices are stored in the first column. See example above
g = @(v, s) sum(abs(abs(v(pq_indices, :) - 1)));
```

## Specifying additional constraints

The objective function can be used to specify additional constraints, which are not covered by the `PQ`, `PV`, and `ref` matrices given to `solve_tree`, by returning`inf` when the constraint is violated.  For example, we could have the following constraint on the reference node:

```
p_1 + abs(q_1) <= 10
```

where `p_1` is the active power and `q_1` is the reactive power. It reflects the fact that the limit on the reactive power could depend on the amount of real power we generate.

To minimize, for example, the stability function, subject to the constraints of the network *and* the constraint above, we can write:

```matlab
stability = @(v) sum(abs(abs(v) - 1)); 
constraint = @(s) real(s(1, :)) + abs(imag(q(1, :))) <= 10; 
false_to_inf = @(z) 1./z - 1;
f = @(v, s) stability(v) + false_to_inf(constraint(s));

[v, s, opt] = solve_tree(f, Z, PQ, PV, ref); % Z, PQ, PV, ref already defined
```
