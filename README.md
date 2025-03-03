[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
![Tests](https://github.com/Breakthrough-Energy/PreREISE/workflows/Pytest/badge.svg)

# PreREISE
This package gathers and builds demand, hydro, solar, and wind profiles. The profiles
are needed to run scenarios on the U.S. electrical grid. PreREISE is part of a set of
packages representing Breakthrough Energy's power system model.

More information regarding the installation of the model as well as the contribution
guide can be found [here](https://breakthrough-energy.github.io/docs/).


## 1. Setup/Install
Here are the instructions to install the **PreREISE** package. We strongly recommend
that you pick one of the following options.


### A. Using pipenv
If not already done, install `pipenv` by following the instructions on their
[webpage](https://pipenv.pypa.io/en/latest/). Then run:
```bash
pipenv sync
pipenv shell
```
in the root folder of the package. The first command will create a virtual environment
and install the dependencies. The second command will activate the environment.


### B. Using the ***requirements.txt*** file
First create an environment using `venv` (more details
[here](https://docs.python.org/3/library/venv.html)). Note that `venv` is included in
the Python standard library and requires no additional installation. Then, activate your
environment and run:
```bash
pip install -r requirements.txt
```
in the root folder of the package.


### C. Path
Whatever method you choose, if you wish to access the modules located in **PreREISE**
from anywhere on your machine, do:
```bash
pip install .
```
in the root folder of your package or alternatively, setup the `PYTHONPATH` global
variable.


## 2. Gather Data for Simulation
This module aims at gathering the required data for building the profiles

### Profile Naming Convention
We call the time series profiles directly generated from mathematical/heuristic models
without any scaling (as we do in different scenarios) as 'raw' profiles.

The name of a raw profile in the simulation framework must be of the form:

[**type**]\_v\[**D**],

where **type** specifies the profile type  ('*demand*', '*hydro*', '*solar*', '*wind*')
and **D** refers to the date the file has been created. Month (3 letters, e.g., Jan) and
year (4 digits, e.g., 2021) are used for the date. If two or more profiles of the same
**type** are created the same month of the same year, the different versions will be
differentiated using a lowercase letter added right after the year, e.g,
***demand_vJan2021.csv*** (first), ***demand_vJan2021b.csv*** (second),
***demand_vJan2021c.csv*** (third) and so forth.


### A. Wind data

#### i. Rapid Refresh
[RAP][RAP] (Rapid Refresh) is the continental-scale NOAA hourly-updated
assimilation/modeling system operational at the National Centers for Environmental
Prediction (NCEP). RAP covers North America and is comprised primarily of a numerical
weather model and an analysis system to initialize that model. RAP provides, every hour
ranging from May 2012 to date, the U and V components of the wind speed at 80 meter
above ground on a 13x13 square kilometer resolution grid every hour. Data can be
retrieved using the NetCDF Subset Service. Information on this interface is described
[here][NetCDF]. Note that the dataset is incomplete and, consequently, missing entries
need to be imputed.

Once the U and V components of the wind are converted to a non-directional wind speed
magnitude, this speed is converted to power using wind turbine power curves. Since real
wind farms are not currently mapped to TAMU network farms, a capacity-weighted average
wind turbine power curve is created for each state based on the turbine types reported
in [EIA Form 860][EIA 860]. The wind turbine curve for each real wind farm is looked up
from a database of curves (or the *IEC class 2* power curve provided by NREL in the
[WIND Toolkit documentation][WIND_doc]) is used for turbines without curves in the
database), and scaled from the real hub heights to 80m hub heights using an alpha of
0.15. These height-scaled, turbine-specific curves are averaged to obtain a state curve
translating wind speed to normalized power. States without wind farms in
[EIA Form 860][EIA 860] are represented by the *IEC class 2* power curve.

Each turbine curve represents the instantaneous power from a single turbine for a given
wind speed. To account for spatio-temporal variations in wind speed (i.e. an hourly
average wind speed that varies through the hour, and a point-specific wind speed that
varies throughout the wind farm), a distribution is used: a normal distribution with
standard deviation of 40% of the average wind speed. This distribution tends to boost
the power produced at lower wind speeds (since the power curve in this region is convex)
and lower the power produced at higher wind speeds (since the power curve in this region
is concave as the turbine tops out, and shuts down at higher wind speeds). This tracks
with the wind-farm level data shown in NREL's validation report.

Check out the **[rap_demo.ipynb][RAP_notebook]** notebook for demo.


### B. Solar data

#### i. The Gridded Atmospheric Wind Integration National Dataset Toolkit
The [Gridded Atmospheric WIND (Wind Integration National Dataset) Toolkit][WIND_web]
provides 1-hour resolution irradiance data for 7 years, ranging from 2007 to 2013, on a
uniform 2x2 square kilometer grid that covers the continental U.S., the Baja Peninsula,
and parts of the Pacific and Atlantic oceans. Data can be accessed using the Highly
Scalable Data Service. NREL wrote [example notebooks][NREL_notebooks] that demonstrate
how to access the data.

Power output is estimated using a simple normalization procedure. For each solar plant
location the hourly Global Horizontal Irradiance (GHI) is divided by the maximum GHI
over the period considered and multiplied by the capacity of the plant. This procedure
is referred to as naive since it only accounts for the plant capacity. Note that other
factors can possibly affect the conversion from solar radiation at ground to power such
as the temperature at the site as well as many system configuration including tracking
technology.

Check out the **[ga_wind_demo.ipynb][GA_WIND_notebook]** notebook for demo.

#### ii. The National Solar Radiation Database
[NSRDB (National Solar Radiation Database)][NSRDB_web] provides 1-hour resolution solar
radiation data, ranging from 1998 to 2016, for the entire U.S. and a growing list of
international locations on a 4x4 square kilometer grid. Data can be accessed via an
[API][NSRDB_api]. Note that the Physical Solar Model v3 is used.

An API key is required to access and use the above databases. Get your own API key
[here][NSRDB_signup].

Here, the power output can be estimated using the previously presented naive method or a
more sophisticated one. The latter uses the System Adviser Model ([SAM][SAM_web])
developed by NREL. The developer tools for creating renewable energy system models can
be downloaded [here][SAM_sdk]. Irradiance data along with other meteorological
parameters must first be retrieved from NSRDB for each site. This information are then
fed to the SAM Simulation Core (SCC) and the power output is retrieved. The SSC reflect
the technology used: photovoltaic (PV), solar water heating and concentrating solar
power (CSP). The [*PVWatts v7*][SAM_pvwatts] model is used for all the solar plants in
the grid. The system size (in DC units) and the array type (fixed open rack,
backtracked, 1-axis and 2-axis) is set for each solar plant whereas a unique value of
1.25 is used for the DC to AC ratio (see article from EIA on inverter loading ratios
[here](https://www.eia.gov/todayinenergy/detail.php?id=35372)). Otherwise, all other
input parameters of the *PVWatts* model are set to their default values. EIA reports in
[form 860][EIA 860] the array type used by each solar PV plant. For each plant in our
network, the array type is a combination of the three technology that is calculated
using the capacity weighted average of the array type of all the plants in
[EIA Form 860][EIA 860] that are located in the same state. If no plants are reported
in [EIA Form 860][EIA 860] for a particular state we then use the information of all
the plants belonging to the same interconnect.

The naive and the SAM methods are used in the
**[nsrdb_naive_demo.ipynb][NSRDB_naive_notebook]** and
**[nsrdb_sam_demo.ipynb][NSRDB_sam_notebook]** demo notebooks, respectively.


### C. Hydro Data

#### i. Hydro v1
EIA (Energy Information Administration) published monthly capacity factors for hydro
plants across the country. This dataset (available [here][hydro_cf]) is used to produce
a profile for each hydro plant in the grid. Note that we are using the same set of
capacity factor independently of the geographical location of the plant. As a result,
the profile of all the hydro plants in the grid will have the same shape. Only the power
output will differ (the scale of the profile).

Check out the **[hydro_v1_demo.ipynb][hydro_v1_notebook]** notebook for demo.

#### ii. Hydro v2
##### a. Western
Hydro hourly profile v2 for Western consists of two components: profile shape to capture
the hourly variance and monthly total generation to capture the historical summation.
EIA published monthly total generation for each hydro plant by state in form EIA 923
(available [here][EIA 923]).

Given that Washington state has the most hydro generation in the Western
Interconnection, we use the aggregated generation of the top 20 US Army Corps managed
hydro dams in Northwestern US (primarily in WA) as the shape of this hydro profile v2
for all the states in Western except for California and Wyoming. The data is obtained
via [US Army Corps of Engineers Northwestern Division DataQuery Tool
2.0][USACE_dataquery].

Observed from system daily outlook of California ISO (available [here][CAISO_outlook]),
the hydro generation profile follows closely to the net demand profile. Thus, we
construct the shape curve of hydro profile v2 in CA using the net demand profile of CA
in the base case scenario (Scenario 87). As for Wyoming, due to the small name plate
capacities of the hydro generators, we simply apply a constant shape curve to avoid
peak-hour violation of maximum capacity.

The final hydro profile v2 is built by scaling the corresponding shape curves based on
the monthly total generation record in each state, then decomposing into plant-level
profiles proportional to the generator capacities. Check out the
**[western_hydro_v2_demo.ipynb][hydro_v2_western_notebook]** notebook for demo.

##### b.Texas
The Electric Reliability Council of Texas (ERCOT) published actual generation by fuel
type for each 15 minute settlement interval. For details, see the fuel mix report
available [here][ERCOT_generation]. Given this time-series historical data, similarly
with the western case, the final hydro profile v2 for Texas is constructed by
decomposing the total hydro generation profile proportional to the generator capacities.
Check out the **[texas_hydro_v2_demo.ipynb][hydro_v2_texas_notebook]** notebook for
demo.

#### iii. Hydro v3
This methodology is designed to generate hourly plant level hydro profile for Eastern,
i.e. eastern hydro v3, which has following features:

- Pumped storage hydro (HPS) and conventional hydro (HYC) are handled separately.
- Historical hourly hydro profiles of four ISOs (ISONE, NYISO, PJM and SPP), are used
directly as regional total hydro profiles to start with.
- For the rest of areas in Eastern Interconnect, which are not covered by the four
ISOs, the hydro profile is generated in the same way as western hydro profile v2.

Note that usa hydro v3 is the concatenation of eastern hydro v3, western hydro v2 and
texas hydro v2

##### a. Pumped Storage Hydro
HPS profile for each plant in [hps_plants_eastern.xlsx][pumped storage hydro model],
**‘all_plantIDs’** sheet, is generated based on the deterministic model presented in
**‘profile’** sheet. The HPS profile model is designed based on local time. The plant
level HPS profiles are generated considering the local time and daylight-saving schedule
of the locations of the corresponding buses. However, during the sanity check
afterwards, it turns out that this model overestimates the HPS output, the final profile
is further scaled by 0.65 to be in agreement with values reported in [EIA 923][EIA 923].

##### b. Conventional Hydro
The buses of the HYC plants were associated to five regions (ISONE, NYISO, PJM, SPP and
the area uncovered by the aforementioned regions) using the same methodology employed
for eastern demand v5. Check Demand Data section below for details. For HYC plants
locating inside the territories of the four ISOs, we decompose the total historical
profile of each ISO into plant -level profiles proportional to the generator capacities.
The time series profile data of each ISO is obtained via their official websites:
[ISONE][ISO New England], [NYISO][NY ISO], [PJM][PJM], [SPP][Southwest Power Pool].

For the rest of HYC plants lying outside the four ISO regions, we apply the same
methodology that is used to generate hydro profile of California in Western hydro v2
(see details in the above subsection). The net demand profile is obtained from Eastern
base case scenario (Scenario 397). The monthly net generations for a state is split
proportional to the total capacities of the generators if the corresponding state is
covered partially by the four ISOs mentioned above.

The final Eastern hydro profile v3 is built by stitching the three parts together:
pumped storage hydro profiles, conventional hydro profiles within the four ISO regions
and conventional hydro profiles in the rest areas . Check out
**[eastern_hydro_v3_demo.ipynb][hydro_v3_eastern_notebook ]** notebook for demo.

### D. Demand Data

*Eastern V6*

The two BAs, SPP and MISO, consist of big areas from north to south. It is unlikely that
the whole area covered by these big BAs have the same hourly profile shape. Eastern
demand v6 addresses the issue by further splitting these two BAs into subareas. We
replace overall MISO and SPP demand with subarea demand obtained directly from contact
at the BAs. We use overall numbers from EIA and use the fractional subarea demand as the
basis to split the MISO and SPP demand.

The hourly subarea demand profiles for SPP and MISO is reported on their official
websites respectively (check [SPP hourly
load](https://marketplace.spp.org/pages/hourly-load), [MISO hourly
load](https://www.misoenergy.org/markets-and-operations/real-time--market-data/market-reports/#nt=%2FMarketReportType%3ADay-Ahead&t=10&p=0&s=MarketReportPublished&sd=desc)).
The geographical definitions of these subareas are obtained directly from contact at the
BAs via either customer service or internal request management system (check
[RMS](https://spprms.issuetrak.com/Login.asp)). Given the geographical definitions, i.e.
list of counties for each subarea, each bus is mapped to the subarea it belongs to.

The overall procedure is similar to v5 except we generate subarea mapping as well as prepare subarea demand from files.

*Eastern V5*

Demand data are obtained from EIA, to whom Balancing Authorities have submitted their
data. The data can be obtained using an API as demonstrated in
**[eastern_demand_v5_demo.ipynb][eastern_demand_v5_demo]** notebook.

Module `get_eia_data` contains functions that converts the data into data frames for
further processing, as performed in
**[eastern_demand_v5_demo.ipynb][eastern_demand_v5_demo]**.

To output the demand profile, cleaning steps were applied to the EIA data for Eastern
V5. We used adjacent demand data to fill missing values using a series of rules:

1. Monday: look forward one day
2. Tuesday - Thursday: average of look forward one day and look back one day
3. Friday: look back one day
4. Saturday: look forward one day
5. Sunday: look back one day

If data is still missing after applying the above rules, week ahead and week behind data
is used:

1. Monday: look forward two days
2. Tuesday: look forward two days
3. Wednesday: average of look forward two days and look back two days
4. Thursday: look back two days
5. Friday: look back two days
6. Saturday - Sun: average of look back one week and look forward one week

If data is still missing after applying the above rules, week ahead and week behind data
is used:

1. Monday - Sunday: average of look back one week and look forward one week

The next step was outlier detection. The underlying physical rationale is that demand
changes are mostly driven by weather temperature changes (first or higher order), and
thermal mass limits the rate at which demand values can change. Outliers were detected
using a z-score threshold value of 3. These outliers are then replaced by linear
interpolation.

The BA counts were then distributed across each region where the BA operates, using the
region populations as weights. For example, if a BA operates in both WA and OR, the
counts for WA are weighted by the fraction of the total counts in WA relative to the
total population of WA and OR.

Lastly, buses were mapped to counties. This step requires an input data file that stores
the list of counties in each BA area territory. Some data cleaning was necessary to deal
with inconsistent county names. We also implemented a check if there are buses where BA
is empty/not found, which is due to bus being outside of United States. These were fixed
manually by assigning the bus to the nearest county.

*Legacy Description*

Demand data are obtained from EIA, to whom Balancing Authorities have submitted their
data. The data can be obtained either by direct download from their database using an
API or by download of Excel spreadsheets. An API key is required for the API download
and this key can be obtained by a user by registering at
<https://www.eia.gov/opendata/>.

The direct download currently contains only published demand data. The Excel
spreadsheets include original and imputed demand data, as well as results of various
data quality checks done by EIA. Documentation about the dataset can be found
[here][demand_doc]. Excel spreadsheets can be downloaded by clicking on the links in
page 9 (Table of all US and foreign connected balancing authorities).

Module `get_eia_data` contains functions that converts the data into data frames for
further processing.

To test EIA download (This requires an EIA API key):
```python
from prereise.gather.demanddata.eia.tests import test_get_eia_data

test_get_eia_data.test_eia_download()
```
To test EIA download from Excel:
```python
from prereise.gather.demanddata.eia.tests import test_get_eia_data

test_get_eia_data.test_from_excel()
```

The **[assemble_ba_from_excel_demo.ipynb][demand_notebook]** notebook illustrates usage.

To output the demand profile, cleaning steps were applied to the EIA data:   1) missing
data imputation - the EIA method was used, i.e., EIA published data was used; beyond
this, NA's were converted to float zeros;   2) missing hours were added.

The BA counts were then distributed across each region where the BA operates, using the
region populations as weights. For example, if a BA operates in both WA and OR, the
counts for WA are weighted by the fraction of the total counts in WA relative to the
total population of WA and OR.

The next step consist in detecting outliers by looking for large changes in the slope of
the demand data. The underlying physical rationale is that demand changes are mostly
driven by weather temperature changes (first or higher order), and thermal mass limits
the rate at which demand values can change. By looking at the slope of demand data, it
is seen that the slope distribution is normally distributed, and outliers can be easily
found by imposing a z-score threshold value of 3. These outliers are then replaced by
linear interpolation.

To test outlier detection, use:
```python
from prereise.gather.demanddata.eia.tests import test_clean_data

test_clean_data.test_slope_interpolate()
```
The **[ba_anomaly_detection_demo.ipynb][demand_anomaly]** notebook illustrates usage.

### E. NREL Electrification Futures Study Demand and Flexibility Data
The National Renewable Energy Laboratory (NREL) has developed the Electrification
Futures Study (EFS) to project and study future sectoral demand changes as a result of
impending widespread electrification. As a part of the EFS, NREL has published multiple
reports (dating back to 2017) that describe their process for projecting demand-side
growth and provide analysis of their preliminary results; all of NREL's published EFS
reports can be found [here][nrel_efs]. Accompanying their reports, NREL has published
data sets that include hourly load profiles that were developed using processes
described in the EFS reports. These hourly load profiles represent state-by-state
end-use electricity demand across four sectors (Transportation, Residential Buildings,
Commercial Buildings, and Industry) for three electrification scenarios (Reference,
Medium, and High) and three levels of technology advancements (Slow, Moderate, and
Rapid). Load profiles are provided for six separate years: 2018, 2020, 2024, 2030, 2040,
and 2050. The base demand data sets and further accompanying information can be found
[here][nrel_base_dem]. In addition to demand growth projections, NREL has also published
data sets that include hourly profiles for flexible load. These data sets indicate the
amount of demand that is considered to be flexible, as determined through the EFS. The
flexibility demand profiles consider two scenarios of flexibility (Base and Enhanced),
in addition to the classifications for each sector, electrification scenario, technology
advancement, and year. The flexible demand data sets and further accompanying
information can be found [here][nrel_flex_dem].

Widespread electrification can have a large impact on future power system planning
decisions and operation. While increased electricity demand can have obvious
implications for generation and transmission capacity planning, new electrified demand
(e.g., electric vehicles, air source heat pumps, and heat pump water heaters) offers
large amounts of potential operational flexibility to grid operators. This flexibility
by demand-side resources can result in demand shifting from times of peak demand to
times of peak renewable generation, presenting the opportunity to defer generation and
transmission capacity upgrades. To help electrification impacts be considered properly,
this package has the ability to access NREL's EFS demand data sets. Currently, users can
access the base demand profiles, allowing the impacts of demand growth due to vast
electrification to be explored. Users can also access the EFS flexible demand profiles,
though integration of demand-side flexibility modeling to the rest of Breakthrough
Energy Sciences' production cost model will be included in a future release.

#### I. Downloading and Extracting EFS Demand and Flexibility Data
The EFS demand data sets are stored on the 'NREL Data Catalog' in .zip files. The 'NREL
Data Catalog' contains nine .zip files, one for each combination of electrification
scenario and technology advancement. The ***.csv*** file compressed within the .zip file
contains the sectoral demand for each state and year. This data can be downloaded and
extracted using the following:
```python
from prereise.gather.demanddata.nrel_efs.get_efs_data import download_demand_data

download_demand_data(es, ta, fpath, sz_path)
```
where `es` is the set of electrification scenarios to be downloaded, `ta` is the set of
technology advancements to be downloaded, `fpath` is the file path to which the NREL EFS
data will be downloaded, and `sz_path` is the file path to which [7-Zip][7zip] is
located for Windows users. `es` and `ta` default to downloading each of the
electrification scenarios and technology advancements, respectively. `fpath` defaults to
downloading the EFS data to the current directory. `sz_path` defaults to the default
installation location for 7-Zip.

The EFS flexibility data sets are also stored on the 'NREL Data Catalog' in .zip files.
The flexibility data is stored differently from the base demand data, with flexibility
data stored in three separate .zip files, one for each electrification scenario. The
***.csv*** file compressed within the .zip file contains the sectoral flexibility for
each state, year, technology advancement, and flexibility scenario. This data can be
downloaded and extracted using the following:
```python
from prereise.gather.demanddata.nrel_efs.get_efs_data import download_flexibility_data

download_flexibility_data(es, fpath, sz_path)
```

where `es` is the set of electrification scenarios to be downloaded, `fpath` is the file
path to which the NREL EFS data will be downloaded, and `sz_path` is the file path to
which 7-Zip is located for Windows users. `es` defaults to downloading each of the
electrification scenarios. `fpath` defaults to downloading the EFS data to the current
directory. `sz_path` defaults to the default installation location for 7-Zip.

Although downloading the .zip files is a simple task, extracting the ***.csv*** files
using Python is more challenging. The .zip files stored on the 'NREL Data Catalog' are
compressed using a compression method (Deflate64, or compression type 9) that is not
currently supported by `zipfile`, a popular Python package for working with .zip files.
To extract the ***.csv*** files in an automated fashion, it is necessary to use either
the Command Prompt or the Terminal, depending on the user's operating system. For
machines running macOS or Linux, the Terminal's extraction tools are capable of
accessing the ***.csv*** file. However, Windows machines are unable to extract the
***.csv*** files using the Command Prompt's extraction tools due to the compression
method. For Windows users with 7-Zip, the ***.csv*** file can be extracted, so long as
7-Zip is installed in the default location. If none of these methods succeed in
extracting the ***.csv*** file, then users can extract the file manually by going to the
.zip file's location and using their extraction tool of choice. For instance, even
though Windows' Command Prompt extraction tools do not work, the native extraction tool
built into Windows' File Explorer does work.

#### II. Splitting the EFS Demand and Flexibility Data by Sector and Year
The EFS demand data for a given electrification scenario and technology advancement is
provided for each sector and each year. Although the sectoral demand is eventually
aggregated for a given year (see next subsection), splitting the demand by sector can be
useful, especially for users that may only want to use a subset of the EFS sectoral
demand (perhaps to pair with their own sectoral demand data). EFS demand data for a
single electrification scenario and technology advancement pair can be split by sector
as follows:
```python
from prereise.gather.demanddata.nrel_efs.get_efs_data import partition_demand_by_sector

sect_dem = partition_demand_by_sector(es, ta, year, sect, fpath, save)
```
where `es` is a string describing a single electrification scenario, `ta` is a string
describing a single technology advancement, `year` is an integer describing the included
year, `sect` is a set of the sectors to include, `fpath` is the file path where the
demand data might be located and to where the sectoral demand will be saved (if
desired), and `save` is a boolean that indicates whether the sectoral demand should be
saved. `sect` defaults to including all the sectors (i.e., all sectoral demand is
retained). `fpath` defaults to the current directory. `save` defaults to `False` (i.e.,
each sectoral demand DataFrame is not saved). `partition_demand_by_sector` returns
`sect_dem`, which is a dictionary of DataFrames, where each DataFrame contains the
demand data for one sector.

The EFS flexibility data for a given electrification scenario is provided for each
sector, year, technology advancement, and flexibility scenario. It is useful to split
the flexibility data by sector for a specific year, technology advancement, and
flexibility scenario. It is useful to split flexibility data by sector because different
sectors may have different operational constraints (e.g., duration between demand
curtailment and recovery, directionality of load shift). EFS flexibility data can be
split by sector as follows:
```python
from prereise.gather.demanddata.nrel_efs.get_efs_data import partition_flexibility_by_sector

sect_dem = partition_flexibility_by_sector(es, ta, flex, year, sect, fpath, save)
```
where `es` is a string describing a single electrification scenario, `ta` is a string
describing a single technology advancement, flex is a string describing a single
flexibility scenario, `year` is an integer describing the included year, `sect` is a set
of the sectors to include, `fpath` is the file path where the flexibility data might be
located and to where the flexibility demand will be saved (if desired), and `save` is a
boolean that indicates whether the sectoral flexibility data should be saved. `sect`
defaults to including all the sectors (i.e., all sectoral flexibility is retained).
`fpath` defaults to the current directory. `save` defaults to `False` (i.e., each
sectoral flexibility DataFrame is not saved). `partition_flexibility_by_sector` returns
`sect_flex`, which is a dictionary of DataFrames, where each DataFrame contains the
flexibility data for one sector.

`partition_demand_by_sector` and `partition_flexibility_by_sector` check the file path
provided by `fpath` to determine if the data is already downloaded and extracted. If the
data is not present as a ***.csv*** file, then `partition_demand_by_sector` and
`partition_flexibility_by_sector` use `download_demand_data` and
`download_flexibility_data`, respectively, to obtain the specified ***.csv*** file. If
the automated approach is not able to extract the ***.csv*** file,
`partition_demand_by_sector` and `partition_flexibility_by_sector` will exit with an
error, and the user will need to manually use their extraction method of choice (as was
discussed in the prior subsection). If manual extraction is required, the user can
simply run `partition_demand_by_sector` and `partition_flexibility_by_sector` again,
making sure that `fpath` points to the appropriate location where the ***.csv*** file is
saved.

#### III. Aggregating the Sectoral Demand Data
For use in the grid model developed by Breakthrough Energy Sciences, all sectoral demand
must be aggregated within each location for each hour (i.e., only one demand data point
per location per hour). Thus, the sectoral demand must be aggregated together to obtain
a single demand value for each location and each hour. Aggregating sectoral demand can
be accomplished as follows:
```python
from prereise.gather.demanddata.nrel_efs.aggregate_demand import combine_efs_demand

agg_dem = combine_efs_demand(efs_dem, non_efs_dem, save)
```
where `efs_dem` is a dictionary of different sectoral demand DataFrames pertaining to
the EFS (this input is intended to be the output of `partition_demand_by_sector`),
`non_efs_dem` is a list of different sectoral demand DataFrames that are independent of
the EFS (this input is intended to be the output of `access_non_efs_demand`, which is
addressed below), and `save` is a string representing the desired file path and file
name to which the aggregated demand will be saved as a ***.csv*** file. Both `efs_dem`
and `non_efs_dem` default to `None`, allowing the user to determine how much sectoral
demand of each type (EFS and non-EFS) they would like to include; note that failing to
specify any sectoral demand will raise an error. `save` defaults to `None`, indicating
that the aggregate demand should not be saved. `combine_efs_demand` returns `agg_dem`,
which is a DataFrame of the aggregate demand data.

The `access_non_efs_demand` function allows sectoral demand that is not associated with
the EFS to be used. `access_non_efs_demand` loads locally-stored sectoral demand data,
checks that it is formatted appropriately, and returns a list of sectoral demand
DataFrames that can be fed into `combine_efs_demand`. Preparing non-EFS sectoral demand
is accomplished as follows:
```python
from prereise.gather.demanddata.nrel_efs.aggregate_demand import access_non_efs_demand

sect_dem = access_non_efs_demand(dem_paths)
```
where `dem_paths` is a list of file paths that point to the ***.csv*** files of
locally-stored sectoral demand data. `access_non_efs_demand` is intended to be used on
sectoral demand data that is not related to the EFS; all sectoral demand data that is
related to the EFS should be handled using `partition_demand_by_sector`. Note that it
will be incumbent upon the user to account for all sectors when building the aggregate
demand profiles (i.e., users will need to ensure that specific sectors are not excluded
or double-counted).

#### IV. Mapping the State Demand to the Appropriate Load Zones
For use in the grid model developed by Breakthrough Energy Sciences, the state-level
demand provided by the EFS must be mapped to the appropriate load zones. Breakthrough
Energy Sciences defines distinct load zones for each state. For smaller states that are
completely contained within a single interconnection, the load zone might very well be
equal to the whole state. However, large states (e.g., California and New York), states
that are in multiple interconnections (e.g., Montana and New Mexico), and states with a
combination of these two characteristics (e.g., Texas) may have multiple load zones.
State-level demand is mapped to the various load zones according to the percentage of a
state's population that resides in a particular load zone. This mapping can be
accomplished as follows:
```python
from prereise.gather.demanddata.nrel_efs.map_states import decompose_demand_profile_by_state_to_loadzone

df_lz = decompose_demand_profile_by_state_to_loadzone(df, save)
```
where `df` is a DataFrame of the hourly demand data for each state in the contiguous
U.S. (intended to be the output of `combine_efs_demand` or the components that are
output from `partition_flexibility_by_sector`) and `save` is a string representing the
desired file path and file name to which the demand will be saved as a ***.csv*** file.
`save` defaults to `None`, indicating that the demand should not be saved.
`decompose_demand_profile_by_state_to_loadzone` returns `df_lz`, which is a DataFrame of
the demand data mapped to each load zone.

#### V. Example of Downloading and Preparing EFS Demand Data
To demonstrate the work flow of these modules, this subsection presents an example of
obtaining the EFS demand data and subsequently preparing it for use in Breakthrough
Energy Sciences' grid model. In this example, EFS demand data under the 'Reference'
electrification scenario and the 'Slow' technology advancement are acquired for the year
2030. After mapping the state-level demand to the different load zones, the aggregate
demand is saved as a ***.csv*** file to the current working directory. This example is
implemented as follows:
```python
from prereise.gather.demanddata.nrel_efs.aggregate_demand import combine_efs_demand
from prereise.gather.demanddata.nrel_efs.get_efs_data import partition_demand_by_sector
from prereise.gather.demanddata.nrel_efs.map_states import decompose_demand_profile_by_state_to_loadzone

sect_dem = partition_demand_by_sector(es="Reference", ta="Slow", year=2030, fpath="")
agg_dem = combine_efs_demand(efs_dem=sect_dem)
agg_dem_lz = decompose_demand_profile_by_state_to_loadzone(df=agg_dem, save="EFS_Demand_Reference_Slow_2030.csv")
```

#### VI. Example of Downloading and Preparing EFS Flexibility Data
This subsection presents an example of obtaining the EFS flexibility data and
subsequently preparing it for use in Breakthrough Energy Sciences' grid model. In this
example, EFS flexibility data under the 'Reference' electrification scenario, 'Slow'
technology advancement, and 'Base' flexibility scenario are acquired for the year 2030.
The state-level sectoral flexibility is mapped to the different load zones, however the
different flexibility data are not saved as ***.csv*** files as was done in the prior
example. This example is implemented as follows:
```python
from prereise.gather.demanddata.nrel_efs.get_efs_data import partition_flexibility_by_sector
from prereise.gather.demanddata.nrel_efs.map_states import decompose_demand_profile_by_state_to_loadzone

sect_flex = partition_flexibility_by_sector(es="Reference", ta="Slow", flex="Base", year=2030, fpath="")
sect_flex_lz = {k: decompose_demand_profile_by_state_to_loadzone(df=v) for k, v in sect_flex.items()}
```

[RAP]: https://www.ncdc.noaa.gov/data-access/model-data/model-datasets/rapid-refresh-rap
[RAP_notebook]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/prereise/gather/winddata/rap/demo/rap_demo.ipynb
[NetCDF]: https://www.unidata.ucar.edu/software/thredds/current/tds/reference/NetcdfSubsetServiceReference.html
[WIND_doc]: https://www.nrel.gov/docs/fy14osti/61714.pdf
[WIND_web]: https://www.nrel.gov/grid/wind-toolkit.html
[WIND_api]: https://developer.nrel.gov/docs/wind/wind-toolkit/
[TE_WIND_notebook]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/prereise/gather/winddata/te_wind/demo/te_wind_demo.ipynb
[NREL_notebooks]: https://github.com/NREL/hsds-examples
[GA_WIND_notebook]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/prereise/gather/solardata/ga_wind/demo/ga_wind_demo.ipynb
[NSRDB_web]: https://nsrdb.nrel.gov/
[NSRDB_api]: https://developer.nrel.gov/docs/solar/nsrdb/
[NSRDB_signup]: https://developer.nrel.gov/signup/
[SAM_web]: https://sam.nrel.gov/
[SAM_sdk]: https://sam.nrel.gov/sdk
[SAM_pvwatts]: https://nrel-pysam.readthedocs.io/en/master/modules/Pvwattsv7.html
[NSRDB_naive_notebook]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/prereise/gather/solardata/nsrdb/demo/nsrdb_naive_demo.ipynb
[NSRDB_sam_notebook]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/prereise/gather/solardata/nsrdb/demo/nsrdb_sam_demo.ipynb
[hydro_cf]: https://www.eia.gov/electricity/annual/html/epa_04_08_b.html
[hydro_v1_notebook]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/prereise/gather/hydrodata/eia/demo/hydro_v1_demo.ipynb
[demand_doc]: https://www.eia.gov/realtime_grid/docs/userguide-knownissues.pdf
[demand_notebook]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/prereise/gather/demanddata/eia/demo/assemble_ba_from_excel_demo.ipynb
[demand_anomaly]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/prereise/gather/demanddata/eia/demo/ba_anomaly_detection_demo.ipynb
[EIA 923]: https://www.eia.gov/electricity/data/eia923/
[CAISO_outlook]: http://www.caiso.com/TodaysOutlook/Pages/default.aspx
[hydro_v2_western_notebook]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/prereise/gather/hydrodata/eia/demo/western_hydro_v2_demo.ipynb
[hydro_v2_texas_notebook]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/prereise/gather/hydrodata/eia/demo/texas_hydro_v2_demo.ipynb
[eastern_demand_v5_demo]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/prereise/gather/demanddata/eia/demo/demo_Eastern_v5/eastern_demand_v5_demo.ipynb
[issue #71]: https://github.com/Breakthrough-Energy/PreREISE/issues/71
[EIA 860]: https://www.eia.gov/electricity/data/eia860/
[ERCOT_generation]: http://www.ercot.com/gridinfo/generation/
[USACE_dataquery]: http://www.nwd-wc.usace.army.mil/dd/common/dataquery/www/
[EIA 930]: https://www.eia.gov/opendata/
[demand_profiles]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/docs/demand_profiles.md
[hydro_profiles]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/docs/hydro_profiles.md
[solar_profiles]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/docs/solar_profiles.md
[wind_profiles]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/docs/wind_profiles.md
[ISO New England]: https://www.iso-ne.com/isoexpress/
[NY ISO]: http://mis.nyiso.com/public/P-63list.htm
[PJM]: http://dataminer2.pjm.com/feed/gen_by_fuel
[Southwest Power Pool]: https://marketplace.spp.org/pages/generation-mix-historical
[hydro_v3_eastern_notebook]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/prereise/gather/hydrodata/eia/demo/eastern_hydro_v3_demo/eastern_hydro_v3_demo.ipynb
[pumped storage hydro model]: https://github.com/Breakthrough-Energy/PreREISE/blob/develop/prereise/gather/hydrodata/eia/demo/eastern_hydro_v3_demo/hps_plants_eastern.xlsx
[nrel_efs]: https://www.nrel.gov/analysis/electrification-futures.html
[nrel_base_dem]: https://data.nrel.gov/submissions/126
[nrel_flex_dem]: https://data.nrel.gov/submissions/127
[7zip]: https://www.7-zip.org/
