**Dask Design Document - 001**

# Backend-Library Dispatching for Dask Collection Creation

**Authors**:

- Richard Zamora (rzamora@nvidia.com)
- Ashwin Srinath
- Prem Sagar Gali
- Benjamin Zaitlen


**Created**: 2022-10-03 (Last Updated: 2022-13-09)


## Abstract

We propose a mechanism for configurable backend-library dispatching in the `dask.array.Array` and `dask.dataframe._Frame`-based classes. In contrast to the data-type dispatching already used within Dask at computation time, the new system is designed to operate at the collection level, with the primary target being the creation of new objects (i.e. input IO). With this system in place, the user's Dask configuration file can be used to specify that a non-default backend library should be used to create new collections.


## Motivation

The primary short-term goal of a configurable backend-dispatching mechanism is to enable users of Dask-cuDF to write the same `dask.dataframe` code for execution on both cpu- and gpu-based systems. However, the long-term goal of this feature is to enable Dask users to leverage any backend library in `dask.array` and `dask.dataframe`, as long as that library conforms to the minimal "array" or "dataframe" standard defined by [the data-api consortium](https://data-apis.org/), respectively. Therefore, this change is clearly well-aligned with the long-term improvement of Dask at both a software and hardware level: (1) We want to move away from hard-coded backend-library code within the collection APIs, and (2) we want to abstract the various hardware possibilities (cpu, gpu, fpga, tpu, etc...).


### Requirements

Given the motivation for library and hardware agnostic collection APIs, this proposal was prepared with certain requirements in mind:

- **Minimum Requirement**: Dask-cudf users should no longer need to import `dask_cudf` to work with a cudf-backed `dask.dataframe` collection.
- **Ideal Requirement**: There should be no required user-code changes for the `dask.dataframe` API when switching between "pandas" and "cudf" backend.
  - The user should **not** need to pass in special kwargs or `engine=`/`like=` arguments manually.
  - Optional tweaks may make sense for performance optimization, but the same code should "work" with the backend changed in the `dask.cofig` file.
- **Ideal Requirement**: Registration of a new backend should not require the user (or up-/down-stream library) to add any code to `dask.dataframe` or `dask.array`.
  - Dask should clearly define the necessary API for defining a new backend.


## Non-Goals

- This feature does not target type- or hardware-based computation dispatching on the worker. The new dispatching system only applies to collection-API usage on the client process.
- This feature does not target the conversion of an existing collection class to a different backend library after the initial collection is created. That is, if the backend configuration is set after the collection is already defined, the backend will **not** be moved to the desired library.
- This feature does not propose a gpu (or RAPIDS) on/off switch. In the future, the backend defaults can be modified to depend on other hardware-preference information stored in the `dask.config` options. However, this feature only calls for backend configuration options at the collection level.
- This feature does not mean Dask will take responsibility for testing a `cudf`-backed version of `dask.dataframe`. It is still the responsibility of the backend library (i.e. `cudf` and `dask_cudf`) to ensure that the collection and backend libraries are compatible.


## Detailed Description


### Designating the Backend (`dask.config`)

From the perspective of the typical Dask user, the only visible result of the proposed feature is the addition of a new field in `dask/dask/dask.yaml`/`dask-schema.yaml` (accessible from `dask.config`). For each of the targeted collections (Dask-Array and Dask-DataFrame), we propose the addition of the "backend.library" field. By default, "backend.library" will be set to "numpy" and "pandas" for Dask-Array and Dask-DataFrame, respectively. However, as shown in the code snippet below, this field can be changed with the existing `dask.config` interface to specify an alternative backend library.


```python
import dask

with dask.config.set({"dataframe.backend.library": "cudf"}):
    # Produce a cudf-backed collection
    ddf = dask.dataframe.read_parquet("./tmpdir")

with dask.config.set({"array.backend.library": "cupy"}):
    # Produce a cupy-backed collection
    darr = dask.array.ones(10, chunks=(5,))
```

### Registering a New Backend (`DaskBackendEntrypoint`)

In order to allow backend registration outside of the Dask source code, we propose that Dask approximately follow [the approach taken by xarray for custom backend interfaces](https://xarray.pydata.org/en/stable/internals/how-to-add-new-backend.html). That is, external libraries should be able to leverage "entrypoints" to tell Dask to register compatible backends in Dask-Array and Dask-DataFrame at run time. To this end, the external library could be expected to define all creation-dispatch logic within a `DaskDataFrameBackendEntrypoint` or `DaskArrayBackendEntrypoint` subclass.  The `__init__` method of the subclass would also be responsible for executing the necessary code to ensure that backend-specific (non-creation) dispatch functions are properly registered. For example, a cudf-based subclass would look something like the `CudfBackendEntrypoint` definition below:


```python
class CudfBackendEntrypoint(DaskDataFrameBackendEntrypoint):

    def __init__(self):
        # Importing this class will guarentee that compute
        # dispatch functions (e.g. `make_meta_dispatch`)
        # are registered, because they are defined and
        # registered in the same module.
        pass
    ...

    def read_json(self, *args, engine=None, **kwargs):
        # Use "pandas" backend with cudf-based engine
        with config.set({"dataframe.backend.library": "pandas"}):
            return dd.read_json(
                *args, engine=cudf.read_json, **kwargs
            )

    def read_orc(self, *args, **kwargs):
        from .io import read_orc

        # Use dask_cudf version of read_orc
        return read_orc(*args, **kwargs)
    ...
```

Once the `DaskBackendEntrypoint` subclass is defined, the new entrypoint can be declared in the library's `setup.py` file (specifying the class with a `"dask.backends"` entrypoint).

Note that the `CudfBackendEntrypoint` example above does not inherit from `PandasBackendEntrypoint`, even though it does **manually** leverage the "pandas" backend for some creation operations. This approach ensures that Dask will raise a `NotImplementedError` for any dispatchable creation function that is not explicitly defined for the "cudf" entrypoint. The next section will discuss where the set of all "dispatchable" functions are defined. 


### Defining dispatchable creation functions

The set of all dispatchable creation functions for Dask-DataFrame and Dask-Array should be defined in `DaskDataFrameBackendEntrypoint` and `DaskArrayBackendEntrypoint`, respectively. Whithin these base classes, the creation functions will be abstract in the sense that they will define the required argument signature, but will return `NotImplementedError`.  These creation functions should also be advertised within the dask-Dataframe and Dask-Array documentation, along-side an (advanced) tutorial on defining a custom collection backend.

**NOTE**: Although this work should make it easier for users to define custom collection backends, the data-centered dispatch system (used at compute time) will likley need further standardization before custom backed definitions are practical in general. There may also be some necessary work to revise internal Dask code that currently uses parts of the panda/numpy API that are outside the DataFrame/Array-API standards.


## Implementation Details

**Reference Implementation** (Draft):

- [Dask Component](https://github.com/rjzamora/dask/tree/backend-class) 
- [CuDF Component](https://github.com/rjzamora/cudf/tree/backend-class) ("cudf" entrypoint definition in `dask_cudf`)

### Dispatching Functions

As described above, we propose that all creation functions for a specific backend be defined within a single `DaskDataFrameBackendEntrypoint` or `DaskArrayBackendEntrypoint` subclass. The only subclasses defined within the dask source code will be the default reference subclasses for numpy, cupy and pandas. These entrypoint classes will be defined in the `backends.py` file for each collection. In order to avoid moving all numpy- and pandas-specific creation logic into `backends.py`, the existing creation functions will be registered to their respective entrypoint class "in place":


```python
from dask.dataframe.backends import dataframe_creation_dispatch
...

@dataframe_creation_dispatch.register_inplace("pandas")
def read_parquet(*args, **kwargs):
    ...
```

[See notes on moving backend-specific code](#moving-backend-specific-code), and [notes on dispatching docstrings](#dispatching-docstrings).

The actual dispatching of creation functions will require the definition of a new `BackendDispatch` class in a new `dask.backends` module (where `DaskBackendEntrypoint` will also be defined). In contrast to the existing `dask.utils.Dispatch` class, `BackendDispatch` will use a backend string label (e.g. "pandas") for registration and dispatching, and the dispatching logic will be implemented at the `__getattr__` level (rather than in `__call__`). More specifically, registered "keys" and "values" for the dispatch class will correspond to backend labels and `DaskBackendEntrypoint` subclasses, respectively. When some Dask-collection code calls something like `backend_dispatch.read_parquet`, dispatching logic will be used to return the appropriate `"read_parquet"` attribute for the current backend.


## Backward Compatibility

The default backend libraries for `dask.array` and `dask.dataframe` will continue to be `numpy` and `pandas`. Therefore, this feature should be completely backward compatible with older user code.


## Alternatives

The primary alternative to the dispatch-based changes proposed here is to standardize the `engine=` argument for all creation functions in the Array and DataFrame collection APIs.  The defaults for this `engine` arguments could depend on one or more fields in `dask.config`, but the logic for selecting/using the desired backend would need to be added to every creation function.  There are already a few Dask-DataFrame creation functions (e.g. `read_parquet`, `read_json`) that leverage an `engine` keyword to  effectively utilize different library backends for creation.  However, the specific usage of `engine=` is inconsistent between the various creation functions, and does **not** necessarily correspond to the use of a distinct dataframe (or array) backend library.  In fact, the “pandas” backend already supports multiple engine options in `read_parquet`, and so the concept of an “engine” is already a bit different from that of a DataFrame “backend”.  Therefore, it may be a significant challenge to design a general mapping between `engine` options and registered backends.

The alternative to the entry point-registration process proposed here is to follow the approach currently employed for `dask.utils.Dispatch`, where the user is expected to explicitly import the external code to ensure the alternative backend is properly registered. Otherwise, the backend definition would need to exist within the Dask source code itself.


## Discussion

### Revision Notes

- September 13, 2022: Automated library-fallback behavior has been removed from the proposal. It is now the responsibility of the backed to implement and document fallback behavior if/when desired.

### Open Questions

#### Moving backend-specific code

**Notes**:

- This proposal does not (yet) specify *where* Pandas- and Numpy-specific code should live. The current reference implementation defines these default implementations in place. However, there may be interest in re-organizing the collection codebase to be more explicit about library-specific code.


#### Dispatching docstrings

**Relevant review comments**:

- "I'm interested in the docstrings. Often we inherit them from pandas and just augment them a bit, but maybe they should also be part of the dispatch mechanism." -jsignell

**Notes**:

- This proposal does not (yet) specify how doctrings should be defined for dispatchable functions.
