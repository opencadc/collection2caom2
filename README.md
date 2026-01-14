# Making Telescope Data Findable

The value of telescope data to the astronomical community increases if telescope data is findable by users other than the original PI. 

The principles of "discovery of, access to, integration and analysis of task-appropriate scientific data" are [generally-recognized](https://www.nature.com/articles/sdata201618), and have been formalized as "FAIR: Findable, Accessible, Interoperable, Re-Usable".

[What makes data 'findable'](https://www.force11.org/group/fairgroup/fairprinciples):

>  F1. (meta)data are assigned a globally unique and eternally persistent identifier.<br>
>  F2. data are described with rich metadata.<br>
>  F3. (meta)data are registered or indexed in a searchable resource.<br>
>  F4. metadata specify the data identifier.<br>

The repositories in this github organization address items F2 and F3 from above. 

The rich metadata (F2) for telescopes is described in [the Common Archive Observation Model](http://www.opencadc.org/caom2/).

The searchable resource (F3) is [here](https://www.cadc-ccda.hia-iha.nrc-cnrc.gc.ca/en/search/).

This software eases the transition from the telescope model of the data to the CAOM data model. Once the data is described in CAOM metadata, the search interface provides a well-known location for Findable telescope data.

### Other Data Models and Search Facilities

1. [IVOA ObsCore Data Model](http://wiki.ivoa.net/internal/IVOA/ObsCoreDMvOnedotOne/WD-ObsCore-v1.1-20150605.pdf)
1. [IVOA TAP Query](http://www.ivoa.net/documents/TAP/)
1. [DaCHS](http://dc.g-vo.org/)

# Theory of Operation
- explain the problem solved in more detail
- describe the strategy it uses
- establish terminology
- be clear about why things are done as they are

## The Problem Being Solved
What astronomers want to be able to do:
"The  ability to pose a single scientific query to multiple [collections] simultaneously" 
The [IVOA ObsCore Data Model](http://wiki.ivoa.net/internal/IVOA/ObsCoreDMvOnedotOne/WD-ObsCore-v1.1-20150605.pdf) explains use cases quite well (see also Appendix A). The summary information from Section 2 is repeated here:
- support multi-wavelength as well as positional and temporal searches.
- support any type of science data product (image, cube, spectrum, time series, instrumental data, etc.)
- directly  support  the  sorts  of  file  content  typically  found  in  archives  (FITS,  VOTable, compressed files, instrumental data, etc.

What inhibits this:
The data models that describe the output from different observatories are complex, the 
work of multiple creative individuals, enduring, expedient, and unique. To illustrate 
the wide range of models, the [ALMA measurement set specification](https://casa.nrao.edu/casadocs/casa-5.4.1/reference-material/measurement-set)
 includes directory 
and file structure, the [HST](https://archive.org/details/firstyearofhstob00kinn/page/270) structure is proprietary and covered by 
ITAR regulations, and [DAO](https://centreoftheuniverse.org/early-history) began taking data in 1918, on glass plates.

The CAOM2 data model provides a consistent description of the data from different observatories. Each &lt;collection&gt;2caom2 pipeline application captures the software which produces those consistent data descriptions. The captured descriptions are then exposed as a searchable inventory for use by anyone who is interested.

Looked at another way, the CAOM2 data model provides a way to group observational data for effective querying, and effective querying requires metadata normalization to the CAOM2 data model, and metadata cleaning for consistency with the CAOM2 data model.

## The Strategy Being Used

There are three parts to having data represented by CAOM2 Observations:
- mapping the telescope data model to the CAOM2 data model. This happens generically in the
python module caom2utils, and collection-specifically in each &lt;collection&gt;2caom2 module.
- determining the relationship between telescope files and CAOM2 entities - i.e. how many Observations, Planes, Artifacts, Parts, and Chunks are created for each telescope file? This is called cardinality in the code.
This happens in collection-specific code only, in each &lt;collection&gt;2caom2 module.
- putting the pieces of mapping and cardinality together, into a repeatable pipeline for automated execution. 
This happens generically in the caom2pipe module, with collection-specific invocations
in &lt;collection&gt;2caom2 modules.

This diagram describes the dependencies between python modules: 

![](cdep_deps.png?raw=true)

- caom2utils - the generic mapping between the TDM and CAOM2, captured as the
ObsBlueprint class
- caom2 - the CAOM2 model
- caom2pipe - the bits of the pipelines, common between all collections
  - astro_composable - confine reusable code with dependencies on astropy here
  - caom_composable - confine reusable code with dependencies on CAOM2 here
    - TelescopeMapping - A default implementation for building up and applying an ObsBlueprintmap for a file, and then doing any n:n (FITS keywords:CAOM2 keywords) mapping, using the 'update' method.
    - Fits2caom2Visitor -     Use a TelescopeMapping specialization instance to create a CAOM2 record, as expected by the execute_composable.MetaVisits class. 
  - client_composable - confine long-lived HTTP `Session` instances here, for use across other classes.
  - data_source_composable - common pipeline code to encapsulate the mechanisms for identifying the set of work to be done. The identification varies based on data source type, which may include listing a local file system directory, reading the contents of a file, retrieving a listing from a service, or issuing a time-boxed query to a database. The entries in the list of work to be done must be understood by collection-specific StorageName specializations. Used in the run_composable module. Default implementations are:
    - ListDirDataSource - list files in a local file system by naming patterns
    - QueryTimeBoxDataSource - time-boxed queries of a TAP service
    - TodoFileDataSource - read the contents of a file
  - execute_composable - common pipeline steps and execution control
    - CaomExecute - common code to implement each TaskType in a pipeline. Extended multiple ways for specific TaskType implementations.
    - OrganizeExecutor - takes the collection-specific Config, picks the appropriate CaomExecute children, and organizes a series of Tasks for execution. Also manages logging.
  - manage_composable - common methods, used anywhere
    - StorageName - generic interface for managing naming issues for files in collections, and for capturing the relationship between file names, observation IDs, and product IDs
    - CaomName - common code to hide CAOM-specific naming structures
    - Metrics - for tracking how long it takes to use CADC services
    - Rejected - for tracking known pipeline failures
    - Observable - container for Metrics and Rejected instances for a pipeline
    - Config - pipeline control
    - State - bookmarking pipeline execution
    - Features - very basic feature flag implementation - False/True only
    - TaskType - enumeration of the allowable task types. This is the verbiage that a user will see in their config file.
    - PreviewVisitor - common code for generating preview and thumbnail images for a collection
  - name_builder_composable - encapsulates common ways to create StorageName instances. Depending on Config parameters to identify whether the work to be done by the pipeline is identified by file names, observation IDs, or URLs
    - FileNameBuilder - builds a StorageName instance with a file_name parameter
    - ObsIDBuilder - builds a StorageName instance with an obs_id parameter
    - StorageNameBuilder - abstract class
    - StorageNameInstanceBuilder - default implementation that returns whatever it gets
  - reader_composable - common code to retrieve FITS headers and file metadata (needed to fill in Artifact metadata) for all file types from some location. Used in the CaomExecute and OrganizeExecutors implementations to limit the retrieval of this sometimes time-wise expensive information to once per file.
    - FileMetadataReader - from local disk
    - StorageClientReader - CADC storage, either AD or Storage Inventory
    - VaultReader - vos storage URIs
  - run_composable - common code used directly by the collections. Relies on name_builder_composable to provide the correct inputs, and data_source_composable to identify the work to be done.
    - RunnerReport - summarizes successes, failures, retries, and execution errors for a pipeline execution instance
    - StateRunner - common code for time-boxed execution. A specialization of TodoRunner.
    - TodoRunner - common code for work execution.
  - transfer_composable - abstraction to represent transferring a file from one point to another, and checking that the transfer occurred correctly
    - CadcTransfer - get a file from CADC storage
    - FtpTransfer - get a file from an FTP site
    - HttpTransfer - get a file from an HTTP site
    - VoTransfer - get a file from vault
- &lt;collection&gt;2caom2 - contains collection-specific code, including:
  - extension of the fits2caom2 module for TDM -&gt; CAOM mapping code
  - file -&gt; CAOM cardinality code
  - extension of the StorageName class, if unique behaviour is required
  - extension of the data_source_composable classes as required. There may be multiple extensions, depending on the character of the collection.
  - extension of the name_builder_composable classes as required.
  - invocation of the run_composable mechanism. These mechanism invocations are the pipeline execution points.

The Advanced Search UI provides a way to find and download data for processing. Other UIs (TBD) provide ways to find and analyze data through the UI.

## Terminology

1. Files are the current unit of input, as that is the output of every telescope currently being discussed here.

# Detailed API Description

## Logs

If `log_to_file` is set to `True` in the `config.yml` file, then in the `log_file_directory`, the pipeline captures individual `<observation_id>.log` and `<observation_id>.xml` files for each file ingested. The `.log` files can sometimes be useful for debugging the causes of ingestion failures. The `.xml` files are useful for reproducing server-side ingestion errors.

There are always summary files of an ingestion run in the `logs` directory. By default, these files are named `success.txt`, `failure.txt`, and `retries.txt`. Successive runs rename any non-zero sized summary files with a `success.<timestamp>.txt`, `failure.<timestamp>.txt`, and `retries.<timestamp>.txt`  pattern.

## Metrics

If 'observe_execution' is set to True in the config.yml file, then the pipeline captures metrics for the CADC services invoked during its execution. In the observable_execution directory, there will be files named like 1568331248.133947.data.yml, 1568331248.133947.caom2.yml, and 1568331248.133947.fail.yml.

The ‘.data.yml’ file will have entries for ‘get’, and ‘put’, and for each file, will list the time in seconds, the rate in bytes/second, and the start time as seconds since the epoch. The ‘.caom2.yml’ file will have entries for ‘create’, ‘read’, and ‘update’, and for each CAOM2 observation, will list the time in seconds, the rate in bytes/second, and the start time as seconds since the epoch. The ‘.fail.yml’ file wil list, for each file affected by a failed action, a count of how many times it failed.

The files are created for every run of the pipeline.

## Reporting

At the end of a pipeline run, a file named <working directory basename>\_report.txt is created in the logging directory. The content will look like this:

```
********************************
Location: tests
Date: 2020-03-18T17:16:08.540069
Execution Time: 3.98 s
    Number of Inputs: 2
 Number of Successes: 0
  Number of Timeouts: 0
   Number of Retries: 1
    Number of Errors: 1
Number of Rejections: 0
   Number of Skipped: 1
********************************
```

- Execution time: the time from initiating the pipeline to completion of execution.
- Number of inputs: the number of entries originally counted. Depending on the content of config.yml, this value will originate from a work file, or files on disk, or cumulative tracking of entries when working by state.
- Number of successes: the number of entries that eventually succeed, including retries.
- Number of timeouts: the number of times exceptions that may involve timeouts were generated. Those exceptions include messages like "Read timed out", "reset by peer", "ConnectTimeoutError", and "Broken pipe". 
- Number of retries: the number of retries executed. Depending on whether and how retry is enabled in config.yml, this value is the cumulative number of entries processed more than once.
- Number of errors: the number of failures after the final retry. If this number is zero, a retry fixed the failures, so all entries were eventually ingested.
- Number of rejections: the number of entries that are rejected due to well-known processing failures.
- Number of skipped: the number of entries with a checksum that is the same at the data source as it is in CADC storage. If the checksum is the same, the pipeline can make no changes to either the data or metdata, so it doesn't try.

## Repository Configuration
The CAOM2 repositories will need an operatorGroup and staffGroup to automatically generate read access tuples for QA.

If a collection is configured with an operatorGroup and a staffGroup, the service only generates Observation-level `metaReadGroups` and `dataReadGroups` when the release date is null or in the future. If the Observation-level `metaRelease` is in the past at the last time the Observation was updated (shown in `lastModified` timestamp) no Observation-level grants are generated.

If a collection is configured with the operatorGroup and staffGroup not present, then `<collection>2caom2` must set the content of the fields `metaReadGroups` and `dataReadGroups` at the Observation and Plane level.

# Worked Examples
- with explanations and motivations
## Mapping Examples

### Direct Creation of Part- and Chunk-level WCS Information

See [here](https://github.com/opencadc-metadata-curation/caom2tools/tree/master/caom2) for an example of how to create a minimal, or a complete Simple Observation, using the class definitions of the python caom2 module, where the WCS information is captured at the Chunk level.

Use this approach if FITS files are stored at CADC, and it's possible to utilize the existing CADC cutout functionality.

### Direct Creation of Plane-level WCS Information

See [the function _build_observation here](https://github.com/opencadc-metadata-curation/draost2caom2/blob/master/draost2caom2/main_app.py) for an example of how to create a Simple Observation, using the class definitions of the python caom2 module, where the WCS information is captured at the Plane level.

Use this approach if FITS files are not stored at CADC, or the data does not exist in FITS, and therefore there is no cutout functionality.

### Blueprint-Based Creation of CAOM2 Information

See [here](https://github.com/opencadc-metadata-curation/vlass2caom2/blob/master/vlass2caom2/vlass2caom2.py) for an example of how to capture the Telescope Data Model to CAOM2 instance mapping using blueprints.

Use this approach if the source data is in simple FITS files, and the mapping capabilities of the blueprint functionality are sufficient to capture the idiosyncracies of the TDM->CAOM2 mapping for the telescope in question.

For blueprint-based implementations, when one attribute in the CAOM2 record is affected by a particular input
value, use the blueprint. When more than one attribute in the CAOM2 record is affected by a particular
input value, use the update method, which is invoked via a 'visitor'.

#### What is a Blueprint?

A blueprint is a map from the CAOM2 elements to the FITS keyword values, before that blueprint has been exposed to the values that are in a specific FITS file. It is an indirect lookup mechanism.

A blueprint fragment:
   ```
   Chunk.position.coordsys:(['RADESYS'], None)
   Chunk.position.equinox:(['EQUINOX', 'EPOCH'], None)
   Chunk.position.axis.axis1.ctype:(['CTYPE1'], None)
   Chunk.position.axis.axis1.cunit:(['CUNIT1'], None)
   Chunk.position.axis.axis2.ctype:(['CTYPE2'], None)
   Chunk.position.axis.axis2.cunit:(['CUNIT2'], None)
   Chunk.position.axis.error1.syser:(['CSYER1'], None)
   Chunk.position.axis.error1.rnder:(['CRDER1'], None)
   Chunk.position.axis.error2.syser:(['CSYER2'], None)
   Chunk.position.axis.error2.rnder:(['CRDER2'], None)
   Chunk.position.axis.function.cd11:(['CDELT1'], None)
   Chunk.position.axis.function.cd12:0.0
   Chunk.position.axis.function.cd21:0.0
   Chunk.position.axis.function.cd22:(['CDELT2'], None)
   Chunk.position.axis.function.dimension.naxis1:(['ZNAXIS1', 'NAXIS1'], None)
   Chunk.position.axis.function.dimension.naxis2:(['ZNAXIS2', 'NAXIS2'], None)
   Chunk.position.axis.function.refCoord.coord1.pix:(['CRPIX1'], None)
   Chunk.position.axis.function.refCoord.coord1.val:(['CRVAL1'], None)
   Chunk.position.axis.function.refCoord.coord2.pix:(['CRPIX2'], None)
   Chunk.position.axis.function.refCoord.coord2.val:(['CRVAL2'], None)
   ```
The pattern of a blueprint:
 
`caom2 element.attribute tree:(['FITS KEYWORD 1', 'FITS KEYWORD 2', …], DEFAULT_VALUE)`

For the example of `CTYPE1`:
- the caom2 element.attribute tree is `Chunk.position.axis.axis1.ctype`
- the list of FITS keywords is length 1, and is `[‘CTYPE1’]`
- the DEFAULT_VALUE is `None`

The software takes the information stored in this blueprint, and processed the FITS header information based on that. When trying to set a value for  Chunk.position.axis.axis1.ctype it will look in the FITS header at the value of the keyword CTYPE1, and use that value if it exists.

## Cardinality Examples 

TBD

## Pipeline Examples

### How To Create A Pipeline
1. Create an appropriately named repository in the opencadc-metadata-curation organization.
1. Create a new repository, using the blank2caom2 template repository, according to [these instructions](https://help.github.com/en/github/creating-cloning-and-archiving-repositories/creating-a-repository-from-a-template).
1. In the new repository:
   1. rename all the items named blank'Something'.
   1. in main_app.py, capture the TDM -&gt; CAOM2 mapping using the blueprint for FITS keyword lookup,
   or for function execution. Also extend and over-ride StorageName here as necessary.
   1. in composable.py, capture the pipeline execution needs of the collection
      1. 'collection'_run assumes the generation of the todo.txt file is done externally
      1. 'collection'_run_by_state generates the todo.txt file content based on queries, whether of
      tap services or other time-boxed data sources
   1. use data_source_composable and name_builder_composable classes, or provide collection-based specializations, to capture query needs
 
## Motivations

1. Spatial WCS and where it's used:
   1. datalink service calls the SODA cutout service and provides maximum bounds (the minimum circle or polygon that would return all the pixels -- so cutouts that work are inside that). The minimum spanning circle is calculated using the `chunk.position.bounds` WCS information.
   1. SODA service converts inputs to pixels and performs some clipping, so it only uses the `chunk.position.function` WCS information.

# Tricks and Traps
- what might confuse users about API and address it directly
- explain why each gothca is the way it is
- all pipeline execution control comes from the file config.yml, so it must exist in the working directory. See [here](https://github.com/opencadc-metadata-curation/collection2caom2/wiki/config.yml) for a description of its contents.
- add the create/update - must read to update from /ams/caom2repo/sc2repo
- repos are all on master, so anyone at any time can pull a repo and build a working version of any pipeline container
- use feature flags to limit the side-effects of work-in-progress commits

## Authorization Doesn't Really Work With sc2 Record Viewing and Retrieval

1. Authorization only consults /ams content, so even if the group and membership configuration is consistent and correct, if something is not always and only public, it must exist in /ams for file download to work.

   Thumbnails and previews are also affected by this /ams-only check. Viewing these files only works on sc2 if the metadata that refers to the files is also present in /ams. The production `ams.properties` file entry also has to have an `artifactPattern=scheme:COLLECTION/` segment in the `COLLECTION` entry.
 
1. An sc2 side-effect: if a collection is configured with operatorGroup and staffGroup in both ams and sc2repo, two common grants are generated. In ams, if the collection also has proposalGroup=true, ams will create a 3rd grant. That setting is not enabled in sc2repo, so a record generated from sc2repo will lack the proposal group grant. This is on purpose because the creation of the proposal group grant also, as a side efect, creates the group and this side effect should not be triggered by sandbox usage (because metadata there is potentially incorrect  and fixing it can't fix/delete previously created groups).
 

## Things To Know About CAOM2 Observation Construction

### Observations

1. Choose a Simple Observation representation as the default.

1. Choose a Derived Observation representation when:
    1. stacking
    2. mosaicing
    3. splitting out commensal observations

1. CAOM Collection names vs Storage Inventory (SI) Namespaces - In CAOM the collection is limited to a single path component string and other places to put data release for such surveys is not very visible. LSST, for example, chose to use obs_collection of LSST.DP1 for this reason.
In SI, the files can be organised into <scheme>:<collection>/DR1 (can have arbitrary paths and namespadce is more loosely any prefix). Technically, the namespace and the collection do not have to match, but when there is an n..m relationship between
collection and SI namespace it makes artifact validation (caom2 vs SI) harder. For example, validation would have to use 2 collections vs 1 namespace so it is doable (from a query point of view)... but if there are artifacts in SI that don't match we don't know which collection they should belong to.

1. Proposl - (from Pat) proposal is mostly for the creation of SimpleObservation(s) by using the telescope (it may also cover the creation of DerivedObservation(s) if that was part of the stated intent of the observing proposal/plan). DerivedObservation(s) created by another project/effort that happens to re-use those available members/inputs but are not part of an observing proposal per se should not have a proposal at all. The proposal information of the members can be found using a multi-step navigation in the general case.

### Planes

1. Plane-level metadata is only computed for productType=science|calibration. Auxiliary artifacts (or parts or chunks) are expected to be part of another plane with science, unless it is a temporary state caused by ingestion order.

1. For radio, plane level position resolution is used to answer the question "what is the smallest scale that can be resolved?", and is most often the synthesized beam width.
 
1. For projects that speak SKA product levels as defined on Page 3 of [https://www.skatelescope.org/wp-content/uploads/2013/04/Cornwell-SKA_Lowscienceassessmentdataproducts.pdf], here's the mapping to the `Plane.calibrationLevel` values in the model:
  - everything up to level 3 (correlator output ) would be calibration level 0 in ObsCore and CAOM.
  - the visibility data in level 3 could be calibration level 0 or 1 (raw).
  - for the things in level 5, calibrated data would be calibration level 2 (but not sure what that is relative to "images" and would have to know that to decide). My understanding of radio "imaging" process is that they would be calibration level 3 (products) because there are many ways to tune the imaging to produce different products for different purposes.
  - for catalogues, ObsCore and CAOM don't treat those as simply higher calibration level; that's just a different kind of thing that itself could be raw (source detection data) or more processed to produce a calibration level 2 or 3 (product)... although the latter sound more like something in level 7 of that slide.
  - level 6 and 7 products in that slide look more like calibration level 2-3 made/vetted by other teams.

1. There is a use case "get me the files associated with this productID", so during data engineering prefer unique `productID` values.

1. A Plane can have provenance.inputs in a Simple Observation. For example, if one file is the calibrated version of another file, they should exist in the same Observation, each with their own Plane/Artifact hierarchy, and the calibrated Plane.provenance.inputs will refer to the raw Plane.productID.


### Artifacts

1. Grouping files into observations/planes/artifacts is determined by how independent the files are. A measure of file independence is whether or not the file can be scientifically understood in isolation. Some observational products may be made up of multiple files - e.g. NGVS .flag, .weight, .image files make up the same observational product. Some observational products may be made up of single files - e.g. CFHT o, p, and b files. In this example, the CFHT files are independent, and thus can be grouped in isolation from each other, while the NGVS files are dependent, and should be grouped together.

1. The impact of `productType`: if an Artifact has `productType = info` it is ignored when the services compute plane metadata.

1. The Artifact.uri value in the CAOM metadata record must match the Artifact.uri value in the Storage Inventory record. These values are in a primary/foreign key relationship between the two systems.


### Chunks

#### CoordAxis1D/CoordAxis2D

1. It is valid to have  bounds, range, and/or function as stated in the model. Practically, bounds is  redundant if you have a range. Bounds provides more detail than range, including gaps in coverage, and enables a crude tile-based cutout operation later. Historically, range was added later in the model life.

#### Chunk.naxis, Chunk.energy_axis, Chunk.time_axis, Chunk.polarization_axis, Chunk.custom_axis

1. In general, assigning axis indices above the value of naxis (3 and 4 here) is allowed but more or less pointless. The only use case that would justify it is that in a FITS file there could be a header with NAXIS=2 and WCSAXES=4 which would tell the fits reader to look for CTYPE1 through 4 and axes 3 and 4 are metadata. Assign those values to Chunk only if you care about capturing that the extra wcs metadata was really in the fits header and so the order could be preserved; in general do not assign the 3 and 4.

1. In an exposure where the position is not relevant and only the energy and time is relevent for discovery, the easiest correct thing to do is to leave naxis, energyAxis, and timeAxis all null: the same plane metadata should be generated and that should be valid. The most correct thing to do would be to set naxis=2, positionAxis1=1, positionAxis2=2 (to indicate image) and then use a suitable coordinate system description that meant "this patch of the inside of the dome" or maybe some description of the pixel coordinate system (because wcs kind of treats the sky and the pixels as two different systems)... I don't know how to do that and it adds very minimal value (it allows Plane.position.dimension to be assigned a value).

1. When there is no spatialWCS, setting naxis, position_axis_1, and position_axis_2 values to null allows services to know that they can't do WCS-based cutouts.  The pixel-based cutouts will still work as they do not use that information.

#### Chunk.observable

The `chunk.observable` doesn’t currently provide info that is actionable in any CADC services, but also there is no other place to put the `observable.cunit` which does seem like it would be useful in future.

`observable.ctype` (coordinate type) was probably not the best name for this field because we’re wanting to put things in it that are not really coordinate types. The UCD is a better way to say “what are these values” since it is a vocabulary of astronomical concepts, so maybe we shpuld put UCDs here (in order to be able to also put the units) and then make sure the UCD of the science chunk.observable is also in the Plane.observable.ucd field. Usable UCD values are pending.

#### SubIntervals

1. SubIntervals are subject to the check "sorted by increasing value, so sample[i].upper < sample[i+1].lower"

#### What if there's no WCS?

1. There are OMM observations with no WCS information, because the astrometry software did not solve. The lack of solution may have been because of cloud cover or because a field is just not very populated with stars, like near the zenith, but the data still has value.

   There is the CoordError datatype which is set for a CoordAxes2D object that would capture this detail. By approximating a WCS solution, and providing a syser value that is representative of the point-model error of the facility (e.g. CFHT has approximately 1 arcmin), and then represent this by making the bounding box bigger by the value of the syser on the coord.

#### Under what conditions should chunk.productType be set?

1. Part.productType and Chunk.productType should be set if the applicable value differs from the parent. In many cases the value in the Artifact is sufficient. I think it best to set productType the minimum places, but it isn't wrong to set it everywhere.

   For example, a FITS file with two extensions: one science and one auxiliary. The Artifact.productType would be science and the Part.productType would be null (for the science part) and auxiliary for the auxiliary part. One would only need to set Chunk.productType if there are multiple chunks and some differed from the Artifact or Part productType.

### Inputs vs Members

1. Pat - Generally, members is what the grouping algorithm intended to include in the processing and inputs is what was actually included. In an ideal world, the inputs are simply the plane(s) of the member(s). Then one has to decide what to do with inputs that were rejected at processing time (inputs is a subset of members since we don't have any role or flags in the input relationship?) and whether inputs should refer to non-member (eg calibration inputs for a science co-add) entities. That's in principle within scope of provenance but I don't have any use cases to guide me in giving advice and we don't really have any s/w or systems that use the provenance metadata. Mostly we want users to be able to tell whether A and B have common "photon ancestry", bit only because we always thought it was important for people to know that (not that we know of anyone doing the necessary queries).

1. Members - Pat - members would all be the same type as the DerivedObservation, so in the typical case the members of a science DerivedObservation would be the science observations that were combined to create it. Thus the members list describes the intended operation.

1. Inputs - Pat - inputs describes the provenance of the resulting data, so it captures what actually happened. That normally includes the planes from each of the members that was used and could also include other kinds of inputs (eg calibration inputs) that do not correspond to any observation in the members list.

    There is currently no role information in the provenance/inputs but it is needed to really generalise the usage beyond just science data. If the goal is to give someone enough info to reproduce a plane from the provenance then other info would be needed.
    
    So, for now, I would only put the real components in members and put at least the planes from those in the inputs; adding more inputs (eg calibration inputs of a science product) can be done but I don't think it is justified right now.

    If the DerivedObservation has intent=science then the members would all be intent = science.

### Permissions

1. CAOM2 access tuple information is only used for READ access to metadata and data.
1. The 3 types of read access tuples can be generated in two ways:
    1. The collection can be configured (sc2repo.properties) with an operatorGroup (aka CADC), and staffGroup (aka NGVS) and these directly enable creation of tuples for those two groups; if staffGroup is set, then a 3rd option proposalGroup=true enables generation of groups based on Observation.collection and Observation.proposal.id (and as a side effect, creation of those groups and adding the staffGroup as admin); the TEST collection has the full config; None of these are set for NGVS so no tuples are generated.
     1. in CAOM-2.4 these permissions are part of the model and read access group URIs can be included in the observation, thus the client can create arbitrary tuples.
     
### MultiPolygon

The use cases for defining boundaries do not include holes.

At the plane level, the searchable (indexed) Plane.positiion.bounds is just the simple Polygon or Circle; the Plane.position.bounds.samples MultiPolygon can store the exact boundaries/coverage (including holes) but there is no way to search that would, for example, exclude data where the target coordinates were in the hole or even between disjoint areas. So, while in principle the plane metadata supplied could include holes, the server-side computation does not (can not?) be made to generate it. Creating a number of artifacts/parts/chunks/ that don't match the file structure would end up being worse. Right now, the best that can be done is to ignore the hole and just store the encompassing polygon.

### Observable Axis

SGw: observable is for the case where a spectra is represented with a 2 row image: one row for wavelength, and one row for flux.

### Outstanding Questions

1. Where in the CAOM2 model do we keep the dimensions of the FITS data?  This is useful metadata for queries.  When searching for FLAT/BIAS calibrations one needs to have ones with the same NAXIS1/NAXIS2 values.

# The Relationship to Long-Term Storage

CAOM2, and the services and databases that implement it, is the metadata record that supports findability. Once files have been found, they need to be used. For that, there is the [Storage Inventory](https://github.com/opencadc/storage-inventory). SI manages data, in the form of archival file storage for science data archives.

The relationship between the metadata and the data is expressed in the value of the `Artifact.uri`.

# Credits and Connections
- contributors

## Sponsors

[CADC](http://www.cadc-ccda.hia-iha.nrc-cnrc.gc.ca/)

# Brief Revision History
