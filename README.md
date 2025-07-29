[SQL-MM 3 Topologies](https://www.iso.org/standard/60343.html) provide many advantages over static vector geometry systems.  Often folks will cite the ability to keep multiple layers of hierarchical entities in harmony or lower storage costs - all true.  However, my focus has been mostly on fast aggregation of numerous and complex polygons.  For all the purposes stated, topologies move the cost of determining adjacency up front.  If one can reify how an entire layer of polygons interact beforehand, one can quickly marshal arbitrary aggregations of those polygons on-demand.  Mathematically unioning together thousands (or even millions) of polygons is a task even the most current CPUs will choke on.  If we already have them in a topology then just determine the universe of faces comprising the aggregation and pull out the edges with one of those faces on one side but not the other.  Then just chain the edges into rings to make the polygon or multi-polygon.  It still can take a while but the gory intersection math is only done once.

I've been using topologies to model the NHDPlus medium resolution catchment dataset.  Users commonly would like the watershed as a polygon for some arbitrary distance or flow time along a given stream network.  If this navigation crosses six catchments, sure we can mathematically union them together and return the results in short order.  But what about six thousand?  That is where topologies shine as we can get that big honkin six thousand catchment polygon for cheap.  And its the same relative cost if the user wants 5,999 or 6,001 catchments.  Now this doesn't mean we shouldn't try to cache results, caching is always faster.  But if we can't know ahead-of-time if its 5,999 or 6,000; caching every possible combination is not feasible.  Topology aggregation is the best approach I am aware of.

NHDPlus is built from several grids using several SRIDs, the continental US (SRID 5070) is by far the largest and the only one of concern.  My NHDPlus v2.1 topology of this area ingests 2.8 million catchments creating 7.5 million nodes, 10.8 million edges and 3.2 million faces.  It can take a solid week to throw together depending on resources, approach and time to babysit. However, the more recent NHDPlus HR is a bit larger delivering 23.3 million catchments.  How long will that take to crunch into a topology?  Are there tricks to speed up the process?  PostGIS bugs to submit?  Or am I going nowhere?  This tract is meant to chronicle the journey in the hopes others might glean some wisdom.

### Quick vocabulary

**topology** - the word is at best imprecise, used across and within many disciplines for quite discrete concepts.  This documentation regards one subdefinition being a data model and modeling system for the storage of winged-edge planar relationships as nodes, edges and faces.  Entities are aggregated from these elements as relation records further abstracted into a database type for storage and easy handling.  Commonly used systems in 2025 would be both my focus here, [PostGIS Topology](https://postgis.net/docs/Topology.html) and the now-desolate [Oracle Spatial Topology](https://docs.oracle.com/en/database/oracle/oracle-database/23/topol/topology-data-model-overview.html).  [Topology functionality](https://pro.arcgis.com/en/pro-app/latest/help/data/topologies/topology-in-arcgis.htm) provided by Esri is a related but very different animal.  If you are looking for Esri help, this is not the place.

**face zero** - Face zero is the conceptual face that defines everything outside your topology (Oracle Spatial terms it the "universal face").  In PostGIS topology an edge with face zero on one side is by definition the exterior of your topology.  Face zero is distinct in that its much cheaper to work against and at this time multiple transactions can execute against face zero.  Keeping your activity focused upon face zero to leverage performance and multiprocessing is a huge part of this document. 

**big honkin' face (BHF)** - a BFH is large complex (usually) intermittent face created in the course of building a topology.  Unlike face zero, a BFH must be reified into a polygon for each transaction.  For example, adding a thousand faces along the perimeter of a BFH means walking the BHF's rings and creating a polygon from it a thousand times.  This is slow.  But more importantly, individual elements (of which the BHF is one) cannot be simultaneously edited.  And doing so will corrupt your topology.  Avoiding BHFs is a theme of these notes. 

**cutter** - a process for adding one or more elements to a topology in a single transaction.  One might imagine them as cookie or pizza cutters clipping face zero into topology elements and entities (whatever metaphor works for you but that is mine).  Please note PostGIS Topology does not officially support multiple concurrent cutter processes at this time.  But we need to use them anyways.  This document is meant to show how to use many cutters simultaneously to greatly speed up topology creation.      

### Factors in crafting large topologies

###### Computing resources

It would be interesting to know what other folks have available for tasks like this.  In my case I am using a little home server with 128 GB of RAM, 16 AMD cores and 2 mid-range NVME drives.  The database runs on the stock PostGIS docker image on an Ubuntu host.  More cores and faster drives is always better but I don't myself know of absolute thresholds.  I do think trying this kind of thing on a 16GB Windows laptop is not going to work.

###### PostgreSQL tuning

If you have access to your PostgreSQL configuration then tweaking things towards a small number of connections gorging on system resources is helpful.  Well, unless you share the database with others!  As I am the sole tenant, I set shared buffers to 1/3 of memory, effective cache to 3/4 of memory, stretch out the WAL to 64GB, set up huge pages using tuned for PostgreSQL.  But your ability to change this may be restricted with cloud or shared resources.  I feel being able to shove critical resources into the cache is vital but there is little benchmarking in this document. 

If you believe this will result in occasional OOM killing by the host OS, you are correct.  See [note 18.4.4](https://www.postgresql.org/docs/current/kernel-resources.html#LINUX-MEMORY-OVERCOMMIT) for details on skirting this issue AND be aware that certain kills can corrupt your topology.  Setting overcommit to "2" mitigates the issue _mostly_ for Ubuntu 24.  At least kills in mode 2 never seem to corrupt my topologies.  Dancing on the edge is dangerous.

###### Layer Preparation

Going into topology creation with both a valid and clean layer is important.  Invalid polygons are just a show stopper, you gotta fix it first.  Messy (but valid) polygons are a slightly different issue.  In this reference the new PostGIS coverage concepts. Ideally you want to start with a clean coverage to craft the most compact topology possible.  If your goal is to load a bad coverage to do the cleanup, well that is out of this document's scope. Every face comes with a cost and if your input is a mess you will have many faces per entity, greatly expanding out the creation cost.  I can't afford that cost when crafting my large topologies so thus clean things up beforehand.  The new PostGIS 3.6 coverage tools should be helpful and just tidying up the layer in a desktop GIS is always a solid approach.  The aforementioned Esri topology is quite helpful in determining how clean or dirty your coverage is.

###### Monitoring

For many years I crafted large topologies in the dark.  I'd track progress by US state codes with a vague idea where each cutter was acting.  This is dangerous for many of the reasons highlighted in this document.  Its very helpful to keep a close eye on the process.  QGIS is an excellent desktop tool for this.  Just add your edge table to a map and watch things build.  This can help you spot BHFs or the risk of creating BHFs.  One can harvest bounding boxes directly from QGIS to drive your cutting.  

###### Multiprocessing Management

Launching multiple cutters is a key component but exactly how one manages those processes is beyond this document.  Automation is always a goal but at the same time the danger of corruption often argues for closer management and scrutiny of each cut.  I am a bit ashamed to admit I often just pop off cutting actions as individual psql commands.  Crafting a control system would be swell but at the same time each topology is unique and for my purposes a one-time event.  Your critiques of my mismanagement of the process and suggestions for automation are always accepted.    

###### Backup Management

Users should become familiar with the pgtopo_export and pgtopo_import commands.  You **will** corrupt your topology.  It just happens.  By far the easiest fix is to fall back to the last good backup.  I would advise (at least) a daily backup regime.  Knowing when your topology has gone bad is another important issue.  Validate the thing, as much as possible, as often as possible.  The worst scenario would be to corrupt some far corner of your topology and not realize it until days or weeks later.  Cutting out corruption is a fraught and complex topic not addressed here.  

### Approaches for building large PostGIS topologies via multiprocessing.

Starting a fresh topology via a single cutter should follow the normal documented steps.  Hopefully your source layer has some internal divisions to help.  In my case I have US state codes and I recommend starting in the center of your layer.  In my case, Arkansas.  An empty topology will be fast and building this initial seed should not take very long.  After that I break the work into two cutters, namely Missouri and Louisiana.  Could I immediately launched a boat load of cutters carefully carving out work zones on state borders?  Yes, but dangers abound.  Keeping your cutters far away from each other is the safest approach.  If you accidentally overlap two work zones - corruption.  If you accidently create a BHF that two cutters touch - corruption.  After losing a few days of work, it can lead a person to be a bit gunshy.  As your topology grows and expands you can add more cutters safely isolated from each other. 

A valid approach would be to start with multiple seeds, perhaps start in your corners and build inward.  The problem is the danger of the BHF.  Several of my topology runs have ended in failure when a BHF was inadvertently created from too many seeds growing in two many directions.  Yeah, don't do that.  But it's harder than it looks when your polygons may differ greatly in size and extent and you take greedy gulps being annoyed at how long things are taking.  Don't do that.  

I tend to want to skirt around the outside edge and work inward.  Bad idea.  Rather its best to start in the center extending arms outward, then cut away with one (maybe two) cutters in the area between the arms, slowly filling in careful to not ever close the region creating a BHF.  But the attraction of starting cutting out on the edges is always there.  It just needs to be done in a manner that does not close off your work space.  A large BHF is the end, restore a backup and try again.

A small BHF may not be the end of things.  Its never ideal as cutting against any face other than face zero is more expensive.  But if it happens and its not that large, then start bisecting it into smaller portions over and over until you can close the entire thing.  If you have 13 cores cutting on face zero and one core puttering about dicing up a BHF, that is probably okay.  But be careful.  The urge might be to bisect the BHF and put two cores on the two new faces.  But the risk is real that they bump into each other.  Finding corruption is a bad day, having it happen and not noticing can be a bad week.  I tend to just play it safe and only use one core around a BHF.  

### Why is cutting topology polygons so expensive?

The simple explanation is that crafting a new face requires reifying neighboring faces into polygons - recursively walking the edges to craft rings to create polygons.  As of versioon 3.52, the query is simply
```
WITH RECURSIVE edgering AS ( 
   SELECT 
    $1 as signed_edge_id
   ,edge_id
   ,next_left_edge
   ,next_right_edge 
   FROM 
   topology_name.edge_data 
   WHERE 
   edge_id = $2 
   UNION 
   SELECT 
    CASE WHEN p.signed_edge_id < $3 THEN p.next_right_edge ELSE p.next_left_edge END
   ,e.edge_id
   ,e.next_left_edge
   ,e.next_right_edge 
   FROM
    topology_name.edge_data e
   ,edgering p 
   WHERE 
   e.edge_id = CASE WHEN p.signed_edge_id < $4 THEN abs(p.next_right_edge) ELSE abs(p.next_left_edge) END 
) 
SELECT * FROM edgering
```
If your face is made up of four edges, its super fast.  If your face is made up of four thousand edges, that is slower. And again if the goal is to ultimately add 23 million faces, that is a bottleneck. I find there is performance to be gained with a covering index specific for that query.
```
CREATE INDEX dz_covering ON topology_name.edge_data(
   edge_id,abs(next_right_edge),abs(next_left_edge))
);
```
This allows the recursion to work against the index alone, which if your system has the space to hold in the cache, will be faster than without.  Maintaining this index has a cost and thus its absence is not necessarily a PostGIS bug.   
  




 
