######################################
Data replication between APDB and PPDB
######################################

Abstract
========

This technical note discusses various issues and implementation options for the process of data replication between Alert Production Database and Prompt Products Database.


Some history
============

Alert Production Database (APDB) development was focused until recently on satisfying performance requirements of the AP Pipeline.
Currently two implementation of APDB exist, one is based of SQL backend (PostgreSQL) and another based on NoSQL `Apache Cassandra`_.
SQL backend exists solely for testing purposes, it was shown early on that a single-server RDBMS cannot satisfy AP performance requirements (`DMTN-113`_).
Cassandra backend was shown to scale reasonably (`DMTN-156`_, `DMTN-184`_), currently we are in the process of setting up Cassandra cluster at USDF for further testing.

APDB can only be used by AP pipeline, for daytime processing and serving user queries the data from APDB have to be copied to Prompt Products Database (PPDB).
Similarly, the data produced during daytime processing in PPDB have to be copied to APDB for use during the next night.
PPDB will be backed by RDBMS server (PostgreSQL) which allows arbitrary queries, though it is not clear yet that performance of a single-server instance is adequate for this purpose.
The data in APDB and PPDB have to be eventually consistent, which makes these two separate systems one distributed database (at least in logical sense).

The design of APDB/PPDB data transfers was the subject of an early `PPDB architecture meeting`_, though at that meeting we mostly considered replication of DIA tables (DIAObject, DIASource, and DIAForcedSource).
More recent e-mail exchange provided additional details for how non-DIA tables (SSObject, SSSource, MPCORB) are populated and replicated.
Here is a list of facts that summarizes AP pipeline interaction with APDB:

- AP detects DIASources and generates DIAObjects from the 12-month history of DIASources and DIAForcedSources, to retrieve history it runs spatial/temporal queries on APDB which cover current detector+visit region.
- If AP can associate DIASource with a known SSObject, it will read 12-month history of SSSources, update SSObject information and include SSObject and SSSources into the generated alert (and it will not make DIAObject).
- To associate DIASource with a SSObject, it will need to read known SSObjects from APDB corresponding to the same detector+visit region.
- All newly created and updated records for DIASource, SSSource, DIAObject or SSObject will be written to APDB.

During daytime processing, which uses data from PPDB, some DIASources can be re-associated to either existing or new SSObjects, this can also cause removal of DIAObjects that have no matching DIASources.
These changes to DIAObject and DIASources need to be propagated back to APDB in addition to all changes to SSObjects, SSSources, and MPCORB tables.

The estimates for the numbers of new objects are:

- Approximately 15k alerts/visit, translating into the same number of DIASources/DIAObjects.
- Average ~400 asteroids/field, probably translating into the same number of SSSources/SSObjects detected by AP pipeline.
  Peak number can reach can reach ~5000 for the dense ecliptic fields.
- As each detector image is processed independently, the per-detector numbers are also interesting.
  Per-detector averages can be obtained by dividing above numbers by 189, but peak numbers may not be spread evenly.
  A guess for peak per-detector asteroid count could be of the order of 100.


Querying non-DIA tables in AP pipeline
======================================

Current APDB interfaces in ``dax_apdb`` implement storing and querying of DIA tables.
DIAObject queries are naturally spatial, they return all DIAObject records that fall into the detector region (or somewhat larger region).
DIASource queries are also region-based in Cassandra implementation which adds a special column for spatial indexing.

Non-DIA tables do not have spatial information that can be used in queries directly, so querying per-detector set of records becomes a challenge.
One simple option to give each AP process a full set of records of each non-DIA table and let them to filter out non-relevant records.
This is obviously not very efficient use of resources, as there will be millions of records in total and only ~100 that are useful for currently processed region.

A better approach could be to extend the schema of non-DIA tables in APDB to include an additional spatial indexing column.
When those tables are replicated to APDB (which happens daily) the positions of the moving objects will be calculated, potentially for the next few days.
From these positions a set of spatial indices (e.g. HTM with a reasonable granularity) will be determined and for each of these indices APDB will store a copy of the original record.
This will result in multiple copies of the same data, but the data volume is not important in this case compared to reduction of the data volume that is returned from a query.
The query will then use this spatial index to ignore the bulk of the orbits that are not close to the region of interest.

There may be other variations of the same idea which can be considered when we start to implement an extended APDB API.


Replicating DIA tables
======================

