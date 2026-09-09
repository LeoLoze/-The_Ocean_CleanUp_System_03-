# Aggregation methods

Here is an explanation of how the group preference is aggregated. This group preference is the objective that is optimised by the GA. Three aggregation methods are available:

- Weighted min-max
- Weighted least squares
- Tetra

### Min-max

For the min-max aggregation, the GA minimises the worst weighted deviation from the goal values across all criteria:

$$
f(x) = \max_{i=1,\dots,n} \left[ w_i \cdot (g_i - p_i(x)) \right]
$$

where:

- $x$ is a candidate solution.
- $n$ is the number of criteria.
- $w_i$ is the weight of criterion $i$, with $\sum_i w_i = 1$.
- $g_i$ is the goal value for criterion $i$.
- $p_i(x)$ is the performance value of solution $x$ on criterion $i$.

The GA tries to find the solution that minimises this maximum value. In other words, it does not optimise the average performance (for us, this is equivalent to the preference score), but rather minimises the loss of the worst-performing (weighted) criterion. Goal values are defined at 100.

### Weighted least squares (A-fine)

The A-fine aggregation algorithm uses weighted least-squares aggregation. The preference scores are first normalised using a z-score:

$$
z_{i,j} = \frac{p_{i,j} - \mu_j}{\sigma_j}
$$

The aggregated preference score for alternative $i$ is then the weighted sum of these normalised scores:

$$
P_i^* = \sum_{j=1}^{J} w_j z_{i,j}
$$

So, instead of taking the worst-performing criterion, this method produces a single representative score that reflects the weighted best fit across all criteria. A more detailed explanation is given in `a-fine-aggregator-description.pdf`, located in the `genetic_algorithm_pfm` folder.

**Note:** this aggregation method scores the best solution in each generation as `-100`, making the score relative rather than absolute. As a result, scores cannot be directly compared across generations to determine whether the population is actually improving, even if it is. This can interfere with the early-stopping mechanism based on `max_stall`, sometimes triggering a "too fast convergence" warning. However, as long as the other monitored parameters (mean and diversity) remain stable across the last generations, this warning does not necessarily indicate a problem and can be disregarded.

### Tetra

Tetra is a more advanced aggregation method developed by [Tetra](https://www.scientificmetrics.com). For our purposes it functions as a black box: the model is sent to a remote Tetra server, where the aggregation is computed. Because of this remote computation, Tetra is also considerably slower than the other two methods. It is therefore not required for the assignment, but is made available for students who wish to explore additional tools.