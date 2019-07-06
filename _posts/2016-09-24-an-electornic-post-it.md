---
title: "(an electronic post-it note)"
---
Alas, the days of "sharing gems" are gone, replaced by the scribbles of a disheveled
(but still statistics-obsessed) mind. But I figure I need to become less self-concious
about sharing my writing, and what better way to start than sharing something that no
one will read!

Anyways, I learned how to use a bunch of new tools this summer, and I might not
work with them for a little while, so it seems worth jotting down some notes for
future reference. But who knows, maybe future Kris will have found better workflows (or
become a mathematician).

## Writing ##

`bookdown` is a potential alternative to `knitr`-`sweave` for writing more involved
statistical reports. It seems to address the main limitations with plain rmarkdown that
had me still writing in `.rnw` -- easy compilation to pdf, cross-referencing,
bibliographies, and actual latex preambles and math. To compile to pdf, use `pdf_book()`;
you can specify a preamble and a bibliography package by including a section like this
in the `_output.yml` file,

{% highlight markdown %}
bookdown::pdf_book:
  includes:
    in_header: preamble.tex
  latex_engine: pdflatex
  citation_package: natbib
{% endhighlight %}

You can include R images as usual by plotting within the code chunk. If you have
a .png, you can write
{% highlight markdown %}
knitr::include_graphics("figures/image.png")
{% endhighlight %}
This is different from sweave, where you'd write ```\includegraphics{}```. A
last tidbit is that a new alternative to `xtable` for including tables built from
R `data.frames` is `kable`. It's main advantage is that it works even if you choose
to compile as an html document (instead of pdf) -- `xtable` just generates the
latex code for a table.

## Emacs ##

This summer probably had the highest emacs-commands-per-day learning
rate since I first started (it helps to be around a bunch of very good emacs users).
In no particular order,

