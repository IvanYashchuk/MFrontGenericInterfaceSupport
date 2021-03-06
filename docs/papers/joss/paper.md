---
title: The `MFrontGenericInterfaceSupport` project
tags:
  - MFront
  - Mechanical solvers
  - Mechanical behaviours
authors:
  - name: Thomas Helfer
    orcid: 0000-0003-2460-5816
    affiliation: 1
  - name: Jeremy Bleyer
    orcid: 0000-0001-8212-9921
    affiliation: 2
  - name: Tero Frondelius
    orcid: 0000-0003-2288-0902
    affiliation: "3, 4"
  - name: Ivan Yashchuk
    affiliation: "5, 6"
  - name: Thomas Nagel
    orcid: 0000-0001-8459-4616
    affiliation: 7
  - name: Dmitri Naumov
    affiliation: 8
affiliations:
  - name: French Alternative Energies and Atomic Energy Commission
    index: 1
  - name: Laboratoire Navier UMR 8205 (École des Ponts ParisTech-IFSTTAR-CNRS)
    index: 2
  - name: Wärtsilä
    index: 3
  - name: University of Oulu
    index: 4
  - name: VTT Technical Research Centre of Finland
    index: 5
  - name: Aalto University
    index: 6
  - name: Technische Universität Bergakademie Freiberg
    index: 7
  - name: Helmholtz Centre for Environmental Research -- UFZ
    index: 8
date: 8 October 2019
bibliography: bibliography.bib
---

<!--
pandoc -f markdown_strict --bibliography=bibliography.bib --filter pandoc-citeproc paper.md -o paper.pdf
-->

# Introduction

Constitutive equations describe how the internal state variables of a
material evolve with changing external conditions or mechanical
loadings. Those state variables can describe many microstructural
aspects of the material (grain size, dislocation density, hardening
state, etc.) or be phenomenological in nature (equivalent plastic
strain). The knowledge of those internal state variables allows the
computation of local thermodynamic forces which affect the material
equilibrium at the structural scale.

At each time step, the constitutive equations must be integrated to
obtain the state of the material at the end of the time step. As most
phenomena are nonlinear, an iterative scheme is required at the
equilibrium scale to find the local loading of the material: the
integration of the constitutive equations is thus called several times
with different estimates of the loading of the material.

Due to the large number of phenomena described (plasticity,
viscoplasticity, damage, etc.), computational mechanics is one of the
most demanding domains for advanced constitutive equations.

The ability to easily integrate user-defined constitutive equations
plays a major role in the versatility of (mechanical) solvers^[The term
solver emphasizes that the numerical method used to discretize the
equilibrium equations is not significant.].

The `MFront` open-source code generator has been designed to simplify
the implementation of the integration of the constitutive equations over
a time step [@helfer_introducing_2015;@cea_mfront_2019].

From a source file, `MFront` generates `C++` code specific to many
well-established (mostly thermo-mechanical) solvers through dedicated
interfaces and compiles them into shared libraries. For example,
`MFront` provides interfaces for `Cast3M`, `code_aster`, `Europlexus`,
`Abaqus/Standard`, `Abaqus/Explicit`, `CalculiX`, etc.

In the following, we use the term "behaviour" to denote the result of
the implementation and compilation of the constitutive equations.

`MFront` recently introduced a so-called `generic` interface. This paper
describes the `MFrontGenericInterfaceSupport` project, which is denoted
`MGIS` in the following. `MGIS` aims at proving tools (functions,
classes, bindings to various programming languages) to handle behaviours
generated using `MFront`' `generic` interface [@helfer_mgis_2019]. Those
tools alleviate the work required by solvers' developers. Permissive
licences have been chosen to allow integration in open-source and
proprietary codes.

This paper is divided into three parts:

1. Section 1 gives a brief overview of `MGIS`.
2. Section 2 describes the various bindings available.
3. Section 3 describes some examples of usage in various open-source
  solvers: `FEniCS`, `OpenGeoSys` and `JuliaFEM`.

# Overview

The aims of the `MFrontGenericInterfaceSupport` project are twofold:

1. At the pre-processing state, allow retrieving metadata about a
  particular behaviour and perform proper memory allocation. At the
  post-processing stage, ease access to internal state variables.
