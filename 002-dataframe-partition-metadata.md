**Dask Design Document - 002**

# Partition Metadata Tracking in Dask DataFrame [Draft]

**Author**:

- Richard Zamora (rzamora@nvidia.com)

**Created**: 2022-18-11 (Last Updated: 2022-18-11)


## Abstract

We propose the consolidation of most `dask.dataframe` metadata attributes (i.e. `_meta` and `divisions`) into a single (`partition_metadata`) attribute of `dask.dataframe._Frame`. This unified metadata object shall correspond to an immutable `PartitionMetadata` class that is designed to track useful schema and partitioning information about the Dask-DataFrame collection. The specific goals of this refactor are: (1) to make metadata tracking more centralized, and (2) to enable the tracking of light-weight statistics and partitioning information that can be used to avoid unnecessary data ingest and shuffling.


## Motivation

The primary motivation for partition-metadata tracking is to improve performance for common Dask-DataFrame workflows. Some opportunities for improvement include the following examples:

**Example 1 - `len(dd.read_parquet())`**

Calculating the length of a Parquet-based DataFrame collection currently requires data ingest, which can be surprisingly slow on large and/or remote datasets. Since the length of each row-group is stored in the Parquet metadata, this information can be accessed much more quickly if we track partition-length info within the metadata of the DataFrame collection itself.

**Example 2 - `dd.read_parquet().groupby/merge/sort_values()`**

In addition to checking the length of a Parquet-based DataFrame collection, it is also relatively-common practice to perform a `groupby`, `merge`, or `sort_values` operation immediately after a `read_parquet` call. Since Dask-DataFrame will only track `divisions` on the index column, these global operations almost always result in a global shuffle or naive tree reduction. In many cases, however, the dataset is already partitioned in such a way that significant data movement can be avoided, and the min/max column statistics are available in the Parquet metadata. Therefore, if we are able to track min/max column statistics for more than the designated index column, we can skip (or simplify) the data-movement stage for many instances of `groupby`, `merge` and `sort_values`.

**Example 3 - `ddf.groupby/merge/shuffle().groupby/merge()`**

Even when a DataFrame collection does not originate from Parquet, the use of a data-movement operation (like `groupby`, `merge`, or `shuffle`) can also establish how the collection is partitioned. This information can then be used to avoid redundant data movement in follow-up operations. For example, we can avoid one or more global shuffle operations if we are merging two collections, and one or both of those collections was already grouped or shuffled by the columns we are merging on.


## Goals

The minimum requirement for this work is to consolidate the management of both `meta` and `divisions` within the same (immutable) attribute on the `dask.dataframe._Frame` class. In order to improve the performance of typical Dask-DataFrame workflows, the proposed partition-metadata class should be designed to track the following information:

- **Schema**: Column names and dtypes
  - Still tracked as an empty `pd`/`cudf` `DataFrame` object (i.e. `meta`)
- **Column statistics**: Dictionary of min/max statistics for each column in the dataset
  - Dictionary may be partially populated
  - Should support mechanism for "lazy loading" from Parquet metadata (*Ideal* requirement)
- **Partition lengths**: Tuple of row-counts for each partition in the collection
  - Should support mechanism for "lazy loading" from Parquet metadata (*Ideal* requirement)
- **Partitioning info**: Dictionary of partitioning information (*Ideal* requirement)
  - Should reflect "ordered" partitioning for an index column with known `divisions`
  - Should be updated for "hash" partitioning after each shuffle operation


### Note on High-level Graph/Query Optimization

Partition-metadata tracking is not intended to replace plans for high-level query optimization in Dask. In fact, by consolidating metadata management into a single class, we make it much easier to move collection-metadata tracking into the high-level graph (or high-level expression system) in the near or distant future. Therefore, **this proposal should be considered a useful stepping stone on the path to high-level graph/query optimization in Dask-DataFrame.**


## Detailed Description

