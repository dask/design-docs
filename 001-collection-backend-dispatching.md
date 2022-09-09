**Dask Design Document - 001**

# Backend-Library Dispatching for Dask Collection Creation

**Authors**:

- Richard Zamora (rzamora@nvidia.com)
- Ashwin Srinath
- Prem Sagar Gali
- Benjamin Zaitlen


**Created**: 2022-03-10 (Last Updated: 2022-09-08)


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

From the perspective of the typical Dask user, the only visible result of the proposed feature is the addition of new fields in `dask/dask/dask.yaml`/`dask-schema.yaml` (accessible from `dask.config`). For each of the targeted collections (Dask-Array and Dask-DataFrame), we propose the addition of "backend", and "backend-options" fields. By default, the "backend" field will be set to "numpy" and "pandas" for Dask-Array and Dask-DataFrame, respectively. However, as shown in the code snippet below, this field can be changed with the existing `dask.config` interface to specify an alternative backend library.


```python
import dask

with dask.config.set({"dataframe.backend.library": "cudf"}):
    # Produce a cudf-backed collection
    ddf = dask.dataframe.read_parquet("./tmpdir")

with dask.config.set({"array.backend.library": "cupy"}):
    # Produce a cupy-backed collection
    darr = dask.array.ones(10, chunks=(5,))
```

Since it is unlikely that an alternative backend will support all numpy- or pandas-based data-creation functions available in the collection API, we also propose the "allow-fallback" and "warn-fallback" subfields of "backend". When the "allow-fallback" field is set to `True` (default), then the backend's designated fallback class will be used to perform IO, and the result will be moved from the fallback backend. The user should also have the option to enable or disable warnings when this fallback behavior occurs:


```python
backend_options = {
    "dataframe.backend.library": "cudf",
    # Allow user to specify if a fallback backend library
    # should be used, and if a warning should be raised when
    # this occurs:
    "dataframe.backend.allow-fallback" : True,
    "dataframe.backend.warn-fallback" : False,
}

dask.config.set(backend_options)
```

