# Coarse-grained-simulation

## Generate complex pdb file
It's easily to position proteins around a target protein by using the internal gromacs command `gmx insert-molecules`.
a simple command might look like:
```shell
$ insert-molecules -f norrin_fzd4crd.pdb -ci lrp6_bp12.pdb -radius 0.6 -o complex.pdb -nmol 2 -try 10000 -box 10
```
-f indicates the target protein and -ci terms to which protein you wanna to add. then you can declare the vdw distance between the target protein and added protein by the `-radius` tag, in my case which is 0.6 nanometers. And you can also decide how many proteins u want to add around the target protein by `-nmol` tag and `-try` lots of times to ensure that u can successfully insert the proteins in you system. Also the box size must be declare by `-box` tag.

## Soluble protein
Keeping in line with the overall Martini philosophy, the coarse-grained protein model groups 4 heavy atoms together in one coarse-grain bead. Each residue has one backbone bead and zero to four side-chain beads depending on the residue type. The secondary structure of the protein influences both the selected bead types and bond/angle/dihedral parameters of each residue[1]. It is noteworthy that, even though the protein is free to change its tertiary arrangement, local secondary structure is predefined and thus imposed throughout a simulation. Conformational changes that involve changes in the secondary structure are therefore beyond the scope of Martini coarse-grained proteins.

Setting up a coarse-grained protein simulation consists basically of three steps:
- converting an atomistic protein structure into a coarse-grained model;
- generating a suitable Martini topology;
- solvate the protein in the wanted environment.

#### 1. martinize.py
Two first steps are done using the publicly available `martinize.py` script, of which the latest version can be downloaded from [Github](http://md.chem.rug.nl/index.php/tools2/proteins-and-bilayers/204-martinize). 
A common command might look like this:
```shell
$ martinize.py -f ../complex.pdb -o topol.top -x complex_cg.pdb -p backbone -ff elnedyn22 -name chain (-ss [secondary structure file] or -dssp /path/to/dssp)
```
`martinize.py` script will generate coaese grained *pdb or *gro file and position restraints information will be appened to the *itp file(thanks to the -p flag).
#### 2. Short minimization in vacuum
Do a short (ca. 10-100 steps is enough!) minimization in vacuum. Before you can generate the input files with grompp, you will need to check that the topology file (.top) includes the correct martini parameter file (.itp). If this is not the case, change the include statement. Also, you may have to generate a box, specifing the dimensions of the system, for example using `gmx editconf`. You want to make sure, the box dimensions are large enough to avoid close contacts between periodic images of the protein, but also to be considerably larger than twice the cut-off distance used in simulations. Try allowing for a minimum distance of 1 nm from the protein to any box edge. Then, copy the example parameter file, and change the relevant settings to do a minimization run. Now you are ready to do the preprocessing and minimization run:
```shell
$ gmx_mpi editconf -f complex_cg.pdb -d 5 -bt cubic -o complex_cg.gro

$ gmx_mpi grompp -p topol.top -f ../minimization.mdp -c complex_cg.gro -o minimization-complex.tpr

$ gmx_mpi mdrun -deffnm minimization-complex
```
#### 3. Solvate the system and adding counter ions
 Solvate the system with `gmx solvate` (an equilibrated water box can be downloaded [here](http://md.chem.rug.nl/index.php/downloads/example-applications/63-pure-water-solvent); it is called water.gro. Make sure the box size is large enough (i.e. there is enough water around the molecule to avoid periodic boundaries artifacts) and remember to use a larger van der Waals distance when solvating to avoid clashes, e.g.:
 ```shell
 $ gmx_mpi solvate -cp minimization-complex.gro -cs ../water.gro -radius 0.21 -o solv.gro -p topol.top
 ```
 Adding 0.15M NaCl and neutralzing system 
 ```shell
 $ gmx_mpi grompp -f ../minimization.mdp -c solv.gro -p topol.top -o em.tpr
 
 $ gmx_mpi genion -s em.tpr -p topol.top -o solv_ions.gro -pname NA+ -nname Cl- -neutral -conc 0.15
 ```
 - using minimization.mdp as ions.mdp
 check the *top file at end and make sure there are no mistakes in the top file.
 Be careful that the atom name between `top` file and `gro` maybe different, in my case, for example, the some NA and CL names in the `gro` file are "NA" but not "NA+". so we must replace "NA" and "CL" to "NA+" and "CL-" to make it as same as `CG_ions.itp` as well as `top` file.
#### 4. Energy minimization and position-restrained (NPT) equilibration
you will then do a short energy minimization and position-restrained (NPT) equilibration of the solvated system. Since the martinize.py script already generated position restraints (thanks to the `-p` flag), all you have to do is specify `define = -DPOSRES` in your parameter file (`.mdp`). At this point you must also add the appropriate number of water beads to your system topology (`.top`):
```shell
$ gmx_mpi grompp -p topol.top -c solv_ions.gro -f ../minimization.mdp -o em.tpr

$ gmx_mpi mdrun -deffnm em -v
```
