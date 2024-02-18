# Polars-API-Compat

Extremely lightweight compatibility layer between Polars, pandas, and cuDF.

Seamlessly support all three, without depending on any of them!

- ✅ Just use a subset of the Polars API, no need to learn anything new
- ✅ **No dependencies** (not even Polars), keep your library lightweight
- ✅ Separate Lazy and Eager APIs
- ✅ Use the Polars Expressions API

More supported libraries to come (probably starting with Modin)!

## Installation

For now, just
```
pip install git+https://github.com/MarcoGorelli/polars-api-compat
```

## Usage

There are three steps to writing dataframe-agnostic code using Polars-API-Compat:

1. use `polars_api_compat.to_polars_api` to translate a pandas, Polars, or cuDF dataframe
   with the Polars API
2. use the subset of the Polars API defined in https://github.com/MarcoGorelli/polars-api-compat/blob/main/polars_api_compat/spec/__init__.py.
3. use `polars_api_compat.to_original_object` to return an object to the user in their original
   dataframe flavour. For example:

   - if you started with pandas, you'll get pandas back
   - if you started with Polars, you'll get Polars back
   - if you started with cuDF, you'll get cuDF back (and computation will happen natively on the GPU!)
   
## Example

Here's an example of a dataframe agnostic function:

```python
from typing import TypeVar
import pandas as pd
import polars as pl

from polars_api_compat import to_polars_api, to_original_object

AnyDataFrame = TypeVar("AnyDataFrame")


def my_agnostic_function(
    suppliers_native: AnyDataFrame,
    parts_native: AnyDataFrame,
) -> AnyDataFrame:
    suppliers, pl = to_polars_api(suppliers_native, version="0.20")
    parts, _ = to_polars_api(parts_native, version="0.20")
    result = (
        suppliers.join(parts, left_on="city", right_on="city")
        .filter(
            pl.col("color").is_in(["Red", "Green"]),
            pl.col("weight") > 14,
        )
        .group_by("s", "p")
        .agg(
            weight_mean=pl.col("weight").mean(),
            weight_max=pl.col("weight").max(),
        )
    )
    return to_original_object(result.collect())
```
By which I mean, if you pass in a pandas, Polars, or cuDF dataframe, the output will be the same!

Let's try it out:
```python
suppliers = {
    "s": ["S1", "S2", "S3", "S4", "S5"],
    "sname": ["Smith", "Jones", "Blake", "Clark", "Adams"],
    "status": [20, 10, 30, 20, 30],
    "city": ["London", "Paris", "Paris", "London", "Athens"],
}
parts = {
    "p": ["P1", "P2", "P3", "P4", "P5", "P6"],
    "pname": ["Nut", "Bolt", "Screw", "Screw", "Cam", "Cog"],
    "color": ["Red", "Green", "Blue", "Red", "Blue", "Red"],
    "weight": [12.0, 17.0, 17.0, 14.0, 12.0, 19.0],
    "city": ["London", "Paris", "Oslo", "London", "Paris", "London"],
}

print("pandas output:")
print(
    my_agnostic_function(
        pd.DataFrame(suppliers),
        pd.DataFrame(parts),
    )
)
print("\nPolars output:")
print(
    my_agnostic_function(
        pl.LazyFrame(suppliers),
        pl.LazyFrame(parts),
    )
)
```
```
pandas output:
    s   p  weight_mean  weight_max
0  S1  P6         19.0        19.0
1  S2  P2         17.0        17.0
2  S3  P2         17.0        17.0
3  S4  P6         19.0        19.0

Polars output:
shape: (4, 4)
┌─────┬─────┬─────────────┬────────────┐
│ s   ┆ p   ┆ weight_mean ┆ weight_max │
│ --- ┆ --- ┆ ---         ┆ ---        │
│ str ┆ str ┆ f64         ┆ f64        │
╞═════╪═════╪═════════════╪════════════╡
│ S1  ┆ P6  ┆ 19.0        ┆ 19.0       │
│ S3  ┆ P2  ┆ 17.0        ┆ 17.0       │
│ S4  ┆ P6  ┆ 19.0        ┆ 19.0       │
│ S2  ┆ P2  ┆ 17.0        ┆ 17.0       │
└─────┴─────┴─────────────┴────────────┘
```
Magic! 🪄 

## Scope

If you maintain a dataframe-consuming library, then any function from the Polars API which you'd
like to be able to use is in-scope, so long as it can be supported without too much difficulty
for at least pandas and cuDF.

Feature requests are more than welcome!

## Related Projects

This is not Ibis. Polars-API-Compat lets each backend do its own optimisations, and only provides
a lightweight (~30 kilobytes) compatibility layer with the Polars API.
Ibis applies its own optimisations to different backends, is a heavyweight
dependency (~400 MB), and defines its own API.

This is not intended as a DataFrame Standard. Please see the Consortium for Python Data API Standards
for such work. This is only intended to be a subset of the Polars API and does not aim to be a Standard.
