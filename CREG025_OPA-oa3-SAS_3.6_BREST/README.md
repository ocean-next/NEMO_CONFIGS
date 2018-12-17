
## Setting up CREG025 / OPA -- OASIS -- SAS-LIM3 on DATARMOR with NEMO3.6 (LOPS version)

### 1/ Compilation of the 3 components

I used the following modules on DATARMOR:

        intel-comp/18 impi/2018.1.163 NETCDF-test/4.3.3.1-impi-intel2018


#### 1.1 Compilation of OASIS

OASIS needs to be compiled first because both XIOS and NENO depend on it.

    svn co https://oasis3mct.cerfacs.fr/svn/branches/OASIS3-MCT_3.0_branch/oasis3-mct

    cd oasis3-mct/util/make_dir

Use the ```make.DATARMOR``` provided here into ```compilation/oasis3-mct/```
    
    make -f TopMakefileOasis3

If everything goes according to plan you should find these in ```oasis3-mct/lib/```:
    
    libmct.a  libmpeu.a  libpsmile.MPI1.a  libscrip.a

*My compiled OASIS:*

        datarmor:/home3/datahome/lbrodeau/DEV/oasis3-mct
    

***
***

#### 1.2 Compilation of XIOS

I use up-to-date version (r1619) of ```xios-2.0``` (100% compatible with XML files of Claude's 3.6 setup):

    svn co http://forge.ipsl.jussieu.fr/ioserver/svn/XIOS/branchs/xios-2.0 xios-2.0_oa3

Copy the 3 DATARMOR-architecture files provided here in ```compilation/xios-2.0``` into ```arch/``` 
You can compile, just mind the extra oasis-related flag in our particular case:

    ./make_xios --arch DATARMOR --prod --full --use_oasis oasis3_mct --netcdf_lib netcdf4_seq --job 6


*My compiled XIOS:*

        datarmor:/home3/datahome/lbrodeau/DEV/xios-2.0_oa3


***
***

#### 1.3 Compilation of NEMO opa.exe and sas.exe

Now, it's getting trickier since we won't compile a single ```nemo.exe``` out of NEMO but one ```nemo.exe``` that we shall rename ```opa.exe``` and another ```nemo.exe``` that we shall rename ```sas.exe```.

The NEMOGCM version I used is the one provided by Claude, it is located here:

    datarmor:/home1/datahome/ctalandi/DEV/NEMODRAK/NEMODRAK_3.6_STABLE_HEAD/NEMOREF/NEMOGCM

The exact same version is also backed-up in my home as ```3.6r7088``` here:

    datarmor:/home3/datahome/lbrodeau/DEV/NEMOGCM_3.6r7088

As usual create and adapt your own ```NEMOGCM/ARCH/arch-YOUR_ARCH.fcm```, mind this
time to complete the ```OASIS``` and ```XIOS``` paths according to your new OASIS and
XIOS install...
The relevant ARCH file for DATARMOR is provided here: ```compilation/NEMOGCM/arch-DATARMOR_OA3.fcm```

Now, in your ```NEMOGCM/CONFIG/``` directory,  add everything you find here in ```sources/CONFIG/```, namely: 

    cfg.txt CREG025_OPA CREG025_SAS_LIM3

 ```cfg.txt``` contains:
    
    CREG025_OPA OPA_SRC
    CREG025_SAS_LIM3 OPA_SRC SAS_SRC LIM_SRC_3

What we shall compile into ```CREG025_OPA``` and ```CREG025_SAS_LIM3``` will eventually become ```opa.exe``` and ```sas.exe```, respectively.


**IMPORTANT: the ```MY_SRC``` directories found under both ```CREG025_OPA``` and ```CREG025_SAS_LIM3``` contains the "LOPS version" source modifications with respect to the reference NEMOGCM they used plus the "OASIS-SAS" modifications that I merged in as well!**
Same for the ```CPP key``` file in each directory. Having a look at these ```CPP key``` files might give you a hint on what we really
do... Like for instance, ```key_lim3``` is obviously only used for the ```CREG025_SAS_LIM3``` config.

So now you just have to compile two ```nemo.exe``` and rename them accordingly:

    ./makenemo -m DATARMOR_OA3 -n CREG025_OPA -j 8
    mv -f CREG025_OPA/BLD/bin/nemo.exe CREG025_OPA/BLD/bin/opa.exe

    ./makenemo -m DATARMOR_OA3 -n CREG025_SAS_LIM3 -j 8
    mv -f CREG025_SAS_LIM3/BLD/bin/nemo.exe CREG025_SAS_LIM3/BLD/bin/sas.exe

*My setup and compiled executables are there:*

        datarmor:/home3/datahome/lbrodeau/NEMO/NEMOv3.6r7088_OA3-SAS/CONFIG




***
We're done with compiling stuffs!
***
***

### 2/ Preparing and launching a simulation


In your run directory, before launching the monster you need to have:

* The 3 executables (see 1): opa.exe, sas.exe, xios_server.exe

* All appropriate NetCDF files (see 2.1)

* The control ASCII files (see 2.2)

***
***

#### 2.1 Setup and forcing NetCDF files

All the netcdf files (mainly those I got from Claude) are in there:

    datarmor:/home3/datawork/lbrodeau/CREG025/CREG025-I

Including the 3 following sub-directories:
- ```BDY``` contains the boundary conditions
- ```FATM``` contains the atmospheric forcing and associated remapping weights
- ```OASIS``` contains the initial ocean and ice states needed for OASIS (relevance of fields in them has no importance)

***
***


#### 2.2 Namelists and XIOS xml files

The appropriate namelists for opa.exe, sas.exe and OASIS, respectively; as well as all the XML files for XIOS are to be found here in: ```namelists/```:

     iodef.xml [general]
     namelist_cfg, namelist_ref, domain_def_opa.xml, field_def_opa.xml, file_def_opa.xml [OPA]
     namelist_sas_cfg, namelist_sas_ref, namelist_ice_cfg, namelist_ice_ref, domain_def_sas.xml, field_def_sas.xml, file_def_sas.xml [SAS]
     namcouple [OASIS]




***
***

#### 2.3 Launching the beast

The "cheap" setup I tested on DATARMOR (28 cores in total = 1 full node):

    mpiexec.hydra -n 1 ./xios_server.exe : -n 15 ./opa.exe : -n 12 ./sas.exe

Of course mind the cpu decomposition according to the number of cores you use
for each component.

In this particular case, for ```opa.exe```, in ```namelist_cfg```, I have:

    &nammpp
    jpni=3
    jpnj=5
    jpnij=15
    /

For ```sas.exe```, in ```namelist_sas_cfg```, I have:

    &nammpp
    jpni=3
    jpnj=4
    jpnij=12
    /


*Run directory on DATARMOR where the successful execution of 5 model days was performed:*

    dartarmor:/home3/scratch/lbrodeau/tmp/CREG025_SASOA3/CREG025_SASOA3-ILBOOSCT_prod


***
***
***

/laurent
