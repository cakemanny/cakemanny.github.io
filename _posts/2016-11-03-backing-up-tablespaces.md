---
layout: post
title: Backing up MySQL tablespaces
date: 2016-11-03
---

If you are running your MySQL 5.6+ database on Windows or macOS get excited! Or
if you are running linux -- you sensible people -- pity us while you run
percona xtra-backup.

Anyone who has a large MySQL database knows that mysqldump just about works for
doing backups but restoring is a forever-taking experience.
Question: wouldn't it be quicker to `cp -r` / `xcopy` our database? The
server has already spent a year putting our data into innodb pages... those are
good enough for me right?

Hold your horses! InnoDB gains durability from writing transactions to the redo
log, the current state of the on-disk data in the idb tablespace files is not
likely to be consistent as InnoDB can choose the order it writes dirty pages
from the bufferpool to the tablespace.

Luckily, MySQL 5.6 introduced transportable tablespaces.
This only works for tables with their own tablespace innodb\_file\_per\_table=1,
and not for tables with full text indexes.

Backing up
----------
```
mysql> FLUSH TABLES `tableA`,`tableB` FOR EXPORT;
```
This forces dirty pages in the named tables to disk and creates a tableA.cfg
file alongside your tableA.ibd

```
macOS $ cp $datadir/dbname/tableA.{cfg,ibd} $backupdir/.
win32> copy %datadir%\dbname\tableA.cfg %datadir%\dbname\tableA.ibd %backupdir%\.
```

We also need definitions of the tables which match exactly:

```
$ mysqldump --routines --events --no-data --result-file=$backupdir/dbname.defs.sql
```

Now we can tell MySQL we are done

```
mysql> UNLOCK TABLES;
```

**NB:** Using the `--master-data` option will leave the mysqldump deadlocked
with your flush tables lock, so avoid that one.

### I said I had a big one! ###
Isn't this going to cause my server to grind to a
halt whilst I copy TBs of data? Oh, yes, sorry...

Windows users I have a solution for you, VSS snapshots.
You can flush tables, create a snapshot and then release your locks.
VSS snapshots either complete in 10 seconds or fail, so if you can spare a 10
second hiccup at 4am then

On your Windows 2008 or later server you have `diskshadow.exe` which can be
scripted.

If you are a developer, Windows Vista SDK or later includes `vshadow.exe`.
If you have, for example VS2015 installed,
    C:\Program Files (x86)\Windows Kits\...
is the place to look.
VShadow can run a bat file while a temporary snapshot exists.

Suppose you have a `mysql -e"flush tables...for export;do sleep(1000000)"` running
somewhere

```
> type backupshadow.cmd
:: somehow tell MySQL to UNLOCK TABLES... - we'll improve later
taskkill /IM mysql.exe
call setvar1.cmd
copy %SHADOW_DEVICE_1%\%datadir%\dbname\tableA.cfg %backupdir%\.
copy %SHADOW_DEVICE_1%\%datadir%\dbname\tableA.ibd %backupdir%\.

> vshadow.exe -script=setvar1.cmd -exec=backupshadow.cmd
```

Or if you are a lowly mortal on a desktop operating system without the Win SDK,
[ShadowSpawn](https://github.com/candera/shadowspawn) provides similar
functionality with a simpler syntax.

```
> type backup2.cmd
taskkill /IM mysql.exe
copy S:\tableA.cfg %backupdir%\.
copy S:\tableA.ibd %backupdir%\.

> shadowspawn %datadir%\dbname S: cmd /c backup2.cmd
```

Restoring
---------
Couldn't be simpler! We need to create the tables with identical table
definition, discard the created tablespace and import the backup.

```
$ mysqladmin create restore_db
$ mysql restore_db < dbname.defs.sql
```

```
mysql> use restore_db
mysql> alter table `tableA` discard tablespace;
```

You may need to correct permissions after copying the files in.

```
$ cp $backupdir/tableA.{cfg,ibd} $datadir/restore_db
$ chown -R _mysql $datadir/restore_db
$ chmod -R 600 $datadir/restore_db
```

```
mysql> alter table `tableA` import tablespace;
```

A log message is written to indicate that we were successful:

```
$ tail $datadir/testhost.err
2016-11-03 18:25:20 3251 [Note] InnoDB: Importing tablespace for table 'dbname/tableA' that was exported from host 'testhost'
2016-11-03 18:25:20 3251 [Note] InnoDB: Phase I - Update all pages
2016-11-03 18:25:20 3251 [Note] InnoDB: Sync to disk
2016-11-03 18:25:20 3251 [Note] InnoDB: Sync to disk - done!
2016-11-03 18:25:20 3251 [Note] InnoDB: Phase III - Flush changes to disk
2016-11-03 18:25:20 3251 [Note] InnoDB: Phase IV - Flush complete
```

Obviously there are some considerations.

- You most likely have more than one or two tables in your database
- MySQL data files are not simply named if characters outside [A-Za-z0-9_] are used

I will shortly upload some python scripts that do all of this in one fell swoop.

