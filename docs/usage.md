# npoint
## Usage
`npoint --set1 listoffiles1.txt --setlabels1 file1.csv --set2 listoffiles2.txt  --setlabels2 file2.csv`
`--covariates covariatefile.csv  --mask mask.nii.gz --model code.R  [ -p N | --sgeN N] --output output`
`--debugfile outputfile `

If a file called `readargs.R` exists that sets a vector called `cmdargs`, this file will be read to obtain the arguments for `npoint` instead of taking them from the command line. This is intended to make it a little easier to remember the long lists of arguments. 

File inputs that are supported are nifti files. _Cifti, and mgz files will be supported in the future.  Alternatively, the user should also be able to supply a CSV file with the data in it, for other types of neuroimaging analysis that might not conform to this model._ The file type is determined simply by the extension (.nii = cifti, .nii.gz = nifti, .mgz is vertex surface). 

`--set1`, `--set2`, ..`--set5`
 By default, the command line variant of the program supports up to 5 input sets. These sets can correspond to longitudinal data points. If more than five input file sets are necessary one should use the programmatic interface, or organize your data as a single input file set in long format with corresponding covariates. Each setfile (`listoffiles1.text ... listoffiles2.txt`) is a text file that contains the filenames in each set, one per row. These sets can correspond to longitudinal data points. They do not have to have the same subjects in each set (i.e., there can be missing data if the models you intend to use support that). **At least one set and corresponding setlabel must be provided **
 
 `--setlabels1`, `--setlabels2`, ... `setlabels5`
 
The setlabel files are csv files that specify variables that correspond to the files in the sets provided above. There must be exactly the same number of setlabel files as sets. If the MRI data files in each set are three dimensional, the list of files in the set  should have exactly the same number of entries as the corresponding label csv file.  If the files are four dimensional (fMRI data), the corresponding label csv files should include one entry per volume (TR). The data from the setlabels must be provided in the same order as the data in the set files. Normally, set label files will include (at the least) an id number and a time point for each 3D file that is specified. For 4D files, set label files will probably include an id number, time point, TR and elements of the fMRI design matrix. The headers of the setlabel files must be consistent across sets, and consistent with headers in the covariate file, if specified, below. **At least one set and setlabel must be provided.**

`--covariates`  Subject identifiers can be associated with any number of covariates as specified in the covariate file, which are csv files. These files can contain additional information about subjects, for example, age at a particular time or IQ. All information in covariate files can also be specified in the setlablels; this option is largely a convenience option. If a covariate file is specified, it will be merged with the content of the setlabel files based on all the header fields that are common to both. An error will occur if there are no header fields in common. 

|Subjectid, |time, |age, |IQ |
|----------|-----|----|---|
|1,         | 1,   |13,  |110|
|2,         |1,    |13.2,|115|
|3,         |1,    |12.9,|98 |


`--mask` The mask must be a file of the same type and size of the first three dimensions  of all the set inputs. The mask must contain 1s and zeros only and computation will be limited to those voxels/vertices which are set to 1 in the mask. (**required**)

