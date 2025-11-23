# Weather data set

This data set comes from ERA5 and contains surface data for 5 cities:

- Berlin
- Brest
- London
- Marseille
- Paris

There is 1 netcdf file per variable per city.
Each file has 40 years of data (except Paris: 41 years)

Some variables are *Analysis* variables
Some variables are *Forecast* variables and are shifted by 3h compared to analysis variable. (you need to rely on the timestamp that is in the netcdf file).

You can use the xarray library to load netcdf files in python and you can then convert this data set into a pandas DataFrame.
