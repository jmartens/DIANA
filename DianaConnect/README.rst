DIANA-connect
=============

| Derek Merck derek_merck@brown.edu
| Brown University and Rhode Island Hospital
| Winter 2018

https://github.com/derekmerck/DIANA/tree/master/DianaConnect

Also sometimes named Tithonus or CopyDICOM

Overview
--------

``DIANA-connect`` is a set of python scripts that can monitor an
installation of Jodogne's `Orthanc <https://orthanc.chu.ulg.ac.be>`__
and copy DICOM imaging to another instance of Orthanc or DICOM tags to a
`Splunk <https://www.splunk.com>`__ index. It can also reduce DICOM
structured report tags into a format following Orthanc's
'simplified-tags' presentation. This can be useful for parsing data from
dose reports in a tag index.

``DIANA-connect`` is intended to be used as an adjunct with an automatic
DICOM data analytics framework, specifically
`DIANA <https://github.com/derekmerck/DIANA>`__, but it works well as a
stand alone tool, with somewhat more intelligent copying than Orthanc's
standard ``Replicate.py`` script. In particular, it does not attempt to
replicate data that is already extant at the destination, and working
with a tag index, it can selectively copy subsets of image data based on
complex queries.

Dependencies
------------

-  Python 2.7
-  Requests

Usage
-----

To use it as a stand-alone script:

.. code:: bash

    $ docker run docker run -d -p 8042:8042 jodogne/orthanc
    $ docker run docker run -d -p 8043:8042 jodogne/orthanc
    $ python CopyDICOM.py replicate \
    >  --src  'http://orthanc:orthanc@localhost:8042' \
    >  --dest 'http://orthanc:orthanc@localhost:8043'

.. code:: bash

    $ docker run docker run -d -p 8088:8088 -p 8089:8089 splunk
    $ python CopyDICOM.py replicate_tags \
    >  --src   'http://orthanc:orthanc@localhost:8042' \
    >  --index 'https://admin:changeme@localhost:8089' \
    >  --hec   'http://Splunk:<token>@localhost:8088

.. code:: bash

    $ python CopyDICOM.py conditional_replicate \
    >  --src   'http://orthanc:orthanc@localhost:8042' \
    >  --index 'https://admin:changeme@localhost:8089' \
    >  --query 'search index=dicom | spath SeriesDescription | search SeriesDescription="Dose Record" | spath ID | table ID' \ 
    >  --dest  'http://orthanc:orthanc@localhost:8043'

To use it as a Python library in a script:

.. code:: python

    >>> import DianaConnect
    >>> DianaConnect.replicate(src='http://orthanc:orthanc@localhost:8042', dest='http://orthanc:orthanc@localhost:8043')

Functionality
-------------

-  ``replicate``: copy all non-duplicate DICOM images from a source
   Orthanc instance to a destination Orthanc instance
-  ``replicate_tags``: copy all non-duplicate DICOM tags from a source
   Orthanc instance to a Splunk index
-  ``conditional_replicate``: Query a Splunk index for a set of
   candidate instances, and copy non-duplicate DICOM images in that set
   from a source Orthanc instance to a destination Orthanc instance.

``conditional_replicate`` is intended to allow automatic duplication of
specific image types from a primary archive into secondary, project
specific DICOM stores, typically with a de-identifier on ingestion. In
DIANA, such secondary image repositories are called "Anonymized Image
Archives" or "AIRs".

Utilization and Dose Reporting with Splunk
------------------------------------------

A simple Splunk query can create a *Count of Studies by Modality by Day*
dashboard from the tag data.

::

    index=dicom | spath ID | spath SeriesDescription | spath StudyDescription | dedup ID | stats count by SeriesDescription StudyDescription

If structured dose reports are included in the archive monitored by
``CopyDICOM replicate_tags``, the dose data will also be available for a
Splunk dashboard, such as reviewing *Dose by Protocol*. This is a
particularly useful function to the Diagnostic Imaging department at RIH
for auditing our quarterly ACR Dose Reports.

For very long, complex dose reports, may need to alter ``_json`` source
type with a new variable: ``TRUNCATE=999999`` to beat the 10k char limit
on a single line.

Testing
-------

An `Ansible <https://github.com/ansible/ansible>`__ playbook is included
that sets up local Docker containers for an Orthanc source (8042), and
Orthanc destination (8043), and a Splunk index (8000/8089) for testing.
Sample data is automatically loaded into the Orthanc source, but the
paths are currently hardcoded for my development system, so other users
will need to modify them.

.. code:: bash

    $ ansible-playbook testbench.yml 

Then login to ``http://admin:changeme@localhost:8000`` and add indices
and get a HEC token. It seems to work best to turn off the global SSL on
the HEC inputs, as well.

Dose Data
---------

For any GE accession, series 997 is the dose S/R series. For Siemens,
504 is the dose S/R series.
