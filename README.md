# SalomeMeca2024_Code-AsterMPI

Build instruction of SalomeMeca2024 with Code-Aster Testing compiled with MPI PETSC &amp; MUMPS solvers

Basicaly it is just a set of instructions to add Code-Aster Testing MPI solver into Salome-Meca 2024 Singularity / Apptainer image.
It is based on the official Salome-Meca 2024.1 SIF image released by EDF here:  https://code-aster.org/spip.php?article295

See my notes customBuildJC.txt for all details and troubleshooting steps I followed to get there...

Short summary:
--------------

0) cd to a clean build directory.
   
```
mkdir SalomeMeca2024_build
cd SalomeMeca2024_build
```

1) Download sif image from Code-aster.org

```
while true;do
 wget -T 15 -c https://code-aster.org/FICHIERS/singularity/salome_meca-lgpl-2024.1.0-1-20240327-scibian-11.sif && break
done
```

2) create sandbox to edit SIF image content

```
singularity build --sandbox SalomeMeca2024_custom salome_meca-lgpl-2024.1.0-1-20240327-scibian-11.sif  
```

3) enter writable environment within sandbox 

```
singularity shell --writable SalomeMeca2024_custom
```

4) follow guide here: https://gitlab.com/codeaster-opensource-documentation/opensource-installation-development/-/blob/main/devel/compile.md

- in short: clone gitlab src and devtools
   
```
cd /opt/
mkdir codeaster
cd /opt/codeaster
git clone https://gitlab.com/codeaster/src.git
git clone https://gitlab.com/codeaster/devtools.git
```

- build (all prerequisites already in container)

```
cd src
./waf configure
./waf install -j 8
./waf install test -n zzzz506c
```

- check output of test result to make sure that job runned properly with MPI

5) fixes to integrate the MPI version in Salome-Meca / AsterStudy and make it work:

- add custom mpi version in the list of available install

```
echo "vers : testing_mpi:/opt/codeaster/install/mpi/share/aster" >> /opt/salome_meca/V2024.1.0_scibian_univ/tools/Code_aster_frontend-202410/etc/codeaster/aster
```

- fix for wrong as_run version for mpi testing version 

```
mv /usr/local/bin/as_run /usr/local/bin/as_run_23
ln -s /usr/local/bin/as_run /opt/salome_meca/V2024.1.0_scibian_univ/tools/Code_aster_frontend-202410/bin/as_run
```

-  fix run_aster to run in MPI mode under SalomeMeca AsterStudy environment: after many debug steps, I found that the "OMPI_xxxx" variables set when running MPI job in AsterStudy are causing mpiexec to fail... also the wrong version of as_run is used (2023 coming from SalomeMeca) and fails to identify the correct path to the testing_mpi Code-Aster install => need to prepend PATH with /usrl/local/bin to find the right as_run

_MY QUICK & DIRTY PATCH: MANUAL EDIT VERSION_

to fix this, we need to modify run_aster_main.py in /opt/codeaster/install/mpi/lib/aster/run_aster/run_aster_main.py
at line ~476;  comment  

```
proc = run(cmd, shell=True, check=False) 
```
and add the following lines:

```
cmdpfx ="lst=`env | grep OMPI_ | cut -d = -f 1`; for item in $lst; do echo 'unset ' $item; unset $item; done; export PATH=/usr/local/bin:$PATH; "
proc = run(cmdpfx+cmd, shell=True, check=False, capture_output=False)
```
_SAME PATCH, AUTOMATED SED VERSION_

 thanks to ChatGPT, here is a sed command to automate this change (complicated but works, make sure to copy - paste it as one block, newlines matter):

```
cd /opt/codeaster/install/mpi/lib/aster/run_aster
cp run_aster_main.py run_aster_main.orig

sed -i -E '/^[[:space:]]*proc = run\(cmd, shell=True, check=False\)/ {
s/^([[:space:]]*)proc = run\(cmd, shell=True, check=False\)/\1cmdpfx = "lst=`env | grep OMPI_ | cut -d = -f 1`; for item in \$lst; do echo '\''unset '\'' \$item; unset \$item; done; export PATH=\/usr\/local\/bin:\$PATH; "\
\
\1proc = run(cmdpfx+cmd, shell=True, check=False, capture_output=False)/
}' run_aster_main.py
```


6) leave sandbox & build new image

```
exit

singularity build SalomeMeca2024_custom.sif SalomeMeca2024_custom
```

7) delete sandbox files 

```    
rm -rf SalomeMeca2024_custom
```

8) install launcher & run the modified image
```
singularity run --app install SalomeMeca2024_custom.sif 

./SalomeMeca2024_custom
```

## Download image:

Image resulting from this process can be downloaded from the Releases section (here on the right >> )

See release notes for more details.

Please note that you obviously need Singularity container (or maybe Apptainer, not tried but it should be compatible in principle) 

For info this was built using Singularity 3.7.0. on Ubuntu 24.04, but this should not matter much as all tools were built using the container's binaries , compilers, libs etc..

# IMPORTANT NOTE / BUG:
- when running MPI version from AsterStudy, uncheck "save databases" (yellow icon on the right of the job  activation line , starting with the green +). Failure to do so may let the solver run indefinetelly in the background even if calculation is finished. 
If someone knows a fix, let me know ;-)
- The image has been tested on linear static studies up to 1.5MDOFs using MPI PETSC + LDLT_SP and MUMPS solvers up to 20 CPUS. Solvers seem to work well. In case of doubt, run your study with the "stable" or "testing" version to check.


