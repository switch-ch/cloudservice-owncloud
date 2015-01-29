CESNET Experiences
====================

This chapter discusses technical challenges we have faced and experiences
we have gained while deploying and running the ownCloud service.

Deployment
----------

When we started deploying the ownCloud service, Puppet made our lives much
easier. We were benefiting from an already existing Puppet infrastructure
that is used to manage the whole storage infrastructure
and some basic Puppet modules from which we could build the service. It
would certainly be much more uncomfortable experience to configure and
deploy all pieces needed to each node manually (or in an 'ssh in a for loop' way).

Puppet automated most of the deployment tasks for us.
We developed modules for each service, then specified to which
nodes to deploy the services and watched Puppet do its job. With this approach, we are able to easily add more nodes to our ownCloud cluster or to deploy the same setup onto a different site.

HA Configuration
----------------

The only major things we kept away from the Puppet's reach were filesystem preparation and a Pacemaker HA configuration. Preparing a filesystem was pretty straightforward, but setting
up a HA cluster has proven to be a major challenge for us. We had to prepare RA's (Resource
Agents) for each component desired to run in HA mode. These agents monitor and manage (start, stop, restart, …) components they are assigned to. As the dependency tree (order of starting/stopping components, collocation, STONITH & failover) among those RA's grown up, things were getting rather complicated. The main source of problems was surely the RA's monitor function. Each running
component must respond to a monitor called from a RA within a certain timeout. When a monitor
timeouts, RA declares component as failed and Pacemaker automatically stops all components
that depends on the failed one. Although this is correct behavior, we were often experiencing failed monitors (and thus stopping of dependent services) when the incriminated component was working just fine. It just responded to monitor too late under some load patterns. We didn't want to set it overly high too (in order to respond to real failure quickly). It took us
a lot of time (and a lot of hair pulled out :-)) to figure out the correct monitor timeouts
for each RA. But nowadays we can say, that it's pretty stable and behaves the way we expect it to.


FIXME: tohle je az moc pohadkovym stylem a hlavne to vubec nerika nic o
tom, co tam bylo za realny problem. Jestli je potreba rict, ze monitory se
musi poladit na time-outy podle aktualni konfigurace, tak by to asi stacilo
strucneji. Takhle to trochu pusobi dojmem, ze jsme idioti (coz rozhodne
nejsme).

ownCloud Application
--------------------

For the time we were running our ownCloud service, we have stumbled upon
a handful of interesting software problems / bugs, both in ownCloud and
synchronization clients. Some of them are described in the following section.

User Files Coming from the Future
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Among the first things we observed (and we are still observing) in our logs
were errors like this::

	{...,An exception occurred while executing
	UPDATE "oc_filecache" SET "mtime" = ?, "etag" = ?, \"storage_mtime\"=?
	WHERE "fileid" = ?: SQLSTATE[22003]: Numeric value out of range: 7
	ERROR:  value \"5998838837323182874\" is out of range for type integer"...}

Root cause of these errors was that files with bad mtimes were
synchronized to us from the user clients. Most of those files were
coming from a cameras without a properly set date, or any other
misconfigured devices. This caused sync problems and even unability
to log in for the one of the users. But right now we don't have a better
solution other than to recommend users to watch out for these files.

FIXME: tohle by chtelo aspon "reportovali jsme vyvojarum, dosud neni
opraveno" pred tu posledni vetu (jestli se to teda stalo)


Cron Job Traffic Jam
~~~~~~~~~~~~~~~~~~~~

For a long time, we have been fighting with a steadily worsening ownCloud website responsiveness.
It got to the point where it was a way too sluggish at a given number of users and usage.
When looking at the page loading times, it took most of the time to load generated JS
files like 'core.js'. But after running XDEBUG we started to suspect the database
to be the major bottleneck. Database node had all CPU cores utilized to the max
almost all the time. The response times of database backend were way too high.
PgPool II deployment and PostgreSQL configuration tuning helped for a while, but after
few months it got worse again.

FIXME: nedame sem jeste graf narustu uzivatelu nekam? Protoze by se mozna
dalo rict, ze by to clovek pricital tomu, ze pocet uzivatelu rostl, ale ze
priciny byly typicky jinde a daly se vyprofilovat.

When we started looking at a queries that were being executed on the database, we suddenly
saw that almost **90%** of them were just SELECTs on the 'oc_jobs' table::

	SELECT `id`, `class`, `last_run`, `argument` FROM `oc_jobs` WHERE `id` > ? ORDER BY `id` ASC

The 'oc_jobs' table also counted more than 80,000 rows. This was a clear indication that the ownCloud's cron job (cron.php script which is supposed to delete old temporary files and clean
the 'oc_jobs' and other DB tables) didn't do its job properly. Cron job was configured to be
executed by the system cron. It was executing every 15 minutes, but it did nothing.

Then we realized that this was because of an existing lockfile placed inside ownCloud's data
folder. This must have been left there for a long time by some previous cron job run (which must 
have ended really badly). We however didn't isolate the source of those SELECTs. But after we removed that lockfile and ran the cron job again, there were only 2 records left in the 'oc_jobs' table. Since then, ownCloud's overall performance went dramatically up and is up to our expectations.

FIXME: opet bych v posledni vete zduraznil, ze i pri 2500+ uzivatelich na
jednom stroji

