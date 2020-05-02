# rsyslog-omprogs

This is a group of scripts that can be used with the [rsyslog omprog output
module][omprog] (where it pipes matching syslog messages as stdin to a
program). The scripts are (currently all) written in Perl.

The currently implemented scripts:

 * omprog-sqlite: send messages to sqlite, with database normalization
 * omprog-graphite-nsd: send stats nsd prints in the syslog to graphite
 * omprog-graphite-unbound: send stats unbound prints in the syslog to graphite

There is also helper utilities:

 * rsyslog-db-viewer: helper script to query the log database from omprog-sqlite

### Dependencies

All scripts depend on the following Perl modules:

 * JSON

In debian/ubuntu:

    apt-get install libjson-perl

omprog-sqlite (and rsyslog-db-viewer) additionally depends on:

 * DBI
 * DBD::SQLite3

In debian/ubuntu:

    apt-get install libdbi-perl libdbd-sqlite3-perl

## omprog-sqlite

Instead of a flat list of syslog messages as is easily provided by the
omlibdbi module distributed with rsyslog, we split it up to a group of
tables:

 * hosts
 * facilities
 * programs
 * severities
 * events

There is also a view to simulate s non-normalized table, called "msgs".

It's a quick hack. I wouldn't disable regular logging mechanisms when using
this, but maybe it turns out to work out fine. Let's wait and see!

What this allows us to do stuff like:

    SELECT program, count(msg) FROM msgs GROUP BY program;

(Admittedly, we have always been able to do this.

    awk '{print $5}' /var/log/syslog | sed -re 's/\[.*//' | sort | uniq -c | sort -n

But if you don't see any potential use cases for having the messages in an
sql database, you are not the intended audience. Thank you for having shown
interest.)

### Table definitions

These are rough outlines, for illustrative purposes; the exact
create statements used can be found in the source code.

facilities and severities are created as:

    CREATE TABLE facilities (
      id INTEGER PRIMARY KEY,
      name VARCHAR(16) UNIQUE
    )

programs and hosts are created as:

    CREATE TABLE programs (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name VARCHAR(128) UNIQUE
    )

events is created as:

    CREATE TABLE events (
      host INTEGER NOT NULL REFERENCES hosts(id),
      facility INTEGER NOT NULL REFERENCES facilities(id),
      program INTEGER NOT NULL REFERENCES programs(id),
      pid INTEGER,
      severity INTEGER NOT NULL REFERENCES severities(id),
      msg TEXT NOT NULL,
      time INTEGER NOT NULL,
      registered INTEGER NOT NULL
    );

### Notes

Currently, there's a naive approach for trying to upsert new entries in the
tables. If we get for instance a severity id from rsyslog that we haven't
seen before, we will insert the new id and associate it with the name
supplied by rsyslog (i.e. the first one wins when it comes to naming the
{severity, facility} ids). For programs and hosts, the situation is
similar; but instead of being fed a numeric id from rsyslog, we invent it
on our own by asking sqlite for it (using the AUTOINCREMENT mechanism).

When working with a single sqlite3 database, you deal with the same
parallelism constraints that you would when dealing directly with sqlite3.
You should therefore probably not run two omprog-sqlite at the same time
(when working against the same database). This means that you can't have
several rsyslog actions that all point to the same database. You should
instead either create rsyslog rulesets or use mutliple databases.

The script is implemented with the DBI perl library for the database
interaction. DBI has support for a wide array of database drivers but only
sqlite3 is currently supported. Also supporting other drivers should be
easily doable. Currently, the limiting factor is the SQL statements used,
specifically the create statements. I would happily accept changes that
makes the sql statements used be inputted to the script somehow (e.g. a
configuration file), so that the implementation isn't coupled to sqlite.
But if contributing upstream "isn't you", just hack the script and adapt
the SQL and you should be fine.

I also implemented a simple "viewer", where you can query by
host/facility/program/severity (see rsyslog-db-viewer).

    # possible filter flags: facility, severity, program, host
    rsyslog-db-viewer --program rsyslogd

    # For severity, use --min-severity to show msgs of at least this level
    rsyslog-db-viewer --min-severity notice

    # Filter further by adding more commands; they are "AND:ed togheter"
    rsyslog-db-viewer --facility kern --min-severity err

    # List the available filterable data keys:
    rsyslog-db-viewer --list-hosts
    rsyslog-db-viewer --list-facilities
    rsyslog-db-viewer --list-programs
    rsyslog-db-viewer --list-severities

Finally, a simple rsyslog.conf fragment is included as an example.

### Alternatives

If all you want to do is putting messages in a database, you can do away
with having to depend on some random internet nobody's perl scripts (I
trust myself, but I'm biased!):

* [**omlibdbi**][omlibdbi]
* [**ommysql**][ommysql]

These database output modules are maintained and distributed with rsyslog.
A safe bet.  But they don't normalize the data. Maybe you can insert into a
view for the same effect, but I don't know how handling insert-if-missing
would work.

### TODO

 * Logging database errors (! this one is important, maybe?)
 * Query events by time ranges
 * migrations/versioning of schema
 * DB cleanup
 * Colors? (for the viewer, maybe not the storage backend.. :))
 * Automated testing
 * Stress testing; how will the script handle bursts?

