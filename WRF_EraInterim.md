# Running WRF real in FRAM



This demonstarate using WRF in FRAM. Because there are already built WRF in FRAM, we don't need to compile from the beginning.

## 1. Some notes about FRAM

FRAM is a Norwegian High Performance Computing (HPC) system. 

Instruction of register an account on FRAM:  https://www.sigma2.no/how-apply-user-account

To run on FRAM, you also need to belong to a project, contact the project leader to be a part of it.

Several WRF models are already installed in FRAM. We will use the `module` command to load it.

Some tips for using modules:

````
module purge #To unload everything incase of some conflicts
module spyder wrf  # to views modules related to wrf
module load <module_name>  #Load a module
module show <module_name>  #to view Path, etc
module list  # show loaded module
module avail # a very looong list of availabel module, use module spyder instead
````

In this step-by-step tutorial, we will use WRF-ARW version 3.9.1, pre-installed in FRAM.

We will need several modules needed for this purpose:

| Nodule name                 | Description                    |
| --------------------------- | ------------------------------ |
| WPS/3.9.1-intel-2016a-dmpar | WRF Preprocessing System (WPS) |
| WRF/3.9.1-intel-2016a-dmpar | WRF model system |
| Boost/1.68.0-intel-2018b-Python-3.6.6 | Python for downloading EraInterim data |



## 2. WRF Preprocessing System (WPS)

### 1.1 Program overview

In `/cluster/software/WPS/3.9.1-intel-2016a-dmpar/WPS`, 

* `geogrid.exe`: to create static data
* `metgrid.exe`: to create WRF input
* `ungrib.exe`: unpack GRIB data into intermediate files
* `link_grib.csh`: to link GRIB files to  the WPS directory  
* `namelist.wps`: namelist for `geogrid.exe`, `ungrib.exe,` and `metgrid.exe`
* `namelist.wps-all_options`: reference to all additional options you can use in  `namelist.wps`


### 1.2 Basic run

<img src="images/wrf_flow.png">



Namelist best practices: https://www2.mmm.ucar.edu/wrf/users/namelist_best_prac_wps.html

#### Run georgid

