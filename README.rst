.. -*- mode: rst, coding: utf-8 -*-
.. The whole GromacsWrapper package is Copyright (c) 2009-2018 Oliver
.. Beckstein and AUTHORS except where noted otherwise.


========================
 README: GromacsWrapper
========================

|build| |cov| |docs| |zenodo| |black| |PRsWelcome| |anaconda|

A primitive Python wrapper around the Gromacs_ tools. The library is
tested with GROMACS 4.6.5, 2018.x, 2019.x, 2020.x, 2021.x, 2022.x,
2023.x, 2024.x (and 5.x and 2016.x should also work). It supports
Python 3.9--3.13 on Linux and macOS.

GromacsWrapper also provides a small library (cook book) of often-used
recipes and helper functions to set up MD simulations.

`Documentation`_ is mostly provided through the python doc strings and
available at https://gromacswrapper.readthedocs.org for recent releases.

The source code is available in the `GromacsWrapper git repository`_.

Please be aware that this software is only minimally maintained and it
most definitely contains bugs. It is *your* responsibility to ensure
that you are running simulations with sensible parameters.

.. _Gromacs: http://www.gromacs.org
.. _Documentation: 
   https://gromacswrapper.readthedocs.org/en/latest/
.. _GromacsWrapper git repository:
   https://github.com/Becksteinlab/GromacsWrapper
.. |build| image:: https://github.com/Becksteinlab/GromacsWrapper/actions/workflows/ci.yaml/badge.svg?branch=main
   :target: https://github.com/Becksteinlab/GromacsWrapper/actions/workflows/ci.yaml
   :alt: Build Status	     
.. |cov| image:: https://codecov.io/gh/Becksteinlab/GromacsWrapper/graph/badge.svg?token=LvbLZ49wxN
   :target: https://codecov.io/gh/Becksteinlab/GromacsWrapper
   :alt: Code Coverage
.. |zenodo| image:: https://zenodo.org/badge/13219/Becksteinlab/GromacsWrapper.svg
   :target: https://zenodo.org/badge/latestdoi/13219/Becksteinlab/GromacsWrapper
   :alt: Latest release on zenodo (with DOI)
.. |docs| image:: https://readthedocs.org/projects/gromacswrapper/badge/?version=latest
   :target: https://gromacswrapper.readthedocs.org/en/latest/?badge=latest
   :alt: Documentation
.. |PRsWelcome| image:: https://img.shields.io/badge/PRs-welcome-brightgreen.svg
   :target: http://makeapullrequest.com
   :alt: PRs Welcome!	 
.. |anaconda| image:: https://anaconda.org/conda-forge/gromacswrapper/badges/version.svg
   :target: https://anaconda.org/conda-forge/gromacswrapper
   :alt: Anaconda.org package
.. |black| image:: https://img.shields.io/badge/code%20style-black-000000.svg
   :target: https://github.com/psf/black	 
   :alt: black   

	 
	 
Quick Start
===========

Given a PDB file ``1iee.pdb``, set up and run a simple simulation (assuming
you have all other input files at hand such as the MDP files)::

  >>> import gromacs
  >>> print(gromacs.release)
  2018.2
  >>> help(gromacs.pdb2gmx)
  DESCRIPTION

  gmx pdb2gmx reads a .pdb (or .gro) file, reads some database files,
  adds hydrogens to the molecules and generates coordinates in GROMACS
  ...
  ...
  OPTIONS

  Options to specify input files:

  -f      [<.gro/.g96/...>]  (eiwit.pdb)
            Structure file: gro g96 pdb brk ent esp tpr
  ...
  ...
  >>> gromacs.pdb2gmx(f="1iee.pdb", o="protein.gro", p="topol.top",
  ...                 ff="oplsaa", water="tip4p")
  >>> gromacs.editconf(f="protein.gro", o="boxed.gro",
  ...                  bt="dodecahedron", d=1.5, princ=True,
  ...                  input="Protein")
  >>> gromacs.solvate(cp="boxed.gro", cs="tip4p", p="topol.top",
  ...                 o="solvated.gro")
  >>> gromacs.grompp(f="emin.mdp", c="solvated.gro", p="topol.top",
  ...                o="emin.tpr")
  >>> gromacs.mdrun(v=True, deffnm="emin")
  >>> gromacs.grompp(f="md.mdp", c="emin.gro", p="topol.top", o="md.tpr")
  >>> gromacs.mdrun(v=True, deffnm="md")


	 
General liquid presets
======================