2. During computations, simplify the integration of the behaviour at
  integration points^[The term "integration points" is used here as a
  generic placeholder. When using FFT for solving the equilibrium
  equations, the integration points are voxels. When using FEM, the
  integrations points are the usual Gauss points of the elements.] and
  the update of the internal state variables from one time step to the
  other.

## Preprocessing and post-processing stages{#sec:prepost}

When dealing with user defined behaviours, most solvers, including
`Abaqus/Standard` for example, deleguates part of the work to the
user. The user must:

1. describe the behaviour in the input
2. take care of the consistency of the behaviour with the hypothesis
  made during the computation (e.g. a finite strain behaviour must be
  used in a finite strain analysis based on the appropriate deformation
  and stress measures as well as reference configurations).

This is error-prone and may lead to spurious or even worse inexact
results.

`MGIS` introduces a very different approach: the user only declares the
shared library, the behaviour and the modelling hypothesis
(tridimensional, plane strain, etc.). With this information, the library
retrieves various metadata which fully describe how to interact with the
behaviour. The solver using `MGIS` can then check if the behaviour is
consistent with the computations to be performed and checks that the
data provided by the user are correct.

The metadata can also be used to allocate the memory required to store
the state of the material at each integration point. `MGIS`' design
allows the following types of storage:

- An `MGIS` data structure per integration point. While this causes
  memory fragmentation, this is the most frequent choice. The memory is
  automatically allocated by `MGIS`.
- An `MGIS` data structure that stores the states of an arbitrary number
  of integration points. `MGIS` can allocate the memory associated with
  the state of all specified integrations points or borrow memory
  allocated by the solver.

For post-processing, `MGIS` provides a set of functions to retrieve
information about the state of the material. For example, one can
retrieve the value of a state variable from the previous data
structures.

## Computation stage

`MGIS` provides a function to integrate the constitutive equations at
one integration point or on a set of integration points^[This strongly
depends on the data structure chosen to store the internal state
variables.].

The integration of the constitutive equations at different integration
points are usually independent: thus, when handling a set of integration
points, `MGIS` can parallelize the integrations using a granularity
chosen by the solver.

# Main language and available bindings

`MGIS` is written in `C++-11`. The `C++` API is described in another
report, see [@helfer_brief_2019].

The following bindings are available:

- `python`.
- `Julia`.
- `Fortran 2003`.
- `C`.

# Examples of usage

## `FEniCS`