* You can use MAGIT for git tracking. The main commands I ended up using are
  - `C-x g` opens MAGIT
  - Pressing `s` on unstaged files stages them, `u` unstages staged files. You can
  also stage and unstage blocks with the usual `C-space C-n ...`
  - Commit staged files by pressing `c c` within the MAGIT window.
  - Push your current branch to origin using `P p`.
  - Pull a branch from origin using  `F p`.
  - Create a new branch using `b n`. It will prompt where to branch from and what
  to call it. Beware, this does not switch you to that branch.
  - `b b` lets you switch branches.
  - Use `m` to merge a branch (and `m m` to merge with master).
  - Use `l l` to view the full branching history. From here you can reset to a
  previous commit by moving the cursor to that commit and typing `C-u x`.
  - In the inevitable "I accidentally committed on master" situation, first
  checkout a new branch (`b n`, this saves the current work) then hard
  reset on master (`l l C-u x`) (see
  [this](http://stackoverflow.com/questions/1628563/move-the-most-recent-commits-to-a-new-branch-with-git)).
* [helm](https://github.com/emacs-helm/helm) is nice for file system navigation (and
can be installed with the usual `M-x package-list-packages`).
* There's a SQL mode which can evaluate code blocks in a SQL process in another
window (kind of like ESS).
	- Use `M-x sql-postgres` to start a postgres process in emacs. There are analogous
	commands for other databases.
	- To avoid typing in credentials each time you log in, you can put this in your `.emacs`
	(with defaults filled-in),
{% highlight elisp %}
(sql-postgres-login-params
	(quote
    ((user :default "")
     (password :default "")
     (server :default "")
     (database :default "")
     (port :default 1234)))))
{% endhighlight %}
* You can use `M-x replace-regexp` to do find-replace with regexes. Two nice substitutions are are
`$` and `^` -- this will replace everything at the end and start of the highlighted block,
respectively (e.g., you can add commas to the end of each line).
* `M-;` can comment / uncomment blocks.
* `C-x r t` can be used to insert the same string across a block.
* `C-_` undos, and `C-g C-_` redos.
* When editing markdown files, you can insert links with `C-c C-a l`, cross-references
with `C-c C-a L`, code blocks with `C-c C-s P`, and headers with `C-c C-t {number #'s}`.

## Postgres ##
I hadn't written much SQL before, so this was all new. Hopefully you won't have
forgotten `SELECTS` and `WHERE`'s by the time you read this again, but there were
a bunch of smaller things that do seem worth writing somewhere.

* Creating indices will dramatically speed up joins. You create one on an existing
column using `CREATE INDEX index_name ON schema_name.table_name(column_to_index)`.
You can drop it using `DROP INDEX IF EXISTS schema_name.index_name`. Every once in a
while, it's good to `VACUUM` tables, to keep the database from bloating.
* You can manipulate GIS data using POSTGIS.
  - You can upload shapefiles to a postgres database using `shp2pgsql`.
- You can get the latitude / longitude of the center a polygon (in a column called geom)
using
{% highlight SQL %}
SELECT ST_Y(ST_CENTROID(geom)) AS latitude
ST_X(ST_CENTROID(geom)) AS longitude
FROM schema.table;
{% endhighlight %}
- You can check whether one geometry is within a certain distance of another using
`ST_DWithin`.
- You can switch coordinate systems using `SELECT ST_Transform(ST_SetSRID(old_geom, old_coord), new_coord) AS new_geom`.
- You probably want to use [4326](http://spatialreference.org/ref/epsg/wgs-84/) for the new coordinate system.
- [QGIS](http://www.qgis.org/en/site/) makes it relatively straightforwards to visualize spatial
data in a database. Just right-click the postgres elephant in the browser panel and type in
the login credentials. Then, you can visualize layers from the layers panel.
* You can use `to_sql` and `from_sql` in `pandas` to fetch and save database tables without
leaving python. For larger transactions though, it's much faster to use `\copy` or `psql`;
[odo](http://odo.pydata.org/en/latest/perf.html) also seems useful (though I never got around
to using it). For more general operations, `psycopg2` and `sqlalchemy` are available. For example,
for selecting rows that meet a condition, you could write (assuming `params` has the login info)
{% highlight python %}
psycopg2.connect(params)
cursor = connection.cursor()
cursor.execute("SELECT * from %s.%s %s" % (
    schema,
    table,
    filter
))
field_names = [i[0] for i in cursor.description]
{% endhighlight %}
* To copy a database table to a csv, you can use `psql -F","`, followed by the usual `-h`,
`-p`, `-U`, and `-d` options (host, port, user, database) you would use to login to the
database.
* You can dump a postgres table using `pg_dump` (make sure to set the `--no-owner` flag if
you want to restore it on a new machine).
* You can restore a postgres dump using `psql -f name_of_dump.sql`.
* Don't try to store model binaries in a database not beacuse the database can't handle it,
but because `psycopg2` sometimes fails on the import (or export, I can't remember). Instead,
just load the path to binary in the database, and hope the link doesn't go stale.
* You can use `explain analyze` to check if what you were about to start would take until
the heat death of the universe.

## Luigi ##

We spent a while this summer writing an ML pipeline in [luigi](https://luigi.readthedocs.io),
and it's probably out of scope to write any detailed examples here. But a basic summary is
that each `requires()` method in a `Task()` class implicitly specifies a file / database
table entry that checks to see whether previosu steps have been accomplished. The `output()`
method specifies the file / `table_update` that dependent tasks will check to see if the
current task is complete. I say implicit because you just write the name of the required
tasks in `requires()`, and luigi knows to check for the file in each of those tasks'
`output()` method. Of course, the `run()` method does all the actually interesting
computation.

The only other points I can think of are
* It's generally easier to touch logging files in a shared disk than to use the automatic
table updates, because this gives more control over how names look (so you can experiment
with "faking" added and deleted dependencies).
* It's useful to have a debug-mode version of the pipeline that runs very quickly, because
it can be tedious to debug a long running pipeline.

## Docker ##

Dockerfiles are instructions for setting up Docker containers, which are like lightweight
VMs. Dockerfiles have the feel of a bash installation script, except the environment is
completely controlled (you know exactly what the OS is and which programs are installed).
So, it's more reliable to to share docker images than installation scripts.

* `docker ps` checks the running containers.
* `docker images` lists all the local images (which might not be associated with running
containers yet).
* `docker run user/image` runs an image from dockerhub, and `docker run imageid` runs from
a local image.
* You can "get inside" a running container using `docker exec -it {docker process id} bash`.
* You can use `docker-compose rm` to remove unused containers.
* You can change the directory where images are built by changing the `DOCKER_OPTS` environmental
variable in `/etc/`. This is useful if you have one partition (`/mnt/`) with a lot more space than
where Docker will usually save images.

There were a few points that tripped me up, so keep this in mind,
* `docker-compose` let's you run several containers that speak to each other through exposed
ports. Make sure to run docker in `-network="bridge"` mode, note `"host"`, if you want to
use this (because `host` mode
[doesn't support inter-container communication](https://docs.docker.com/engine/reference/run/#network-settings)).
* When you write a Dockerfile, changing users [doesn't change permissions when using COPY](https://github.com/docker/docker/issues/6119).

## Misc. ##

- `screen` isn't so complicated after all.  If you want to run something that will take a long time, log into
  the cluster, create a new screen using `screen -S name-of-screen`, start the process you want,
  and type `C-a d` to detach the screen. Now you can log off the cluster, and it'll keep on running.
  To see what is happening again, log back on and type `screen -r name-of-screen`. You can see all
  the running screens using `screen -ls`, and you can remove old ones with `screen -X -S name-of-screen quit`.
- There's a python [package](https://github.com/googlemaps/google-maps-services-python)
for sending geocoding queries to google maps.
- You can create new EC2 instances, RDS databases, and S3 buckets from [here](us-west-2.console.aws.amazon.com/console/),
though it's probably better to stick with Sherlock while at Stanford.
- You can see the sorted disk usage usign `du -ch | sort -h`, or to shame people for using
too much space, `du -sch | sort -h`.
- It's helpful to setup aliases for cluster logins that you use all the time.
- You can see the few lines around grep matches using the `-C` flag, e.g., `grep -C 4 'search' .`shows
the 4 lines before and after each line containing `search`.
- You can view all the processes matching some string using `ps -fea | grep name`.
- You can kill processes with specific PIDs using `kill -9 pid`.
- To view all the processes sorted by memory / compute usage, you can use `htop` (instead
of activity monitor).
- You can't just copy id_rsa's and expect to have permissions -- you need to use ssh-keygen.
- If you want to impress people with no effort, use `DT` in Shiny. Actually, same comment
applies to QGIS.

Sigh, my writing has become insufferable, but that's OK, see paragraph 1.
