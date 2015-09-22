Vlasolve is a simple semi-Lagrangian Vlasov-Poisson solver for spherically symmetric 3D problems. It is parallelized with both OpenMP and MPI and implements in particular a 3rd order distributed (MPI) spline interpolation. 

For further information on the numerical method, see `manual/method.pdf` and an application of the code is described in this [scientific article](http://adsabs.harvard.edu/abs/2015MNRAS.450.3724C).

I) Installing
-------------

 - Required programs : cmake
 - Required libraries : None
 - Optional libraries : MPI

Installing is pretty simple, go to the distribution directory `${DIST_PATH}` and type:

    cd build/
    cmake ../ <options>

A report will be displayed with default configuration and indications on how to change options (such as the installation path or path to libraries). Typical options are :

    -DCMAKE_PREFIX_PATH=<PATH> to indicate a default path were to look for libraries
    -DNO_MPI=true to compile without MPI support

Other options are indicated in the report. Problems occuring during configuration that may need fixing are indicated in the report together with a hint on what should be done. If such a problem occurs, you may optionally rerun cmake with different options (it is sometimes advised to clean the cache first):

    rm CMakeCache.txt
    cmake ../ <new options>

Once everything is set, the code is built by running:

    make -j N
    make install

where N is the number of cores to use in parallel.
NOTE: it may sometimes be necessary to clean-up all the files generated by cmake before changing cmake options (e.g. when a library was found but you want to use one from a different location). This can be done by running the `cleanup` script from the distribution directory

    cd ${DIST_PATH}
    ./cleanup


II) Running
--------------
The program parameters can be specified on the command line abd take the format '-group.paramName'. If a parameter is not specified, a default valued is assigned to it (default values assigned to non specified parameters are displayed when program starts).

A list of available parameters :

      -threads_per_node
      number of openMP threads

      -reload [restart file name]
      restart the run from a restart file

      -init.type [UDF,PLDF,PLUMMER,HERNQUIST = UDF]
      The type of initial conditions (uniform density sphere, power law density
      sphere, plummer and hernquist profile). Depending on the type, different
      parameters may be available as init.[parameter name].

       -init.type=UDF: uniform density sphere (see fujiwara (1983) )
         -init.R0 [=2] The uniform sphere radius radius
         -init.M [=1] The total mass
         -init.alpha [=0.5] Initial value of the virial ratio
         -init.smoothingRadius [=0.5] The sphere apodization radius 
         (f(r)=0.5*(1.+erf((init.R0-r)/init.smoothingRadius)))

       -init.type=PLDF: power low density sphere (see Hozumi & al. (2000) )
         -init.M [=1] The total mass
         -init.alpha [=1] velocity anistropy
          defined as alpha=2*Kr/Kt with Kr and Kt the radial and tangential kinetic energy
         -init.eta [=0.1] Initial value of the virial ratio
         -init.n [=1] Index of the power law
         -init.smoothingRadius [=0.5] The sphere apodization radius 
         (f(r)=0.5*(1.+erf((init.R0-r)/init.smoothingRadius)))

    -solver.G [value=1]
    value of G

    -solver.T0 [value=0]
    initial time

    -solver.Tmax [value]
    final time

    -solver.dt [value=0.005]
    time step (constant)

    -solver.nSpecies [value=1]
    number of species (1)

    -solver.statsEvery [value=25]
    How often to compute statistics (e.g. energy, entropy, ...)

    -solver.snapshotEvery [value=500]
    How often to dump a snapshot of the density

    -solver.restartEvery [value=5000]
    How often to dump a restart file (see -reload)

    -grid.Pmin [value=0.01]
    -grid.Pmax [value=25]
    -grid.Pres [value=200] 
    -grid.Pscale [scale type = logarithmic] 
    minimum/maximum values for the radial coordinate and resolution

    -grid.Umin [value=-2 0]
    -grid.Umax [value=2 1.6]
    -grid.Ures [value=200 200]
    -grid.Uscale [scale type = linear quadratic] 
    minimum/maximum values for the velocity/angular momentum coordinates and resolution.
    