## omprog-graphite-...

I've written several omprogs to extract statistics sent to syslog
by various services and forward them to graphite.

To enable them in rsyslog, first load the module and define a template:

    module (load="omprog")
    template(name="graphite_forward" type="string" string="%HOSTNAME%#%msg%\n")

The scripts expect lines prefixed with the hostname of the message origin,
followed by a `#` and then the message.

Also make sure that you configuration unbound/nsd to output metrics at a
rate compatible with the minimum retention period in carbon, or else you'll
see weird values/gaps. For instance, for nsd, you add the following section
to carbon's storage-schema.conf (first match from the top wins):

    [nsd]
    pattern = ^nsd\.
    retentions = 60s:90d

Then, you should add a matching statement in nsd.conf (i.e. dump stats once
every minute):

    server:
      statistics: 60

And similarly, for unbound (let's say we want a granularity of 10 minutes
instead), here's the section of carbon's storage-schema.conf:

    [unbound]
    pattern = ^unbound\.
    retentions = 600s:90d

and here's the setting you want in unbound.conf:

    server:
      statistics-interval: 600

Currently the scripts reuse some code by means of duplication, which is...
fine? I guess that's where all my security holes will be found. But it does
simplify deployment, as you just need to drop one file on the target per
omprog (plus rsyslog config).

### omprog-graphite-nsd

Extract the statistics nsd prints to syslog and send them to graphite.

Enable in rsyslog with a config like this (change the path to the
script to match your environment):

    :programname,isequal,"nsd" action(type="omprog" binary="/sbin/omprog-graphite-nsd" template="graphite_forward")

or

    if $programname == 'nsd' then action(type="omprog" binary="/sbin/omprog-graphite-nsd" template="graphite_forward")

(they are equivalent; the former is objectively prettier, the latter is
more expressive.)

You can filter further to only match the actual stats log messages, but I
leave it as an exercise for the reader. (Do this excercise if you expect
nsd to log a lot of non statistics messages.)

Note that the statistics nsd prints out to syslog is **cumulative**, i.e.
the values don't reset but keeps on growing until nsd is restarted. When
using nsd-control, you can use the `stats` or `stats_noreset` subcommands
to affect this behavior, but when depending on the syslog mechanism, I
haven't found a way to change this. Instead, use the `derivative()`
function in your graphite queries.

Sample input:

    example#NSTATS 1588369800 1588368963 A=8 AAAA=7

goes to `nsd.example.rtypes.*`

    example#XSTATS 1588369800 1588368963 RR=0 RNXD=0 RFwdR=0 RDupR=0 RFail=0 RFErr=0 RErr=0 RAXFR=0 RLame=0 ROpts=0 SSysQ=0 SAns=15 SFwdQ=0 SDupQ=0 SErr=0 RQ=15 RIQ=0 RFwdQ=0 RDupQ=0 RTCP=0 SFwdR=0 SFail=0 SFErr=0 SNaAns=0 SNXD=0 RUQ=0 RURQ=0 RUXFR=0 RUUpd=0

goes to `nsd.example.metrics.*`

### omprog-graphite-unbound

Extract the statistics unbound prints to syslog and send them to graphite.

    :programname,isequal,"unbound" action(type="omprog" binary="/sbin/omprog-graphite-unbound" template="graphite_forward")

or

    if $programname == 'unbound' then action(type="omprog" binary="/sbin/omprog-graphite-unbound" template="graphite_forward")

(they are equivalent; the former is objectively prettier, the latter is more
expressive.)

You can filter further to only match the actual stats log messages, but I
leave it as an exercise for the reader. (Do this excercise if you expect
unbound to log a lot of non statistics messages.)

Note that unlike nsd, the statistics unbound prints out to syslog is **not
cumulative**, i.e.  the values *do* reset each time they are printed. This
is ideal behavior for us, so you don't have to do anything special in your
graphite queries. There is however an unbound configuration to change this,
so if you do see ever increasing values, see if `statistics-cumulative:
yes` is set in your unbound configuration. Either change this, or use the
`derivative()` function in your graphite queries, just as for nsd.

Sample input:

    example# [7613:0] info: server stats for thread 0: 89 queries, 84 answers from cache, 5 recursions, 43 prefetch, 0 rejected by ip ratelimiting"

goes to `unbound.example.thr-0.*`

    example# [7613:0] info: server stats for thread 0: requestlist max 67 avg 14 exceeded 0 jostled 0"

goes to `unbound.example.thr-0.reqlist.*`

    example# [7613:0] info: average recursion processing time 0.121216 sec"

goes to `unbound.example.recursion_avg_ms`

# See also

* [RSyslog documentation](https://rsyslog.readthedocs.io/en/latest/index.html)
* Specifically: [Output module: omprog][omprog]

[omprog]: https://rsyslog.readthedocs.io/en/latest/configuration/modules/omprog.html
[omlibdbi]: https://rsyslog.readthedocs.io/en/latest/configuration/modules/omlibdbi.html
[ommysql]: https://rsyslog.readthedocs.io/en/latest/configuration/modules/ommysql.html
