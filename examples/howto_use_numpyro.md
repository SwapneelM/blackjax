---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.14.0
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

# Use with Numpyro models

Blackjax accepts any log-probability function as long as it is compatible with JAX's primitive. In this notebook we show how we can use Numpyro as a modeling language together with Blackjax as an inference library.


``` {admonition} Before you start
You will need [Numpyro](https://github.com/pyro-ppl/numpyro) to run this example. Please follow the installation instructions on Numpyro's repository.
```

We reproduce the Eight Schools example from the [Numpyro documentation](https://github.com/pyro-ppl/numpyro) (all credit for the model goes to the Numpyro team).

```{code-cell} ipython3
:tags: [hide-cell]
import numpy as np


J = 8
y = np.array([28.0, 8.0, -3.0, 7.0, -1.0, 1.0, 18.0, 12.0])
sigma = np.array([15.0, 10.0, 16.0, 11.0, 9.0, 11.0, 10.0, 18.0])
```

We implement the non-centered version of the hierarchical model:

```{code-cell} ipython3
import numpyro
import numpyro.distributions as dist
from numpyro.infer.reparam import TransformReparam


def eight_schools_noncentered(J, sigma, y=None):
    mu = numpyro.sample("mu", dist.Normal(0, 5))
    tau = numpyro.sample("tau", dist.HalfCauchy(5))
    with numpyro.plate("J", J):
        with numpyro.handlers.reparam(config={"theta": TransformReparam()}):
            theta = numpyro.sample(
                "theta",
                dist.TransformedDistribution(
                    dist.Normal(0.0, 1.0), dist.transforms.AffineTransform(mu, tau)
                ),
            )
        numpyro.sample("obs", dist.Normal(theta, sigma), obs=y)
```

```{warning}
The model applies a transformation to the `theta` variable. As a result, the samples generated by Blackjax will be samples in the *transformed space* and you will have to transform them back to the original space with Numpyro.
```

We need to translate the model into a log-probability function that will be used by Blackjax to perform inference. For that we use the `initialize_model` function in Numpyro's internals. We will also use the initial position it returns to initialize the inference:

```{code-cell} ipython3
import jax

from numpyro.infer.util import initialize_model

rng_key = jax.random.PRNGKey(0)
init_params, potential_fn_gen, *_ = initialize_model(
    rng_key,
    eight_schools_noncentered,
    model_args=(J, sigma, y),
    dynamic_args=True,
)
```

Numpyro return a potential function, which is easily transformed back into a logprob function that is required by Blackjax:


```{code-cell} ipython3
logprob_fn = lambda position: -potential_fn_gen(J, sigma, y)(position)
initial_position = init_params.z
```

We can now run the window adaptation for the NUTS sampler:

```{code-cell} ipython3
import blackjax

num_warmup = 2000

adapt = blackjax.window_adaptation(
    blackjax.nuts, logprob_fn, num_warmup, target_acceptance_rate=0.8
)
last_state, kernel, _ = adapt.run(rng_key, initial_position)
```

Let us now perform inference with the tuned kernel:

```{code-cell} ipython3
:tags: [hide-cell]

def inference_loop(rng_key, kernel, initial_state, num_samples):
    @jax.jit
    def one_step(state, rng_key):
        state, info = kernel(rng_key, state)
        return state, (state, info)

    keys = jax.random.split(rng_key, num_samples)
    _, (states, infos) = jax.lax.scan(one_step, initial_state, keys)

    return states, (
        infos.acceptance_probability,
        infos.is_divergent,
        infos.num_integration_steps,
    )
```

```{code-cell} ipython3
num_sample = 1000

states, infos = inference_loop(rng_key, kernel, last_state, num_sample)
_ = states.position["mu"].block_until_ready()
```

To make sure that the model sampled correctly, let's compute the average acceptance rate and the number of divergences:

```{code-cell} ipython3
:tags: [hide-cell]
acceptance_rate = np.mean(infos[0])
num_divergent = np.mean(infos[1])

print(f"\Average acceptance rate: {acceptance_rate:.2f}")
print(f"There were {100*num_divergent:.2f}% divergent transitions")
```

Finally let us now plot the distribution of the parameters. Note that since we use a transformed variable, Numpyro does not output the school treatment effect directly:

```{code-cell} ipython3
:tags: [hide-cell]
import seaborn as sns
from matplotlib import pyplot as plt

samples = states.position

fig, axes = plt.subplots(ncols=2)
fig.set_size_inches(12, 5)
sns.kdeplot(samples["mu"], ax=axes[0])
sns.kdeplot(samples["tau"], ax=axes[1])
axes[0].set_xlabel("mu")
axes[1].set_xlabel("tau")
fig.tight_layout()
```

```{code-cell} ipython3
:tags: [hide-cell]
fig, axes = plt.subplots(8, 2, sharex="col", sharey="col")
fig.set_size_inches(12, 10)
for i in range(J):
    axes[i][0].plot(samples["theta_base"][:, i])
    axes[i][0].title.set_text(f"School {i} relative treatment effect chain")
    sns.kdeplot(samples["theta_base"][:, i], ax=axes[i][1], shade=True)
    axes[i][1].title.set_text(f"School {i} relative treatment effect distribution")
axes[J - 1][0].set_xlabel("Iteration")
axes[J - 1][1].set_xlabel("School effect")
fig.tight_layout()
plt.show()
```

```{code-cell} ipython3
:tags: [hide-cell]
for i in range(J):
    print(
        f"Relative treatment effect for school {i}: {np.mean(samples['theta_base'][:, i]):.2f}"
    )
```