The specific configuration options proposed here are certainly not set in stone.  However, we do feel that the user should have complete control over fallback behavior.  For example, the user should be able to specify if falling back to "numpy"/"pandas" should result in a warning or error message, or if it should be ignored altogether. [See notes on supporting **multiple** fallback options](#supporting-multiple-fallback-options).


### Registering a New Backend (`DaskBackendEntrypoint`)

In order to allow backend registration outside of the Dask source code, we propose that Dask approximately follow [the approach taken by xarray for custom backend interfaces](https://xarray.pydata.org/en/stable/internals/how-to-add-new-backend.html). That is, external libraries should be able to leverage "entrypoints" to tell Dask to register compatible backends in Dask-Array and Dask-DataFrame at run time. To this end, the external library could be expected to define all dispatch IO logic within a `DaskBackendEntrypoint` subclass.  For example, a cudf-based subclass would look something like the `CudfBackendEntrypoint` definition below:

```python
class CudfBackendEntrypoint(DaskBackendEntrypoint):

    @cached_property
    def fallback(self):
        """Fallback entrypoint object to use for missing attributes

        Returning anything other than ``None`` requires that
        ``move_from_fallback`` be properly defined.
        """
        return PandasBackendEntrypoint()

    def move_from_fallback(self, ddf):
        """Move a Dask collection from the fallback backend"""
        if isinstance(ddf._meta, pd.DataFrame):
            return ddf.map_partitions(cudf.DataFrame.from_pandas)
        elif isinstance(ddf._meta, pd.Series):
            return ddf.map_partitions(cudf.Series.from_pandas)
        return ddf
    ...

    def read_json(self, *args, engine=None, **kwargs):
        return self.fallback.read_json(*args, engine=cudf.read_json, **kwargs)

    def read_orc(self, *args, **kwargs):
        return dask_cudf.read_orc(*args, **kwargs)
    ...
```

Once the `DaskBackendEntrypoint` subclass is defined, the new entrypoint can be declared in the library's `setup.py` file (specifying the class with a `"dask.backends"` entrypoint).

Note that the `CudfBackendEntrypoint` example above selects `PandasBackendEntrypoint` as the fallback entrypoint class, but does not directly inherit from this reference class. This approach allows Dask to properly move data from the pandas fallback for any IO functions that lack cudf-specific definitions. If the cudf subclass were to directly inherit from `PandasBackendEntrypoint`, then "fallback" behavior would not result in data-movement or user warnings. [See notes on defining dispatchable IO functions](#defining-dispatchable-io-functions).


## Implementation Details

**Reference Implementation** (Draft):

- [Dask Component](https://github.com/rjzamora/dask/tree/backend-class) 
- [CuDF Component](https://github.com/rjzamora/cudf/tree/backend-class) ("cudf" entrypoint definition in `dask_cudf`)

### Dispatching Functions

As described above, we propose that all IO functions for a specific backend be defined within a single `DaskBackendEntrypoint` subclass. The only subclasses defined within the dask source code will be the default reference subclasses for numpy and pandas. These entrypoint classes will be defined in the `backends.py` file for each collection, and will define all dispatch-able IO functions.

The actual dispatching of IO functions will require the definition of a new `BackendDispatch` class in `dask.utils`. In contrast to the existing `dask.utils.Dispatch` class, `BackendDispatch` will use a backend string label (e.g. "pandas") for registration and dispatching, and the dispatching logic will be implemented at the `__getattr__` level (rather than in `__call__`). More specifically, registered "keys" and "values" for the dispatch class will correspond to backend labels and `DaskBackendEntrypoint` subclasses, respectively. When some Dask-collection code calls something like `backend_dispatch.read_parquet`, dispatching logic will be used to return the appropriate `"read_parquet"` attribute for the current backend.

In order to avoid moving numpy- or pandas-specific IO logic into `backends.py`, the existing IO functions will  be renamed to `*_pandas`, and referenced "in place". To insure that the real IO functions are still defined at the same absolute and relative paths, and that the original doc-strings are recognized, we can add a few lines below the `*_pandas` definition to direct the original function name to the dispatching machinery:


```python
def read_parquet_pandas(...):
    <Previous read_parquet definition and doc-string>

def read_parquet(*args, **kwargs):
    return dataframe_backend_dispatch.read_parquet(*args, **kwargs)

read_parquet.__doc__ = read_parquet_pandas.__doc__
```

[See notes on moving backend-specific code](#moving-backend-specific-code), and [notes on dispatching docstrings](#dispatching-docstrings).


## Backward Compatibility

The default backend libraries for `dask.array` and `dask.dataframe` will continue to be `numpy` and `pandas`. Therefore, this feature should be completely backward compatible with older user code.


## Alternatives

The primary alternative to the dispatch-based changes proposed here is to standardize the `engine=` argument for all input-IO functions in the Array and DataFrame collection APIs.  The defaults for this `engine` arguments could depend on one or more fields in `dask.config`, but the logic for selecting/using the desired backend would need to be added to every IO function.  There are already a few Dask-DataFrame IO functions (e.g. `read_parquet`, `read_json`) that leverage an `engine` keyword to  effectively utilize different library backends for IO.  However, the specific usage of `engine=` is inconsistent between the various IO functions, and does **not** necessarily correspond to the use of a distinct dataframe (or array) backend library.  In fact, the “pandas” backend already supports multiple engine options in `read_parquet`, and so the concept of an “engine” is already a bit different from that of a DataFrame “backend”.  Therefore, it may be a significant challenge to design a general mapping between `engine` options and registered backends.

The alternative to the entry point-registration process proposed here is to follow the approach currently employed for `dask.utils.Dispatch`, where the user is expected to explicitly import the external code to ensure the alternative backend is properly registered. Otherwise, the backend definition would need to exist within the Dask source code itself.


## Discussion

### Open Questions

#### Supporting multiple fallback options

**Relevant review comments**:

- "Should the user be able to control what library they fallback to? I am imagining a future where you might want to fallback from sparse to either pandas or cudf." -jsignell

**Notes**:

- The proposed design requires every new backend-entrypoint definition to define a `fallback` property. The entrypoint must also implement the necessary logic to **move** data from the dedicated fallback library.
- It should be possible to support multiple fallback options within the external `DaskBackendEntrypoint.fallback` definition. It may make sense to add an official config option for this in Dask (e.g. `"dataframe.backend.fallback-library"`). However, it would need to be the responsibility of the external entrypoint definition to use and validate this field.


#### Defining dispatchable IO functions

**Notes**:

- The current design explicitly discourages the use of `NotImplementedError` in the interest of supporting seamless fallback behavior. Therefore, we cannot simply define all dispatchable functions at the `DaskBackendEntrypoint` level (or in collection-specific subclasses). This limitation means that the "dispatchable" functions will need to be advertised in the documentation.


#### Moving backend-specific code

**Notes**:

- This proposal does not (yet) specify *where* Pandas- and Numpy-specific code should live. The current reference implementation defines these default implementations in place. However, there may be interest in re-organizing the collection codebase to be more explicit about library-specific code.


#### Dispatching docstrings

**Relevant review comments**:

- "I'm interested in the docstrings. Often we inherit them from pandas and just augment them a bit, but maybe they should also be part of the dispatch mechanism." -jsignell

**Notes**:

- This proposal does not (yet) specify how doctrings should be defined for dispatchable functions.