e.g. to set a linear scale on radial velocity and quadratic on angular momentum, with velocity ranging from -3 to 3 and angular momentum from 0 to 1.6 :

    vlasolve -grid.Umin -3 0 -grid.Umax 3 1.6 -grid.Uscale linear quadratic


III Output files format
-----------------------
The NDfield binary format is a generic format used to store N-dimensional binary arrays.

For the particular case of Vlasolve, The output files are 3D files containing 'double' floating point data. The 3 dimensions correspond to X, V and J respectively (radial postion, radial velocity and angular momentum). Note that while X and V give the value of the distriubtion function at points whose coordinates are the vertices of the grid, the third dimension represent values of the distribution function integrated over small ranges of J (i.e int [Jk and Jk+1]).

The values of X, V and J corresponding to grid node (i,j,k) are computed as:

    X_i = exp(log(X0) + i* (log(Xmax)-log(X0))/(Ni-1) )
    V_j = V0 + j*(Vmax-V0)/(Nj-1)
    J_k = pow( sqrt(J0)+k*(sqrt(Jmax)-sqrt(J0))/(Nk-1) ,2);
    
where `X0` and `Xmax=X0+deltaX` are the bouding box min and max coordinates along each axis, and i,j and k are integers in `{0,..,Ni}`, `{0,..,Nj}` and  `{0,..,N_k}` respectively. Note that `Nk` is equal to the number of values along the 3rd dimensions plus one !

The binary file format is organized as follows :

**NDfield format:**

|field       |type         |size   |comment                                    |
|:-----------|:-----------:|:-----:|:------------------------------------------|
|dummy       |int(4B)      |1      |for FORTRAN compatibility                  |
|tag         |char(1B)     |16     |identifies the file type. Value : "NDFIELD"|
|dummy       |int(4B)      |1	   |                                           |
|dummy       |int(4B)      |1	   |                                           |
|ndims       |int(4B)      |1      |number of dimensions of the embedding space|
|dims        |int(4B)      |20     |size of the grid in pixels along each dimension, or [ndims,nparticles] if data represents particle coordinates (i.e. fdims_index=1)|
|fdims_index |int(4B)      |1      |0 if data represents a regular grid, 1 if it represents coordinates of tracer particles (always 0 in our case !)|
|datatype    |int(4B)      |1      |type of data stored (see below)            |
|x0          |double(8B)   |20     |origin of bounding box (first ndims val. are meaningfull)|
|delta       |double(8B)   |20     |size of bounding box (first ndims val. are meaningfull)|
|dummy_ext   |char(1B)     |160    |dummy data reserved for future extensions  |
|dummy       |int(4B)      |1	   |                                           |
|dummy       |int(4B)      |1	   |                                           |
|data        |sizeof(type) |N      |data itself (A value for each pixel in out case !)
|dummy       |int(4B)      |1	   |                                           |

with the possible 'datatypes' values being (64 bits system):

|type       |size(B) |numeric_type  |code       |
|-----------|:------:|:------------:|-----------|
|ND_CHAR    | 1      | integer      |1 (=1<<0)  |
|ND_UCHAR   | 1      | integer      |2 (=1<<1)  |
|ND_SHORT   | 2      | integer      |4 (=1<<2)  |
|ND_USHORT  | 2      | integer      |8 (=1<<3)  |
|ND_INT     | 4      | integer      |16 (=1<<4) |
|ND_UINT    | 4      | integer      |32 (=1<<5) |
|ND_LONG    | 8      | integer      |64 (=1<<6) |
|ND_ULONG   | 8      | integer      |128 (=1<<7)|
|ND_FLOAT   | 4      | float        |256 (=1<<8)|
|ND_DOUBLE  | 8      | float        |512 (=1<<9)|

n.b.: blocks are delimited by dummy variables indicating the size of the blocks for FORTRAN compatibility, but they are ignored in C.