As specified above, all records in DIA tables created in APDB have to be replicated to PPDB before daytime processing starts.
Additionally, daytime processing can re-associate DIASources to SSObjects and also delete DIAObjects which have no associated DIASources.

Cassandra implementation of APDB is optimized for regular AP pipeline queries using spatial indexing.
Replication process, on the other hand, is time-based and cannot use spatial indexing.
Cassandra supports secondary indexing, but its performance is not adequate for replication use case.
One of the ideas suggested at `PPDB architecture meeting`_ was to write new DIA records to a separate set of files and transfer those files asynchronously to PPDB.
This approach would cause many complications, so the alternative solution was designed that uses the same Cassandra storage.

The new records, as they are inserted into regular DIA tables, are also stored in separate set of replica tables that use time-based partitioning and indexing.
Every store operation that inserts a number of records into APDB is associated with a ``chunk ID`` token.
The replication process will discover ``chunk ID`` tokens in APDB that are ready to be transferred, and  copy the data corresponding to that token to PPDB.
The tokens and their corresponding data will be removed from APDB after some period.
APDB has special table that tracks newly-created ``chunk ID`` tokens, PPDB (or a separate replication database) will need similar approach to track tokens that have been replicated.

The ordering of transfers in replication process is important.
For optimal performance the DIAObject records in Cassandra implementation do not have their ``validityEnd`` column updated, so it is always ``NULL``.
When DIAObjects are copied to PPDB, the replication process has to populate ``validityEnd`` of the DIAObject which is already in PPDB using ``validityStart`` of the same DIAObject being replicated.
This implies that the time order of the replication needs to be the same as time order in which data were originally inserted into APDB, without such ordering it will be very difficult to manage ``validityEnd`` updates.

To implement propagation of PPDB updates to APDB the replication process will need to know which DIASources need to be re-associated and which DIAObjects need to be removed.
To support this, PPDB will probably use similar approach of keeping the separate log of updates that can be replayed on APDB side.


Replicating non-DIA tables
==========================

New SSSource and SSObject records produced by AP pipeline need to be copied to PPDB along with DIA tables.
It could be reasonable to use the same mechanism relying on ``chunk ID`` tokens and separate staging tables for the new records.
Special care will be needed in the replication process to order PPDB updates, as SSSource records have a dependency on their corresponding DIASource records and some DIASources are linked to SSObjects (it may not be expressed explicitly in the current ``sdm_schema`` definition, but it should be).

Details of replication from PPDB to APDB depend on how things get updated in PPDB.
``MPCORB`` table is recomputed completely for each daily processing, so it makes sense to transfer the complete contents of that table to APDB and drop/recreate ``MPCORB`` table in APDB, which may also be most efficient approach for Cassandra.
``SSObject`` table can use the same approach as it has one-to-one correspondence with ``MPCORB`` records.
``SSSource`` table, on the other hand, may be more efficient to transfer in incremental way, only including additions since previous replication.
To support incremental ``SSSource`` updates, PPDB may need to implement tracking of the updates similarly to what was done on APDB side.
Potential complications for incremental replication could arise if PPDB can remove or replace existing records, as opposed to just adding new records.
Incremental updates may also be problematic if spatial index needs to be added to the records.

For both DIA and non-DIA tables, the replication process will need a non-trivial logic to handle ordering and dependencies between updates.
This logic will likely need an extensive persistent state to keep the record of the transfers.
For that purpose it may be necessary to create additional tables in PostgreSQL database, possibly in a separate schema or database.


Further work
============

USDF is presently has a Cassandra cluster that will be used for various tests with AP pipelines and for development of replication service.
The replication system will need an interface to both APDB and PPDB, ``dax_apdb`` implements access to APDB, existing ``dax_ppdb`` package, which was initially created for APDB, will be used to develop PPDB interface.
``dax_ppdb`` interfaces will initially be targeted for replication use, they can be extended later for other types of queries, though direct use of SQL (e.g. via ``SQLAlchemy``) may be an option for general queries.



.. _PPDB architecture meeting: https://confluence.lsstcorp.org/display/DM/2021-09-21+PPDB+Tag+Up
.. _Apache Cassandra: https://cassandra.apache.org
.. _DMTN-113: https://dmtn-113.lsst.io/
.. _DMTN-156: https://dmtn-156.lsst.io/
.. _DMTN-184: https://dmtn-184.lsst.io/
