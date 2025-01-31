---
description: Tips for using Prefect when dealing with big data.
tags:
    - big data
    - flow configuration
    - parallel execution
    - distributed execution
    - caching
search:
  boost: 2
---

# Big data with Prefect

In this guide you'll learn tips for working with large amounts of data in Prefect.

Big data doesn't have a widely accepted, precise definition.
In this guide, we'll discuss methods to reduce the processing time or memory utilization of Prefect workflows, without editing your Python code.

## Optimizing your Python code with Prefect for big data

Depending upon your needs, you may want to optimize your Python code for speed, memory, compute, or disk space.

Prefect provides several options that we'll explore in this guide:

1. Remove task introspection with `quote` to save time running your code.
1. Write task results to cloud storage such as S3 using a block to save memory.
1. Save data to disk within a flow rather than using results.
1. Cache task results to save time and compute.
1. Compress results written to disk to save space.
1. Use a [task runner](/concepts/task-runners/) for parallelizable operations to save time.

### Remove task introspection

When a task is called from a flow, each argument is introspected by Prefect, by default.
To speed up your flow runs, you can disable this behavior for a task by wrapping the argument using [`quote`](https://docs.prefect.io/latest/api-ref/prefect/utilities/annotations/#prefect.utilities.annotations.quote).

To demonstrate, let's use a basic example that extracts and transforms some New York taxi data.

```python hl_lines="2 26" title="et_quote.py"
from prefect import task, flow
from prefect.utilities.annotations import quote
import pandas as pd


@task
def extract(url: str):
    """Extract data"""
    df_raw = pd.read_parquet(url)
    print(df_raw.info())
    return df_raw


@task
def transform(df: pd.DataFrame):
    """Basic transformation"""
    df["tip_fraction"] = df["tip_amount"] / df["total_amount"]
    print(df.info())
    return df


@flow(log_prints=True)
def et(url: str):
    """ET pipeline"""
    df_raw = extract(url)
    df = transform(quote(df_raw))


if __name__ == "__main__":
    url = "https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2023-09.parquet"
    et(url)
```

Introspection can take significant time when the object being passed is a large collection, such as dictionary or DataFrame, where each element needs to be visited.
Note that using `quote` reduces execution time at the expense of disabling task dependency tracking for the wrapped object.

### Write task results to cloud storage

By default, the results of task runs are stored in memory in your execution environment.
This behavior makes flow runs fast for small data, but can be problematic for large data.
You can save memory by writing results to disk.
In production, you'll generally want to write results to a cloud provider storage such as AWS S3.
Prefect lets you to use a storage block from a Prefect cloud integration library such as [prefect-aws](https://prefecthq.github.io/prefect-aws/) to save your configuration information.
Learn more about blocks [here](/concepts/blocks/).

Install the relevant library, register the block with the server, and create your storage block.
Then you can reference the block in your flow like this:

```python
...
from prefect_aws.s3 import S3Bucket

my_s3_block = S3Bucket.load("MY_BLOCK_NAME")

...
@task(result_storage=my_s3_block)

```

Now the result of the task will be written to S3, rather than stored in memory.

### Save data to disk within a flow

To save memory and time with big data, you don't need to pass results between tasks at all.
Instead, you can write and read data to disk directly in your flow code.
Prefect has integration libraries for each of the major cloud providers.
Each library contains blocks with methods that make it convenient to read and write data to and from cloud object storage.
The [moving data guide](/guides/moving-data/) has step-by-step examples for each cloud provider.

### Cache task results

Caching allows you to avoid re-running tasks when doing so is unnecessary.
Caching can save you time and compute.
Note that caching requires task result persistence.
Caching is discussed in detail in the [tasks concept page](/concepts/tasks.md/#caching).

### Compress results written to disk

If you're using Prefect's task result persistence, you can save disk space by compressing the results.
You just need to specify the result type with `compressed/` prefixed like this:

```python
@task(result_serializer="compressed/json")
```

Read about [compressing results with Prefect](/concepts/results/) for more details.
The tradeoff of using compression is that it takes time to compress and decompress the data.

### Use a task runner for parallelizable operations

Prefect's task runners allow you to use the Dask and Ray Python libraries to run tasks in parallel and distributed across multiple machines.
This can save you time and compute when operating on large data structures.
See the [guide to working with Dask and Ray Task Runners](/guides/dask-ray-task-runners/) for details.