``general_nvt.mdp``, ``general_npt.mdp``, ``lincs_nvt.mdp``, and ``lincs_npt.mdp`` are starting presets for homogeneous bulk liquids with finite molecules, three-dimensional periodic boundaries, and no position restraints.
They were written from the `GROMACS 2025.4 MDP reference <https://manual.gromacs.org/documentation/2025.4/user-guide/mdp-options.html>`_, independently of the older bundled templates.
The ``general_`` family uses a 1 fs timestep and ``constraints=none``; the ``lincs_`` family uses a 2 fs timestep and LINCS constraints on covalent bonds involving hydrogen.
Topology-defined constraints, including SETTLE for rigid water, remain active in both families.
All four use leap-frog dynamics, PME electrostatics, 1.2 nm cutoffs, dispersion corrections for energy and pressure, and one v-rescale bath at 300 K.
The default run length is 1 ns in both families (1000000 general steps or 500000 LINCS steps); it is not an equilibration criterion.
Energy/log output is every 0.1 ps and compressed coordinates every 1 ps in both families.
Automatic thermostat/barostat coupling intervals, compressed-coordinate precision, PME accuracy, and LINCS accuracy use GROMACS defaults.
Verlet neighbor searching and van der Waals Cut-off match GROMACS defaults; PME overrides the literal Coulomb Cut-off default.
The 1.2 nm cutoffs and EnerPres dispersion correction are explicit model choices, not GROMACS defaults (1.0 nm and no correction).
NPT additionally uses isotropic C-rescale at 1 bar with a 5 ps coupling time.
Its compressibility of 4.5e-5 bar^-1 is a water-like estimate that should be replaced for the liquid being studied.
Cutoffs, dispersion treatment, constraints, and timestep must agree with the chosen force field; these presets do not define a universal liquid model.

Create a run file by resolving the packaged template and explicitly selecting a different output filename::

    from gromacs import cbook, config

    template = config.get_template("general_nvt.mdp")
    unused = cbook.edit_mdp(
        template,
        new_mdp="nvt.mdp",
        extend_parameters=[],
        ref_t=300,
        gen_temp=300,
        nsteps=1000000,
    )
    if unused:
        raise ValueError(f"Parameters absent from the preset: {unused}")

NVT generates initial velocities and should start from minimized coordinates.
NPT retains velocities: preprocess with ``gmx grompp -f npt.mdp -c nvt.gro -t nvt.cpt -p system.top -o npt.tpr`` after creating ``npt.mdp`` from ``general_npt.mdp``.
To retain state in a subsequent NVT stage, override ``continuation="yes"`` and ``gen_vel="no"`` and supply the preceding checkpoint to ``grompp -t``.
For an exact restart of an existing run, use ``mdrun -cpi`` with its original run input instead of generating fresh velocities.
The fixed ``gen-seed`` and ``ld-seed`` make seed selection explicit; change both for independent fresh replicas, without assuming bitwise reproducibility across hardware.
Use explicit MDP editing and command calls to preserve these choices; high-level ``setup.MD`` helpers apply additional defaults and can modify the template settings.


License
=======

The **GromacsWrapper** package is made available under the terms of
the `GNU Public License v3`_ (or any higher version at your choice)
except as noted below. See the file COPYING for the licensing terms
for all modules.

.. _GNU Public License v3: http://www.gnu.org/licenses/gpl.html


Installation
============

Releases
--------

The `latest version of GromacsWrapper from PyPi`_ can be installed
with ::

  pip install GromacsWrapper


or as a `conda-forge package`_ with ``conda`` from the *conda-forge* channel ::

   conda install -c conda-forge gromacswrapper
    

.. _`latest version of GromacsWrapper from PyPi`:
   https://pypi.org/project/GromacsWrapper/

.. _`conda-forge package`:
   https://anaconda.org/conda-forge/gromacswrapper


Development version
-------------------

The *main* branch in the GitHub source repository generally
contains useful code but nevertheless, things can break in weird and
wonderful ways. Please report issues through the `Issue Tracker`_.

To use the *development code base*:  checkout the ``main`` branch::

   git clone https://github.com/Becksteinlab/GromacsWrapper.git   

and install ::

   pip install GromacsWrapper/

Code contributions are welcome. We use `black`_ for uniform code
formatting so please install black_ and run it on your code.

.. _`black`: https://github.com/psf/black

Download and Availability
=========================

The GromacsWrapper home page is
http://github.com/Becksteinlab/GromacsWrapper.  The latest release of the
package is being made available from https://github.com/Becksteinlab/GromacsWrapper/releases

You can also clone the `GromacsWrapper git repository`_ or fork for
your own development::

  git clone git://github.com/Becksteinlab/GromacsWrapper.git

Questions
=========

Please ask questions in the `Discussion forum`_ (instead of private email).


Reporting Bugs and Contributing to GromacsWrapper
=================================================

Please use the `Issue Tracker`_ to report bugs, installation problems,
and feature requests. Ask questions in the `Discussion forum`_.

**Pull requests** for bug fixes and enhancements are very welcome. See http://makeapullrequest.com for a 
general introduction on how make a pull request and contribute to open source projects.

.. _Issue Tracker: https://github.com/Becksteinlab/GromacsWrapper/issues
.. _Discussion forum: https://github.com/Becksteinlab/GromacsWrapper/discussions


Building Documentation
======================

Install Sphinx::

   pip install sphinx

and compile::

  cd GromacsWrapper
  python setup.py build_sphinx
  

Citing
======

|zenodo|

GromacsWrapper was written by Oliver Beckstein with contributions from
many other people. Please see the file AUTHORS_ for all the names.

If you find this package useful and use it in published work I'd be
grateful if it was acknowledged in text as

  "... used GromacsWrapper (Oliver Beckstein et al,
  https://github.com/Becksteinlab/GromacsWrapper doi: 10.5281/zenodo.17901)"

or in the Acknowledgements section.

Thank you.

.. _AUTHORS:
   https://raw.githubusercontent.com/Becksteinlab/GromacsWrapper/main/AUTHORS

