# <span id="anchor"></span>Synthetic Data Project - Findings

Author: Alessandra Polimeno  
Research and development: Paul Boon & Alessandra Polimeno  
Date: 20/10/2026

## Table of Contents
- [Task description](#anchor)
- [Phase 1: Metasyn and Dataverse](#anchor)
- [Phase 2: Solutions for Synthetic Data Creation](#anchor)
- [Phase 3: Demo Implementation](#anchor)


## <span id="anchor"></span>Task description

This report summarizes the work done by DANS in the [TDCC-SSH Project
Synthetic Data: Leveraging the potential of sensitive data in
SSH](https://tdcc-synthetic-data.nl/) research Work Package 2 Task 1:
Synthetic Data Tools. This task investigates what is needed for tools
that generate synthetic data to be integrated into the [Data Station
SSH](https://ssh.datastations.nl/). We focused on the
[Metasyn](https://github.com/sodascience/metasyn) tool, which provides a
way to generate synthetic data of tabular datasets with Python while
ensuring a high privacy protection of the original data.

A common use case of synthetic data is concerned with transforming a
restricted tabular data file into a synthetic version that can be openly
shared and used by others for, among other things, educational purposes,
as test data, to check the reproducibility of code, or to start
developing a program while access to the full data is pending. The
scenario we envision allows a data owner to create synthetic datasets
from their original dataset within the depositing page through the
Dataverse External Tools module. The synthetic version can then be
published openly at DANS while the original dataset may be restricted or
unpublished. We do not cover the scenario in which a user would want to
create or request a synthetic dataset of an existing restricted dataset.
This scenario will further be investigated in another part of the WP.

The task was carried out in three phases. The first phase focused on the
question of whether the Dataverse API provides enough metadata to
generate a synthetic version with Metasyn without access to the dataset
itself. The second phase took these findings and proposes potential
solutions that allow depositors to create a synthetic dataset without
the need to adapt or create new Dataverse functionalities. In the third phase a demo was developed that showcases a graphical user interface that allows users to create synthetic data from the dataset landing page in Dataverse.   

## <span id="anchor"></span>Phase 1: Metasyn and Dataverse

### <span id="anchor"></span>Metasyn description

[Metasyn](https://github.com/sodascience/metasyn?tab=readme-ov-file) is
an open source, privacy-friendly generator for synthetic tabular data.
The tool fits a model to an existing dataset, capturing the datatypes
and distributions of individual variables. The user can save the model
and use it to generate synthetic data that looks like the original
datafile. The statistical information is strictly limited (e.g. not
capturing relations between variables) so that it avoids privacy leaks.

To create a synthetic dataset with Metasyn, two components are required:
the original dataset and the corresponding schema that at least
specifies the variable names and their data types.

From the dataset and schema information, Metasyn extracts statistical
metadata used for the creation of the synthetic dataset.

This information is saved to a standard called [Generative Metadata
Format](https://github.com/sodascience/generative_metadata_format) (GMF),
which stores the statistical metadata in JSON. A GMF is modular and
extensible, allowing for the addition of more distributions and privacy
enhancing mechanisms by the user. With the GMF, the user can create
synthetic versions of the data without needing access to the original
dataset.

Users can create synthetic data with Metasyn with a Python script or
notebook. Both the GMF and synthetic data file can be saved for reuse.

### <span id="anchor"></span>Dataverse ingest details

The biggest challenge for creating synthetic datasets from the data
currently in DANS is that the data files that are most interesting to
have synthetic data for are restricted access and can hence only be
viewed and used by the original depositor or anyone who has access. This
limits the (automatic) processing of the data in a tool like metasyn.
However, it should also be possible to generate synthetic data if the
original dataset is not available, as long as there is enough schema
information in the file metadata to create a GMF file. A first step we
took was to assess whether Dataverse provides enough metadata for such a
GMF file. We found that while the Dataverse API allows for the
extraction of file metadata, it does not provide enough information to
populate a GMF. The missing information is mostly concerned with the
distribution types of the data. See [this
report](https://github.com/tdcc-synthetic-data/synthetic_metadata_interop/blob/main/research/dataverse_findings.md#summary-and-implications)
for an overview of the missing information.

We investigated whether the Dataverse Ingest Function could be extended
to provide the necessary information during depositing. See [this
report](https://github.com/tdcc-synthetic-data/synthetic_metadata_interop/blob/main/research/dataverse_ingest_details.md)
for the full description of this process. We found that extending the
ingest function would technically be possible, but there are some issues
that make it undesirable:

-  Calculating the distributions at ingest may take too much time to be
  acceptable for a depositor to wait for.
- The code to calculate the distribution information would need to
  somehow be ported, as we cannot easily make changes to Dataverse.
- Large files do not currently have an ingest configuration in the SSH
  Data Station which would thus still require an alternative solution

Due to these issues, we decided it was not within scope of the current
project to follow the route of extending the Dataverse ingest function.
The next section describes the alternative we propose instead.

## <span id="anchor"></span>Phase 2: Dataverse External Tool Functionality and a Metasyn API

This phase of the task explores how we can use the External Tools
functionality within Dataverse to provide the generation of synthetic
data of a tabular file. It also includes a prototype for a locally run
Metasyn API that can be called as external tool.

### <span id="anchor-1"></span>DV External Tools

The Dataverse [External
Tools](https://guides.dataverse.org/en/latest/api/external-tools.html)
functionality allows for the configuration of external tools that can be
called from within Dataverse. It initiates a request to another service
outside of Dataverse during depositing or editing a dataset and allows
for saving the output of the external tool to the dataset as an
auxiliary file. See [this
report](https://github.com/tdcc-synthetic-data/synthetic_metadata_interop/tree/main/dataverse_external_tools)
for details on its setup and use.

The External Tools functionality can, when properly configured, be used
by navigating to the file you want to manipulate, clicking *Access File*
on the download button of that file, and choosing a tool listed under
*Explore Options* in the drop-down menu. The browser will then load an
html page containing the external tool.

In this window, the original tabular file can be used as long as the
user calling the tool has access to the file (so it does not work for
users wanting to work with restricted files they have not received
access to). Any output generated by the external tool can be saved as an
auxiliary file, which can be downloaded via the same *Access File*
dropdown menu, and either saved to the user’s disk or uploaded to the
dataset.

During this experiment, one limitation of the External Tools
functionality was discovered, namely that an auxiliary file can
currently only be created for restricted files. For the current use
case, this is unlikely to pose problems because synthetic data is most
often generated from restricted data, but it may be a good idea to have
the issue fixed to avoid later bugs.

For this approach to work with Metasyn, we would need to turn it into a
service that can be called within the External Tools menu. One way to do
this would be to create an API that allows for the creation of a GMF
file and/or the resulting synthetic data. The next section describes the
first attempt at developing such an API.

### <span id="anchor-1"></span>Metasyn API

We created a simple fastAPI wrapper for the Metasyn functionalities of
creating and downloading a GMF and generating synthetic data with the
GMF. Both functionalities have their own endpoint which can be called
separately. See [the github
repo](https://github.com/tdcc-synthetic-data/synthetic_metadata_interop/tree/main/metasyn_api)
for more details and instructions on installing and using the API.

The API can currently only be run locally and does not include any
authentication or security features. It is intended as a starting point
for further development, and for testing if it could work in tandem with
the External Tools functionality.

### <span id="anchor-1"></span>Discussion and next steps 

The steps taken above by the WP2 team of DANS were discussed with the
Metasyn team from Utrecht.

In this discussion, it became clear that the development of a
production-ready API service that automates the generation of the GMF
and the creation of the synthetic data file was not a planned feature
and will require more capacity than provided in the current project.

Another conclusion from this discussion was that the creation of a GMF
should be done by the depositor and that the GMF should be manually
checked to ensure it represents the data well and does not contain any
information that should not be distributed. Rather than automating this
process, the project team will invest in creating a detailed handbook
for depositors to create a GMF from the original dataset.

This GMF can then be uploaded to the dataset in the Data Station as an
open access file.

A next step to explore is whether this GMF file can then be transformed
into synthetic data using the external tool functionality. For this, the
external tool functionality would need to be working for open access
files. In addition, DANS would need to run Metasyn for the GMF to
synthetic data conversion which would require more would require more
development work on the API, for instance by making it more secure so
that it can be openly accessible. Since the input data would be openly
accessible, the requirements should however be less complex.

## <span id="anchor"></span>Phase 3: Integrating the Metasyn API into Dataverse
The goal of this phase is to demonstrate a possible integration of the Metasyn API we created in the previous phase into Dataverse.  

The additions include: 
- a [graphical user interface](https://metasyn.labs.dansdemo.nl/) that uses the API.   
- connecting the Dataverse external tools functionality to the Metasyn API (for more details, see [dataverse_metasyn](../dataverse_metasyn/))
- demo deployment of the API as a web service on the DANS demo server. 

This demo shows that it is possible to call the Metasyn API from a .tab file in Dataverse, generate a GMF, add it to the dataset, and create and download a synthetic dataset from the GMF. No changes are made to the Dataverse code to allow for the integration of this external tool. 

You can use the demo tool on the [SSH test instance](https://test.ssh.datastations.nl/) as long as it is deployed. Follow these steps: 
- While editing a draft dataset, click on the *Access File* menu and choose *Synthetic Replicant File Tool*. This will open a new window showing the web service interface. 
  - Click *Fit model* to generate the GMF.  
  - Add it to the dataset and close the tool window.
- When back in Dataverse, click on the *Access File* menu of the GMF file and again choose *Synthetic Replicant File Tool*.  
  - Click *Generate synthetic data* and download the result. 

### Next steps 
- Implement a restriction that only allows the tool to be called on GMF files instead of all JSONs. 
  - In the current demo, the tool can be called on any JSON file in Dataverse, but will not generate any output on JSONs that are not GMFs.   
  - Dataverse does not currently allow for more detailed conditions under which an external tool can be called, so we would have to make a feature request to Dataverse to make this possible. 
  - One solution could be to propose to expand the existing Dataverse file tag system which allows you to tag files with either Data or Software tags. A designated tag for GMFs could be introduced, allowing us to specify the Metasyn tool can only be called on JSON files that also have the GMF tag. 
- Expand the tool to work with more types than .tab. 
- Also create a provenance file when adding the GMF that describes how the GMF was generated.  
- While we created this web service, the Metasyn developers have also worked on an official web service that provides access to the tool. Once this is launched, we should investigate if we can replace our service with theirs. 



## <span id="anchor-2"></span>Summary

- The goal of this task was to explore how Metasyn, a tool for synthetic
  data generation, can be integrated into the DANS Data Station SSH.
- We propose to use the External Tools functionality from Dataverse that
  allows external tools to be used without leaving the
  depositing/editing page of a dataset.
- We developed a demo version of a Metasyn web service that is integrated into our Dataverse instance. 
- It can be used on .tab files to create a GMF file and add it to the dataset. 
- It can also be used on the GMF to create and download a synthetic dataset. 

## <span id="anchor-2"></span>AI Statement

No AI was used in writing this report.