1. **[Download](https://www2.mmm.ucar.edu/wrf/OnLineTutorial/Basics/GEOGRID/ter_data.php)** the terrestrial input data

2. Edit the **&share** and **&geogrid** sections of the **namelist.wps** file for your particular domain set-up.

3. `./geogrid.exe`

   Tips: Run `ncl util/plotgrids_new.ncl` to view your domain is in the right location **before** running geogrid.exe (The NCL scrpt actually read the namelost.wps to plot the domains)

#### Run Ungrib

Unpacking the data is controlled via the "**share**" and "**ungrib**" sections of the WPS namelist. WPS:

- is NOT dependent on any WRF model domain.
- is NOT dependent on GEOGRID.
- does NOT cut down the data according to your model domain specification. It simply unpacks the required fields and writes them out into a format that the **METGRID** program can read.
- makes use of **Vtables** *(see sample Vtables in the **WPS/ungrib/Variable_Tables/** directory)* to specify which fields to unpack from the GRIB files. The **Vtables** list the fields and their GRIB codes that must be unpacked from the GRIB files.
- Tips: **WPS/util** directory there are two utilities to view GRIB data - **g1print.exe** and **g2print.exe** (for GRIB1 and GRIB2 data). These utilities print a listing of the fields, levels, and dates of the data in the file. The **grib2ctl** tool can be used to display GRIB1 data in [GrADS](http://cola.gmu.edu/grads/)

**Steps:**

1. **Download data and place in a unique directory**

2. **Link** (with the UNIX command ***ln\***) the correct Vtable
     -For example, if you are using GFS data, type:
     `ln -sf ungrib/Variable_Tables/Vtable.GFS Vtable`
     
3. **Link** (with supplied script ***[link_grib.csh](https://www2.mmm.ucar.edu/wrf/OnLineTutorial/Basics/UNGRIB/data_link.php)\***) the input GRIB data
     `./link_grib.csh *path_to_data`*
     
     We can use wildcard to specify files to link, for example `./link_grib.csh <path>/EI_sf_*.grib`
     
4. Edit the **&share** and **&ungrib** sections of the **namelist.wps** file. You only need to pay attention to the following parameters:
      start_date ; end_date ; interval_seconds ; prefix
      **Note**: Normally one will leave the "prefix" set to "FILE", except in cases where this may overwrite data.

5. `./ungrib.exe`

6. **Tips**: `./util/rd_intermediate.exe` print the information of the intermidiate file

7. **Notice:** WRF will need boundary conditions for the ENTIRE time you plan on running the model. Make sure to UNGRIB enough input data.

Input data not in GRIB file: need a DIY program to convert them into intermidiate file.



Run with ERA-Interim data: https://	/2017/12/20/Initializing-the-WRF-model-with-ECMWF-ERA-Interim-reanalysis/



#### Run Metgrid

1. Edit the **&share** and **&metgrid** sections of the namelist.wps file.
2. Input to METGRID is the `geo_em.dxx.nc` output files from GEOGRID, and
     the intermediate output files from UNGRIB (e.g., `FILE:YYYY-MM-DD_hh`).
3. `./metgrid.exe`
4.  Output from this program will be: `met_em.d0X*.YYYY-MM-DD_hh*:00:00.nc` - one file for per time, for each domain ("d0X" represents the domain).

#### Run WRF

1. **Link or copy** the **met_em** files to the run directory.
       ln -sf *path_to_met_em_files*/met_em.d0* 
2. Edit the namelist.input file for your particular run. For descriptions of the namelist parameters, as well as suggestions for best practices, see our [Best Practices WRF Namelist](http://www2.mmm.ucar.edu/wrf/users/namelist_best_prac_wrf.html) page.
3.  **[Run](https://www2.mmm.ucar.edu/wrf/OnLineTutorial/Basics/WRF/run_code.php) real.exe** *([verify](https://www2.mmm.ucar.edu/wrf/OnLineTutorial/Basics/WRF/verify.php) that the program runs correctly)*
       -You should have the following **output files** (*default setup*): **wrfinput_d01** & **wrfbdy_d01**
       -This is true for single domain and default nested runs.
       -If you plan to use a nested domain, you will have a **wrfinput_d\*xx\*** file for each domain.
       (*more on this in 'nested case studies'*).
4.  **[Run](https://www2.mmm.ucar.edu/wrf/OnLineTutorial/Basics/WRF/run_code.php) wrf.exe** *([verify](https://www2.mmm.ucar.edu/wrf/OnLineTutorial/Basics/WRF/verify.php) that the program runs correctly)*
       -You should have the following **output files** (*default setup*): **wrfout_d\*xx\*_[\*initial_date\*]** (*one for each domain*)
       -Each file (*by default*) will contain all the forecast output times.

### Initialize WRF with ERA Interim (0.75)

Note: A higher resolution dataset is ERA5 (0.25), which use **cdsapi** to download. See https://dreambooker.site/2018/04/20/Initializing-the-WRF-model-with-ERA5/

##### Mandatory fields

**3D	Data	(data	on	pressure	levels,	for	example)** 

* Temperature 
* U	and	V	components	of	wind 
* Geopotential Height 
* Relative	Humidity/Specific	Humidity • 

**2D	Data** 

* Surface	pressure 
* Mean	sea-level	pressure 
* Skin	temperature/SST 
* 2	meter	temperature	and	relative	humidity 
* 10	meter	U	and	V	components	of	wind 
* Soil data	(temperature	and	moisture)	and	soil	height •

**Recommended	Fields** 

* LANDSEA	mask	field	for	input	data 
* Water	equivalent	snow	depth	
* SEAICE

**References**

https://www2.mmm.ucar.edu/wrf/OnLineTutorial/Introduction/start.php
