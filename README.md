# Beginner GRIB processing guide

This guide will walk you through how to:
 - Filter DMI's HARMONIE GRIB files to a single parameter
 - Aggregate all timesteps into a single file
 - Import into `xarray`, a standard Python tool for working with scientific data
 - Export to [NetCDF](https://docs.unidata.ucar.edu/netcdf-c/current/faq.html#What-Is-netCDF), a standard machine 
   independent format for working with scientific data

This guide will use standard Python tools as well as official ECMWF tooling for GRIB.

## Prerequisites

First you need to download all the GRIB files of the forecast model run you wish to process. It is up to you to do this.

The following commands shows how you could download a HARMONIE NEA SF forecast on Linux with the command line tools `curl` and `jq`:
```bash
# Latest model run:
curl https://dmigw.govcloud.dk/v1/forecastdata/collections/harmonie_nea_sf/items?api-key=??? | jq '[.features[].properties.modelRun]|unique|sort|.[-1]'
# Retrieve download links from latest model run, paste the output of the previous command into where it says ?modelRun\=
curl https://dmigw.govcloud.dk/v1/forecastdata/collections/harmonie_nea_sf/items\?modelRun\=???\&api-key=??? | jq -r '.features[]|[.asset.data.href,.id]|@csv' > latest_model_run_links
# Download the actual GRIB files
cat latest_model_run_links | awk -F, '{print $1, $2}' | xargs -L1 sh -c 'curl $0 > $1' 
```
Note: you need to use your own API key in the `curl` commands.

## Setup

Dependencies are most easily installed through `conda`, [official conda documentation](https://docs.conda.io/en/latest/). 
You need to install the following packages in your conda environment of choice:
```bash
conda install -c conda-forge xarray dask netCDF4 bottleneck eccodes cfgrib
```

DMI's HARMONIE model uses some parameters defined by DMI themselves. In order for our tooling to read and understand 
these parameters, it needs [these configuration tables](https://confluence.govcloud.dk/download/attachments/76153348/dmi_grib_definitions_v1.0.0.tar?version=1&modificationDate=1662378216765&api=v2).
[This link](https://confluence.govcloud.dk/pages/viewpage.action?pageId=76153348) explains how to install them by 
downloading a tarball, extracting the folder and defining the environment variable `ECCODES_DEFINITION_PATH` to point 
at the extracted folder.

## Processing GRIB data

### Filter and aggregate GRIB files

By installing the `eccodes` package in the "Setup" section of this guide, you now have access to the [ECMWF GRIB tools](https://confluence.ecmwf.int/display/ECC/GRIB+tools).
These are command line tools for working with GRIB.

Using `grib_ls` on one of your downloaded GRIB files will show metadata about contained parameters, which levels the 
parameter exists in, what type of levels are used etc.

In our example we will work with the temperature parameter, so using `grib_ls` and looking for the temperature 
parameter (which has short name `t`) we will see:
```bash
> grib_ls -p typeOfLevel,level,shortName HARMONIE_NEA_SF_20XX-XX-XXTXXXXXXZ_20XX-XX-XXTXXXXXXZ.grib
typeOfLevel  level        shortName    
...
heightAboveGround  50           t           
heightAboveGround  100          t           
heightAboveGround  150          t           
heightAboveGround  250          t           
...
heightAboveGround  2            t           
heightAboveGround  0            t           
...
hybrid       65           t           
...
heightAboveSea  2            t           
...
heightAboveGround  760          t           
heightAboveGround  772          t           
heightAboveGround  802          t           
heightAboveGround  950          t           
...
```
The output from this command tells us that temperature exists in various levels. It also exists in a few different
`typeOfLevel`'s, like height above ground, height above sea and "hybrid". In this example we want to work with 
temperature forecasted in heights above ground.

Choosing a `typeOfLevel` to work with is necessary when later importing the data into `xarray`. Our tooling simply does 
not know how to combine several different `typeOfLevel`'s into one dataset. We could import several parameters into one 
dataset, as long as they share `typeOfLevel`.

In order to filter out the temperature parameter from the GRIB file, we use the command `grib_filter`, full documentation [here](https://confluence.ecmwf.int/display/ECC/grib_filter).

`grib_filter` needs a filtering script that tells it to choose GRIB data with the `shortName` as `t` and `typeOfLevel` as `heightAboveGround`:
```
# filter_temp.txt
if (shortName is "t" and typeOfLevel is "heightAboveGround") {
    write "./temp.grib";
}
```
The `write "./temp.grib"` determines the output file created by `grib_filter`.

`grib_filter` will also be able to aggregate GRIB files if they are all given as input parameters. We can now create a GRIB 
file containing only temperatures defined in heights above ground with all the time steps aggregated into one file:
```bash
grib_filter filter_temp.txt HARMONIE_NEA_SF_*
```

Running `grib_ls -p shortName,level,stepRange temp.grib` shows us that our output file `temp.grib` contains only the 
temperature parameter in various `level`'s and `stepRange`'s. The step range defines the step of the forecast since 
the beginning of the forecast. Because the HARMONIE model works with hours, a `stepRange` of 10 would be 10 hours 
after the beginning of the forecast. 

This shows us `grib_filter` has filtered everything except temperature above ground and has aggregated all our time steps!

### Importing into `xarray`

Thanks to installing the `cfgrib` package, it is almost trivial to import GRIB data from HARMONIE into `xarray`:

```python
import xarray as xr

# For very large files, maybe adjust chunk size accordingly
ds = xr.open_dataset('temp.grib', engine="cfgrib")
```

This will index the GRIB file, and these indexes will show up as files in your folder with the name: `temp.grib.XXX.idx`.
Do _NOT_ delete this file.

You can now work with the data as a regular `xarray` dataset.

It might be worth noting that the dataset does not contain the data in a regular lat/lon grid, but rather the lat/lon 
coordinates are defined in a 2-dimensional array as shown here:
```python
>>> ds.coords
Coordinates:
time               datetime64[ns] ...
* step               (step) timedelta64[ns] ...
heightAboveGround  float64 ...
latitude           (y, x) float64 44.38 44.39 44.4 ... 72.64 72.64 72.64
longitude          (y, x) float64 -6.256 -6.228 -6.2 ... 36.87 36.94 37.01
valid_time         (step) datetime64[ns] ...
```

This is because the HARMONIE model uses a rotated lat/lon grid. The un-rotated coordinates are in a regular grid around the equator
and prime meridian. For a more official explanation, please see [DMI's documentation](https://confluence.govcloud.dk/display/FDAPI/About+HARMONIE).

`cfgrib` has rotated all the coordinates for us, so `latitude` and `longitude` contains the real world coordinates. The
coordinate at `ds.coords['longitude'][4][5]` contains the longitude for
`ds.data_vars[some_var][some_step][some_height][4][5]`, same goes for latitude.

### Exporting to NetCDF

One might assume we need to use the GRIB command line tool `grib_to_netcdf` installed through `eccodes`. This 
does not work for rotated grids, which HARMONIE sadly is.

Luckily, `xarray` has a built-in method for exporting to NetCDF which works:

```python
ds.to_netcdf('temp.nc')
```