`--model` The model specifies the R template code to run the model and return results.  This can also include any initialization code (e.g., libraries that must be included. The model must define the function `processVoxel(v)` that takes as an argument a voxel number `v`. (**required**).

`-p x` The `-p` argument specifies that multicore parallelism will be implemented using `x` processors. An warning is given if the number of processors specified exceeds number of cores. **See notes below on running a model using multicore parallelism.**

`--sgeN N` Alternatively, the `--sge` argument specifies to read the data and divide it into `N` jobs that can be submitted to  the SGE (using a script that is generated called, suggestively, `runme`) or divided among machines by hand and run using GNU make. If SGE parallelism is used, we assume that the directory that the program is called from is read/writeable from all cluster nodes. **See notes below on running a model using SGE parallelism.**

`--output` Specify an output prefix that is prepended to output files. This is useful for organizing output for SGE runs; you can specify something like `--output model-stressXtime/mod1` to organize all the output files and execution scripts into a subdirectory. In addition, the model that you used and the calling arguments will be copied with this prefix so that you can remember what you ran. This is modeled off of how FSL FEAT copies the .fsf file into the FEAT directory (so simple and so handy)! (**required**)

`--debug debugfile` Write out external representations of the design matrix, the fMRI data, and a function called `imagecoordtovertex`, which maps three-dimensional image coordinates (e.g. from fslview) into a vertex number, to the file `debugfile`. This may be useful for development and testing of your model, or troubleshooting problems with the setfiles or covariate files. The debugfile will be prefixed by the output prefix. See the Vignette for instructions for how to use the debugfile.

## Writing the processVoxel function
This function takes a single value `v` which is a numeric index for a voxel. The code should also load any libraries that you need to support your model (e.g., `nlme`). Before calling `processVoxel`, the code will have attached to the design matrix, so that you have access to all of the named columns in this matrix. Note that we attach to minimize memory copies that might be necessary when running in multicore mode.

After you run your model, you should create and return a `list` structure in `R` that contains the values that you want to write out as files. Single scalar values will be combined and reassembled as three-dimensional files, and lists of values (e.g., random effects) will be reassembled as four-dimensional files.

## IMPORTANT: General considerations for running in parallel 
You will be doing the same thing over a lot of voxels in some kind of loop. If your model generates an error, you will lose the entire job. Therefore, you want to be proactive about trapping errors of all types. Specifically, multicore parallelism is a little touchier with respect to error handling than running in a loop, so it is entirely possible that a code that runs correctly for SGE parallelism does not run on a multicore.

Depending on what package you are using in your model and how it is compiled, you may find that the package or underlying libraries themselves are multi-threaded. This will be obvious if you run `top` while executing your model on a multi-core machine. If things are working well, the `R` processes should show up as using up to 100% of the CPU. If your parallelism is fighting with multithreading, you will see your processes using _over_ 100% of the CPU. This is not a good thing! It means that two levels of parallelism are fighting with each other for resources. When they do this, you might find that jobs take many times longer than they should, or worse, never complete.

You will need to figure out how to turn off any underlying parallelism. For the `nlme` library you should set the environment variable `OMP_NUM_THREADS=1` in your shell. Setting this variable in the R script did not work (although it might be that if I do so before loading the `nlme` library it would work). However, other libraries may have other environment variables that should be manipulated.

Practically, I recommend running using SGE parallelism to give
yourself the widest range of opportunities to complete your job using
limited resources. If you do that, you can use `make` on a multicore
machine, you can use any SGE that you like, and it should be pretty
easy to use any other batch scheduler that takes scripts. You can even
copy files to multiple computers and run subsets of the job on
different machines.

## Running a model using SGE parallelism 

The `readargs.R` file in `example.rawfmri` is configured so that it will create a directory called `sgetest` with the assembled design matrix file (in rds format), the split up fMRI data (also in rds format), and files to run the job. These files are:

`Makefile` This file contains the rules for running each subjob and assembling the results. Note that the executables `npointrun` and `npointmerge` must be in your path environment. You can run your job by typing `make -j <ncores>` at the command line in the `sgetest` directory, or by calling the script `runme.local`. You can also type `make mostlyclean` to remove all the intermediate files once your job has completed and you have reassembled your output (by any method). If instead you type `make clean`, you can remove all the rds files also. 

`sgejob.bash` This is the job submission script for processing the data using SGE. Note that `npointrun` needs to be in your path. The commands in the job submission script are bash commands.

`runme.sge` This script will submit the job to the SGE and call Make to merge the resulting files when the job has completed. It is an SGE/Make hybrid.

## Running a model using multicore parallelism

The `readargs.R` file in the `example.flournoy` directory is configured so that it will use 24 cores to compare two models. You should change this number to be lower if your  machine does not have 24 cores. Note that data are not included for `example.flournoy`.

Note that the syntax for trapping errors is a little bit different. We check to see whether the error inherits from `try-error`.

```
# npointrun
Run a model on a single data set sequentially, without data splitting

##Usage
`npointrun --mask mask.nii.gz --model code.R  --designmat designmat.rds`

`--mask mask.nii.gz` Nonzero mask voxels must indicate locations of the data in the corresponding RDS file.  The name of the mask is used to obtain the name of the RDS data file (substituting .nii.gz for .rds). (**required**)

`--model code.R` The model specifies the R template code to run the model and return results. The model must define the function `processVoxel(v)` that takes as an argument a voxel number `v`. (**required**)

`--designmat designmat.rds` The design matrix must be the same dimensions as the voxeldat, read from the RDS file. (**required**)


# npointmerge
Merge some number of files by summing them

## Usage
`npointmerge output <list of files>`
