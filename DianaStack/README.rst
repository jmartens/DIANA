DIANA-stack
===========

| Derek Merck derek_merck@brown.edu
| Brown University and Rhode Island Hospital
| Winter 2018

https://www.github.com/derekmerck/DIANA/tree/master/DianaStack

Dependencies
------------

Python Package Requirements
~~~~~~~~~~~~~~~~~~~~~~~~~~~

-  `Python <http://www.python.org>`__ 2.7.11+ for scripting
-  `Ansible <http://www.ansible.com>`__ for service orchestration
-  `pyyaml <http://pyyaml.org>`__
-  `jinja2 <http://jinja.pocoo.org>`__

External Requirements
~~~~~~~~~~~~~~~~~~~~~

-  `Docker <http://www.docker.com>`__ for service virtualization
-  `Orthanc <http://www.orthanc-server.com>`__ for DICOM storage and
   bridging DICOM to REST
-  `Postgresql <http://www.postgresql.org>`__ 9.5+ for scalable Orthanc
   backend
-  `Splunk <https://www.splunk.com>`__ 6.6+ for data indexing

Use Cases
---------

DIANA for Mass DICOM Archive
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

role: ``dicom_archive``

1. Create an index role
2. Create a DB role
3. Create n+m DICOM repo roles

-  n archive workers
-  m queues, muxes

4. Create an http reverse proxy role for archive workers
5. Create a bridge role repo/series -> index

6. Install the *dicom-series* dashboards

Creating the index first allows the system to ship logs directly to the
index

DIANA for PACS Crawling
~~~~~~~~~~~~~~~~~~~~~~~

role: ``pacs_crawler``

1. Create an index role
2. Create a DICOM repo role
3. Create a bridge role repo/remote -> index

4. Register the Orthanc repo with the PACS as a valid query source and
   move target
5. Install the *remote-studies* dashboards

DIANA for VPN'd DICOM forwarding
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

role: ``vpn_forwarder``

1. Create a VPN client role
2. Create a DICOM repo role

-  configure as a queue that receives locally and forwards to the VPN

Great for small, low powered devices at remote facilities that need to
send images or structured dose reports to a primary archive. Building on
ARM (Raspberry Pi 3) is problematic.

DIANA for Dose Management
~~~~~~~~~~~~~~~~~~~~~~~~~

role: ``dose_monitor``

1. Create an index role
2. Create a DICOM repo role
3. Create a bridge role repo/dose -> index

4. Point modality dose reports to the dose repo
5. Install the *dose-monitoring* dashboards

Reconciling Missing Data on Dose Reports
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Occassionally the EHR will be unavailable to automatically assign
accession numbers or protocols as the studies are performed. For
example, we recently lost LifeChart for 4 hours, and ended up with about
40 studies that had absent or hand-input improper accession numbers
assigned. Because the index assumes the accession numbers are correct,
this can lead to missing studies in the summary information.

Reconciling this is somewhat complicated, but can be scripted with
CopyDICOM.

index = Splunk gateway dicom\_node = Orthanc with structured dose
reports remote\_bridge = Orthanc with access to the PACS where the
updated accession numbers live

1. Extract the Study Instance UID of the bad reports from Splunk

::

    study_ids = index.DoQuery( "index=dose_reports | ..."" )

2. Ask the PACS for the updated accession numbers for those
   StudyInstanceUIDs.

   ::

       for study_id in study_ids:
         accessions.append( remote_bridge.DoRemoteQuery( {"StudyInstanceUID": study_id} ) )

3. Modify each report in the DICOM node

   ::

       For study_id, accession_num in zip( study_ids, accessions ):
         dicom_node.modify( {"StudyInstanceUID": study_id}, {"AccessionNumber": accession_num} )

4. Drop the bad reports from the indices

   ::

       index.DoQuery( "index=dicom_series | ... | del" )
       index.DoQuery( "index=dose_reports | ... | del" )

5. Re-index the modified files and update the dose registry

   ::

       index.UpdateSeries( dicom_node )
       index.UpdateDoseRegistry( dicom_node )

DIANA for HL7 Dashboards
~~~~~~~~~~~~~~~~~~~~~~~~

role: ``hl7_consumer``

1. Create an index role
2. Create an FTP forwarder role

3. Install dashboards

REST Head Uploads
-----------------

Max file size is set to 8MB in ``nginx``. Most CTs are only .5MB, but
some long scouts for panscans may be 4+MB uncompressed.

License
-------

MIT