!["Figure 1: Large strain elasto-plastic modelling of a notched
bar"](img/FEniCS.png "Large strain elasto-plastic modelling of a notched
bar")

`FEniCS` is a popular open-source computing platform for solving partial
differential equations [@logg_automated_2012;@alnaes_fenics_2015].

Non linear mechanics computations combining `FEniCS` at the equilibrium
scale and `MFront` to describe the constitutive equations can be
performed through the `python` bindings of `MGIS` as demonstrated by
Bleyer et al. (see [@bleyer_elasto-plastic_2019;@bleyer_fenics_2019]).

Extensions to finite strain elastoplasticity have been recently added as
shown in Figure 1 which models a tensile test on a notched bar^[This
case is adapted from a non-regression test of `Code_Aster` finite
element solver, see @edf_ssna303_2011 for details].

## `OpenGeoSys`

OpenGeoSys (OGS) is a scientific open-source initiative for the
numerical simulation of thermo-hydro-mechanical/ chemical (THMC)
processes in porous and fractured media, inspired by FEFLOW and ROCKFLOW
concepts and continuously developed since the mid-eighties, see
([@Kolditz:1990;@Wollrath:1990;@Kroehn:1991;@Helmig:1993;@kolditz_opengeosys:_2012;@Bilke2019]).

The OGS framework is targeting applications in environmental geoscience,
e.g., in the fields of contaminant hydrology, water resources and waste management,
geotechnical applications, geothermal energy systems and energy
storage.

The most recent version, `OpenGeoSys-6` (`OGS-6`)
([@Naumov:2018;@Bilke2019]), is a fundamental re-implementation of the
multi-physics code `OpenGeoSys-4/5` ([@Kolditz2004225;@Wang:2006]) using
advanced methods in software engineering and architecture with a focus
on code quality, modularity, performance and comprehensive
documentation.

Among its recent extensions are the implementation of numerical methods
for the propagation of discontinuities, such as enriched finite element
function spaces, non-local formulations and phase-field models for
fracture ([@Watanabe2012;@Parisio2018;@Yoshioka2019]).

To simplify the implementation of new constitutive models for solid
phases developed with `MFront`, `OGS-6` relies on `C` bindings of
`MGIS`.

!["Figure 2: Elasto-plastic modelling of a cyclically loaded cavity in a cohesive-frictional material (left). Shear bands forming beneath an  applied traction load on a frictional material."](img/MCAS_disc_hole_cyclic_show_axes.png "Elasto-plastic modelling of a cyclically loaded cavity in a cohesive-frictional material.")

Figure 2 shows the results of a test simulation of a cavity in a
cohesive-frictional material modelled by a non-associated plastic
behaviour based on the Mohr-Coulomb yield criterion and subjected to a
cyclically varying anisotropic stress field (see @Nagel2016 for a
complete description and verification against an analytical solution
in the isotropic case). The right hand side of Figure 2 shows a compressive
load applied to a non-associated frictional material in the presence of
gravitational loading.



<!--
Current tests include elastic (isotropic and anisotropic),
elasto-plastic (see Fig. 2) and visco-plastic materials.
-->

## `JuliaFEM`

`JuliaFEM`
[@frondelius_juliafem_2017;@rapo_natural_2017;@rapo_implementing_2018;@aho_introduction_2019;@rapo_pipe_2019;@aho_juliafem_2019]
is an open-source finite element solver written in the Julia programming
language [@bezanson_julia:_2017]. JuliaFEM enables flexible simulation
models, takes advantage of the scripting language interface, which is
easy to learn and embrace. Besides, it is a real programming environment
where other analyses and workflows combine with simulation.

!["Figure 3: Block diagram showing the software layers involved in using
`MFront` behaviours in `JuliaFEM`"](img/MFrontInterface.png "Software
layers.")

The `MFrontInterface.jl` [@frondelius_mfrontinterface_2019] is a `Julia`
package where `MFront` material models are brought to `Julia` via wrapping
`MGIS`, see Fig. 3. Installation is, as easy as any julia packages,
i.e., `pkg> add MFrontInterface`. For example `TFEL` and `MGIS`
cross-compiled binary dependencies are automatically downloaded and
extracted. Lastly, Fig. 4. shows a simple 3D geometry example using
JuliaFEM and MFrontInterface together.

!["Figure 4: Simple isotropic plasticity modelling of a 3D beam in
JuliaFEM with MFrontInterface."](img/3dbeam_mfront.png "Simple JuliaFEM
plus MFrontInterface 3D demo")

# Conclusions

This paper introduces the `MFrontGenericInterfaceSupport` library which
considerably eases the integration of `MFront` generated behaviours in
any solver. In particular, the library provides a way of retrieving the
metadata associated with a behaviour, data structures to store the
physical information, functions to perform the behaviour integration
over a time step. Examples of usage in various open-source solvers
(`FEniCS`, `OpenGeoSys`, `JuliaFEM`) have been provided.

# Acknowledgements

This research was conducted in the framework of the `PLEIADES` project,
which is supported financially by the CEA (Commissariat à l’Energie
Atomique et aux Energies Alternatives), EDF (Electricité de France) and
Framatome.Acknowledgements

We would like to express our thanks to Olaf Kolditz and the entire
community of developers and users of OpenGeoSys(OGS). We thank the
Helmholtz Centre for Environmental Research -- UFZ for long-term funding
and continuous support of the OpenGeoSys initiative. OGS has been
supported by various projects funded by Federal Ministries (BMBF, BMWi)
as well as the German Research Foundation (DFG). We further thank the
Federal Institute for Geosciences and Natural Resources (BGR) for
funding.

Also, we would like to acknowledge the financial support of Business
Finland for both ISA Wärtsilä Dnro 7734/31/2018, and ISA VTT Dnro
7980/31/2018 projects.

This project uses code extracted from the following projects:

- https://github.com/bitwizeshift/string_view-standalone by Matthew
  Rodusek
- https://github.com/mpark/variant: by Michael Park
- https://github.com/progschj/ThreadPool by Jakob Progsch and Václav
  Zeman
- https://github.com/martinmoene/span-lite by Martin Moene
- https://bitbucket.org/fenics-apps/fenics-solid-mechanics/ by
  Kristian B. Ølgaard and Garth N. Wells.

# References
