easy-update-metadata-dataset
===========
[![Build Status](https://travis-ci.org/DANS-KNAW/easy-update-metadata-dataset.png?branch=master)](https://travis-ci.org/DANS-KNAW/easy-update-metadata-dataset)


SYNOPSIS
--------

    easy-update-metadata-dataset [--doUpdate] <input-file> [<complex_values_directory>]


DESCRIPTION
-----------

Batch-updates XML streams of objects in a Fedora Commons repository.
The command is idempotent, so running it either once or multiple times gives the same result.


ARGUMENTS
---------

           --doUpdate   Without this argument no changes are made to the repository, the default is a test mode
                               that logs the intended changes
       -h, --help       Show help message
       -v, --version    Show version of this program
         
       trailing arguments:
           input-file (required)                     The CSV file (RFC4180) with required changes. The first line must
                                                     be 'FEDORA_ID,STREAM_ID,XML_TAG,OLD_VALUE,NEW_VALUE', in that
                                                     order. Additional columns and empty lines are ignored.
           complex_values_directory (not required)   A reference to a directory containing files with the complex
                                                     values. These files are referenced in the input-file.
                              
                              
EXAMPLES
--------

### Example: update simple value in EMD

This module was developed to update simple values in the EMD, like the AccessCategory. 
`easy-update-metadata-dataset --doUpdate changeAccessCategory.csv`

```csv
FEDORA_ID,      STREAM_ID, XML_TAG,      OLD_VALUE,    NEW_VALUE
easy-dataset:1, EMD,       accessRights, GROUP_ACCESS, OPEN_ACCESS
easy-dataset:1, DC,        rights,       GROUP_ACCESS, OPEN_ACCESS
```
### Example: check update datasetState in AMD

To check a potential change of the state of a dataset, use the following
`easy-update-metadata-dataset changeDatasetState.csv`

```csv
FEDORA_ID,      STREAM_ID, XML_TAG,      OLD_VALUE, NEW_VALUE
easy-dataset:1, AMD,       datasetState, DRAFT,     SUBMITTED
```

when the dataset does not contain the expected value as given in `OLD_VALUE` it will produce a log line like `expected AMD <datasetState> [DRAFT] but found [PUBLISHED]`. No values will be changed, because the `--doUpdate` arguments is missing.

### Example: Adding complex content

`easy-update-metadata-dataset --doUpdate changeContributors.csv ~/contributorSnippets/`

```csv
FEDORA_ID,      STREAM_ID, XML_TAG,         OLD_VALUE, NEW_VALUE
easy-dataset:1, EMD,       emd:contributor, NILL,      contributor_1.xml
```

when the `OLD_VALUE` is `NIL`, the value in `NEW_VALUE` must be __added__ to the tag 
mentioned in `XML_TAG` instead of updated. A check is performed to see if the 
`NEW_VALUE` already exists, to prevent doubles.
The `complex_values_directory` is given, so the value of `NEW_VALUE` is interpreted 
as a relative path in this directory, referencing a file containing the new value in ~/contributorSnippets/contributor_1.xml
In this way, complex tags, with child-elements, can be added.
The content of which is 
```xml
<eas:contributor>
    <eas:organization>EU</eas:organization>
    <eas:organizationId eas:identification-system="http://isni.org/" eas:scheme="ISNI">0000 0001 2170 6158</eas:organizationId>
    <eas:role eas:scheme="DATACITE">RightsHolder</eas:role>
</eas:contributor>
```

CONTEXT
-------

It is the responsibility of the caller to

* Provide _valid_ new values in the input file.
* Make sure the CSV file is properly stored as UTF8, please export a spreadsheet with open-office for a valid format.
* Specify all changes for one dataset in subsequent lines, thus at most one dataset will have inconsistent values
  in case of trouble such as the old value not found or a system failure.
* Verify the preconditions for stream `AMD` and XML tag `datasetState` which requires a change history.
  Details are documented with [tests], note that some legitimate preconditions are not implemented and cause a failure,
  not expected preconditions might pass without a warning.
* Execute with a representative sample in test mode (without `--doUpdate`) and review the logged changes.
* Change the streams `DC` and `EMD` alike as far as applicable.
  In case of `EMD,accessRights` / `DC,rights` also
  * Update [file rights].
  * Call [easy-update-fs-rdb].
  * Reboot the web-ui to clear the [hibernate] cash.
* Call [easy-task-add-new-license] if EMD and/or file rights were changed, the link requires access to the legacy code base.
* Update relations such as `hasDoi`, `isMemberOf`, collections ...
* Call [easy-update-solr-index] if necessary.

The legacy code base provides [CSV examples] for the `deasy` environment.

[easy-update-fs-rdb]: https://github.com/DANS-KNAW/easy-update-fs-rdb
[file rights]: https://github.com/DANS-KNAW/easy-update-metadata-fileitem
[hibernate]: http://hibernate.org/
[easy-task-add-new-license]: https://github.com/DANS-KNAW/easy-app/blob/master/tool/task-add-new-license/README.md
[easy-update-solr-index]: https://github.com/DANS-KNAW/easy-update-solr-index
[tests]: src/test/scala/nl/knaw/dans/easy/umd/TransformerSpec.scala
[CSV examples]: src/test/resources


INSTALLATION AND CONFIGURATION
------------------------------
Currently this project is built as an RPM package for RHEL7/CentOS7 and later. The RPM will install the binaries to
`/opt/dans.knaw.nl/easy-update-metadata-dataset` and the configuration files to `/etc/opt/dans.knaw.nl/easy-update-metadata-dataset`. 

To install the module on systems that do not support RPM, you can copy and unarchive the tarball to the target host.
You will have to take care of placing the files in the correct locations for your system yourself. For instructions
on building the tarball, see next section.

BUILDING FROM SOURCE
--------------------
Prerequisites:

* Java 8 or higher
* Maven 3.3.3 or higher
* RPM

Steps:

        git clone https://github.com/DANS-KNAW/easy-update-metadata-dataset.git
        cd easy-update-metadata-dataset
        mvn clean install

If the `rpm` executable is found at `/usr/local/bin/rpm`, the build profile that includes the RPM 
packaging will be activated. If `rpm` is available, but at a different path, then activate it by using
Maven's `-P` switch: `mvn -Pprm install`.

Alternatively, to build the tarball execute:

    mvn clean install assembly:single
