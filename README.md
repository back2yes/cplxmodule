# CplxModule

A lightweight extension for `pytorch.nn` that adds layers and activations,
which respect algebraic operations over the field of complex numbers.

The implementation is based on the ICLR 2018 parer on Deep Complex Networks
[1]_ and borrows ideas from their [implementation](https://github.com/ChihebTrabelsi/deep_complex_networks).


# Installation

Just run to install with `pip`
```bash
pip install --upgrade git+https://github.com/ivannz/cplxmodule.git
```
or
```bash
python setup.py install
```
to install from the root of the repo.


# Example

Basically the module is designed in such a way as to be ready for plugging
into the existing `torch.nn` sequential models.

Importing the building blocks.
```python
import torch
import torch.nn

# converters
from cplxmodule import RealToCplx, CplxToReal

# layers of encapsulating other complex valued layers
from cplxmodule.sequential import CplxSequential

# common layers
from cplxmodule.layers import CplxConv1d

# activation layers
from cplxmodule.activation import CplxModReLU, CplxActivation

# special layers from signal processing
from cplxmodule.signal import CplxMultichannelGainLayer
```

After `RealToCplx` layer the intermediate inputs are pairs of tensors real and imaginary.
```python

n_features, n_channels = 16, 4
z = torch.randn(3, n_features*2)

re, im = RealToCplx()(z)
```

Stacking and constructing linear pipelines:
```python
n_features, n_channels = 16, 4
z = torch.randn(256, n_features*2)

# gain network works on the modulus of the complex input
modulus_gain = torch.nn.Sequential(
    torch.nn.Linear(n_features, n_channels * n_features),
    torch.nn.Sigmoid(),
)

# purely complex-to-complex sequential container
complex_model = CplxSequential(
    # complex: batch x n_features
    CplxMultichannelGainLayer(modulus_gain, flatten=False),

    # complex: batch x n_channels x n_features
    CplxConv1d(n_channels, 3 * n_channels, kernel_size=4, stride=1),

    # complex: batch x (3 * n_channels) x (n_features - (4-1))
    CplxModReLU(threshold=0.15),

    # complex: batch x (3 * n_channels) x (n_features - (4-1))
    CplxActivation(torch.flatten, start_dim=-2),
)

# branching into complex within a real-to-real model
real_input_model = torch.nn.Sequential(
    # real: batch x (n_features * 2)
    torch.nn.Linear(n_features * 2, n_features * 2),

    # real: batch x (n_features * 2)
    RealToCplx(),

    # complex: batch x n_features
    complex_model,

    # complex: batch x (3 * n_channels * (n_features - (4-1)))
    CplxToReal(),

    # real: batch x ((3 * n_channels * (n_features - (4-1))) * 2)
)

real_input_model(z).shape
# >>> torch.Size([256, 312])
```

# References

.. [1] Trabelsi, C., Bilaniuk, O., Zhang, Y., Serdyuk, D., Subramanian,
       S., Santos, J. F., ... & Pal, C. J. (2017). Deep complex networks.
       arXiv preprint arXiv:1705.09792