**Reference Implementation:**

- [[POC] Introduce partition_metadata attribute to DataFrame](https://github.com/dask/dask/pull/9473)

### Part 1. Introduce the `PartitionMetadata` Class

The central change proposed in this work is the introduction of a new `PartitionMetadata` class. This class should be implemented with immutable attributes in mind (possibly by using [Python dataclasses](https://docs.python.org/3/library/dataclasses.html)). For example:

```python
class PartitionMetadata:
    """Container for DataFrame partition metadata"""

    __meta: Any
    __npartitions: int
    __divisions: tuple | None
    __partitioning: dict
    __partition_lens: tuple | Callable | None
    __column_statistics: dict

    ... Methods to coordinate/manage attributes ...
```

In order to *use* this new class in `_Frame` (and its sub-classes), those classes must be refactored so that a new `_Frame.partition_metadata` attribute is used to query and set traditional metadata properties like `meta` and `divisions`. Due to the immutability of `PartitionMetadata`, the act of modifying `meta` or `divisions` must result in the full replacement of the collection's `partition_metadata` attribute.


#### Update `new_dd_object` and `_Frame` Constructors

To evolve the `partition_metadata` attribute within Dask-Dataframe operations, the existing `new_dd_object` function (and related `_Frame` constructors) must be updated to accept `PartitionMetadata`-based objects in several places where traditional `meta` objects are currently expected. By expanding the positional `meta` argument to **also** accept a `PartitionMetadata` object, it becomes relatively easy to improve metadata evolution throughout the Dask-DataFrame codebase in an incremental way:


```python
def new_dd_object(dsk, name, meta, divisions=None, parent_meta=None):
    """Generic constructor for dask.dataframe objects.
  
    Decides the appropriate output class based on the type of `meta` provided.
    """
    partition_metadata = None
    if isinstance(meta, PartitionMetadata):
        ... use meta to create partition_metadata ...
    else:
        ... use meta/divisions to create partition_metadata ...
    ...
```

To clarify, since the `meta` argument of `new_dd_object` can be either an empty pandas/cudf object **or** a proper `PartitionMetadata` object, there is no requirement to immediately update all `dask.dataframe` logic at once to recognize the `partition_metadata` attribute.


### Part 2. Track Statistics

In order to avoid unnecessary data ingest and/or data movement in operations like `len`, `groupby`, `shuffle` and `sort_values`, it must be possible to track and query optional partition-length information and min/max column statistics. Since partition-lengths are common for all columns, we propose the use of two distinct `PartitionMetadata` properties:

- `partition_lens: Tuple[int]`: Tuple of integer partition lengths
- `column_statistics: Dict[str, Dict[str, Iterable]]`: Outer keys correspond to column names, while inner keys correspond to statistic labels (e.g. "min" and "max")

Since the overhead for parsing this information from Parquet metadata can be non-trivial, we propose a simple mechanism for populating `partition_metadata.partition_lens` and/or `partition_metadata.column_statistics` *lazily*. The simplest implementation allows a `Callable` or `Delayed` object to be assigned to one or both of these properties. For example, if `column_statistics["date"]` is a `Delayed` object, Dask may compute that object to update the column statistics for the "date" column.


### Part 3. Track Partitioning State

Although Dask-DataFrame already tracks sorted-index partitioning through the `divisions` property, many real-world workflows result in other forms of partitioning. For example, it is common for the collection to be partitioned by one or more non-index columns. This is sometimes the case on disk, but also occurs frequently after an explicit `shuffle`, `groupby`, `merge`, or `set_index` operation.

We propose the inclusion of a `PartitionMetadata.partitioning: dict` property to track the full partitioning state for the global DataFrame collection. Each key of the `partitioning` dictionary corresponds to a tuple of column/index names that the collection is partitioned by, and the values correspond to a tuple encoding *how* the collection is partitioned.


## Alternatives

TODO

## Discussion

TODO
