`orenstein-analysis` is a functional python package for loading, processing, and manipulating hierarchical and multidimensional datasets in Joe Orenstein's optics laboratory at UC Berkeley. The best way to think about this packages is as a set of general tools to streamline scripting of new data pipelines and recycle code common to all analysis - most of the heavy lifting comes from xarray and the user can focus on writing only code that is specific to their experiment. See below for a quick example.

The package builds on top of the xarray DataArray and Dataset data structures - for a good introduction to xarray see the [xarray Getting Started and User Guide](https://docs.xarray.dev/en/stable/getting-started-guide/index.html).

Briefly, xarray DataArrays are labeled, indexed wrappers around numpy ndarrays. In addition, there is a high-level distinction between data variables (ndarrays of dependent variables, for example temperature as function of x, y, and z), coordinate variables (ndarrays for independent variables, such as the x, y and z vectors), and dimensions (labels for each axis associated with a data or coordinate variable, such as 'x', 'y', and 'z'). Once data has been put into a Dataset, which is essentially a dictionary of DataArrays, working with the data is highly intuitive and simple. The choice to work with xarray is based on the following:

(1) highly functional - for example, storing multidimensional datasets into one data structure reduces the task of processing and isolating desired sections of the data to a few lines of easy-to-read code, coordinates and dimensions are implicitly carried through operations which significantly simplifies code and makes it more readable.
(2) general - xarray handles common low-level manipulation of arbitrary datasets so that users can focus on the specifics that differ from dataset to dataset.
(3) extendable - easy to adopt new measurement modalities and datasets without having to rewrite or copy-paste

### Data organization

Although today there are many powerful databases for storing and organizing data (such as HDF5 or netCDF, which xarray is based on), for the sake of backward compatibility with existing methods of storing data, `orenstein-analysis` assumes the following structure for loading data.

At the core of any optics measurement in the Orenstein Lab is a 1-dimensional dataset. For example, in a pump-probe measurement we measure some signal (and other observables or diagnostics) as a function of time-delay, which gets stored as a text file of the form:

```
[METADATA]
...
...
[DATA]
time delay (ps) signal 1 (V) signal 2  (V)
... ... ...
... ... ...
```

Multidimensional datasets are built up in a directory as a set of such text files with names that specify additional coordinates. For example, a pump-probe experiment as a function of temperature might look like:

```
data_directory/
  pumpprobe_10K_100mW.txt
  pumpprobe_20K_100mW.txt
  pumpprobe_30K_100mW.txt
```

The `loader.load_measurement()` function loads 1 dimensional data from a single text file into an xarray Dataset, while `loader.load_ndim_measurement()` searches through a directory and assembles a multidimensional Dataset, extracting and organizing other coordinates based on the filename and a few simple user inputs (future work will include ability to search metadata for other coordinates). For example, after a call to `loader.load_ndim_measurement()`, the user will have access to a data set 'signal 1 (V)' which is a function of 'time delay (ps)' and 'temperature (K)'. It is emphasized that adding additional coordinates is trivial within this framework.

### Workflow

The intended workflow for `orenstein-analysis` reflects the underlying 1D nature of our experiments while still leaving options for post processing on multidimensional sets.

The main feature of `loader.load_measurement()` and `loader.load_ndim_measurement()` is the `instruction_set` **kwarg, which accepts a list of functions `[f1, f2, f3, ...]`. These are functions, typically defined within an analysis notebook, which accept an xarray Dataset as the only argument and return a modified Dataset. Both loading functions sequentially process each 1-dimensional Dataset after importing, such that if `ds` is the initial dataset, at the end of the loading operation the stored dataset is `f3(f2(f1(ds)))`. Following our example above, each time trace might be fit to a given functional for at each temperature and then the multidimensional Dataset will include a data variable containing the time constant as a function of temperature.

In addition, the `process.add_processed()` function works similarly, taking in a Dataset `ds` and instruction_set `[g1, g2, g3, ...]` and returning `g3(g2(g1(ds)))`. This can be used to build pipelines for processing multidimensional data, such as 2D Fourier transforms on already processed data.

### Example

The following is an example of a birefringence experiment the measures a balanced photodiode as a function of the co-rotation of two half-wave-plates and space (x and y). This example can be run in the tests directory of this package.

```
import os
from orenstein_analysis.measurement import loader
from orenstein_analysis.measurement import process
from orenstein_analysis.experiment import experiment_methods
import xarray as xr
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

# specify path to directory containing datafiles (and only datafiles)
path = os.path.dirname(os.path.realpath(__file__))+'/test_data/'

# define functions for processing, which accept a measurement and return a measurement. These functions wrap more elaborate functions which forms the backbone of the user input. It is in writing these functions that the user tailors to their application.
set_coord = lambda meas: process.define_dimensional_coordinates(meas, {'Polarization Angle (deg)':2*meas['Angle 1 (deg)']})

fit_bf = lambda meas: process.add_processed(meas, (experiment_methods.fit_birefringence, ['Polarization Angle (deg)', 'Demod x']))

# load and preliminary processing of data. Note the only other user input is in defining the new coordinates and python regex strings for extracting coordinate values from filename.
ndim_meas = loader.load_ndim_measurement(path, {'x ($\mu$m)':'_x[0-9]+', 'y ($\mu$m)':'_y[0-9]+'}, instruction_set=[set_coord, fit_bf])
print(ndim_meas)

# visualize data
fig, [ax1, ax2, ax3]  = plt.subplots(1,3)

ndim_meas['Demod x (fit)'].sel({'x ($\mu$m)':2800, 'y ($\mu$m)':2475}, method='nearest').plot(ax=ax1, label='fit')
ndim_meas['Demod x'].sel({'x ($\mu$m)':2800, 'y ($\mu$m)':2475}, method='nearest').plot(ax=ax1, marker='o', ms=8, linestyle='None', label='data')
ax1.legend()
ax1.set_title('Birefringence fit at x = 2800 and y = 2475')

ndim_meas['Birefringence Angle'].plot(ax=ax2, cmap='twilight')
ax2.set_title(r'EuIn$_2$As$_2$')

xr.plot.hist(ndim_meas['Birefringence Angle'], ax=ax3)

fig.set_size_inches(12, 4, forward=True)
plt.tight_layout()
plt.show()
```
![output from above code!](./tests/output.png)

### A few final remarks

The syntax for this package needs to be consistent, intuitive, and clear. All data is stored in a xarray Dataset as a DataArray. To distinguish from the xarray syntax so as to avoid confusion between code native to xarray and code native to this package, we will often refer to a Dataset as a measurement. Sometimes, however, the two terms are used interchangeably. If needed, DataArrays will be referred to as measurement_data

In addition, the package is divided into two module categories. (1) measurement methods, which are general purpose and do not reference any specific type of experiment, and (2) experimental methods, which is a space to add specific functionality on top of the measurement methods.

For more information on python regular expressions (regexp), which are used to loader.load_ndim_measurement() to extract coordinate values, see [this nice tutorial](https://docs.python.org/3/howto/regex.html#regex-howto)
