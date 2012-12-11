Title: Oracle Backup & Recovery Training
Author: Neil Holmes
Date: 11/12/2012

# Oracle Backup & Recovery Training

1. [Intro][]
2. [Oracle Architecture][]
3. [Using RMAN][]
4. [RMAN Recovery Catalog][]
5. [Backup Concepts and Strategies][]

100. [Notes][]

## Intro
* Grid Control requires Oracle Web Logic to be installed
* 3 day course
* Custom course, mainly thestandard Oracle course, but with some customisations
* Oracle practices are not very real-life oriented. It tends to descrive what the problem is beforehand, so there's no figuring out what's broken

## Oracle Architecture
* Goal is decreasing MTBF (mean time between failures)
* Decrease MTTR (mean time to recover)
* Categories of failure:
	* Statement failure
	* User process failure
	* Network failure
	* User error
	* Instance failure
	* Media failure
* We're not sure who will be defining our backup/recovery strategy
* Statement failure: This is failure of a single statement. This could be a select or otherwise. E.G. creating an index. 1/2 way through you run out of disk space, this will throw an error. How is this cleaned up? Oracle will automatically roll back the changes. If a statement fails to complete Oracle always automaticlly performs a Statement Level Rollback. Nothing to do with backups here
* User Process Failure (more accurately; session failure): This is user or session process stopping or comms between the two going down. Insert/update/delete are not comited until specifically commited. In cases where one of these errors happens, the session process may be dead, so cannot roll back the tran. It's PMON (process monitor) that would roll this back. Nothing to do with backups here
* Network failure: Listener failure, or anything else that affects communication. Again, nothing to do with backups
* User error: User inadventently deletes data and commits, or drops a table. (*NOTE:* Drops (and DDL) is automatically commited on completion). In case of a dropped table, the table is not deleted, but renamed and hidden (this is called recycle bin). We can get tables back from the recycle bin using [flashback][]: ``flashback table XX``. If a long time has passed, the table may no longer be in recycle bin. You can't restore a single table through RMAN, but you can if you have exported the table. If you don't have an export you can, of course, do a point-in-time recovery but this will have to roll back the WHOLE DB to an SCN before the drop, at this point you could take an export of the table, roll everything forward then restore the table from the export copy
* Instance failure: Say the instance dies. This is a loss of memory, not disk. When starting the database again it will perform automatic instance recovery. See [Overview][] for more.
* LGWR writes when the redo log buffer is a third full. The writes are not instantaneous, so we need more room in the redo buffer to allow for redo to be written to memory, whilst the old redo is written to disk. LGWR also writes every 3 seconds, this is because DBWn fires every 3 seconds to update the data files from redo. LGWR must ALWAYS fire before DBWR.LGWR also writes at commit time and before any clean shutdown
* **NB** Roll forward is done in mount mode, roll backs are done in open mode
* You can set the MTTR target, the time needed to recover. The default is 0 (disabled). This is configured in the HTTR advisor in EM or with fast_start_mttr_target. Why would you set this to less than 1 second? This would take a lot more IO. The lower the value, the more often DBWn must update the data files
* * Media failure: This is the key area where we need backups. This is failure of a disk drive, controller, corruption etc.
* **Question:** Are we multiplexing control files and online redo logs? I don't think we are. It might be worth doingto guard against human error

## Using RMAN
* RMAN is accessed in line mode with the rman command
* ``rman target /`` or ``rman target sys/password`` or ``rman target sys/password@tnsalias`` (N.B. No ``as sysdba`` is necessary)
* RMAN relies on the controlfile for the target DB. The key areas it needs are block 1 (the header block), the data file descriptors (pointing to the data files), online redo log descriptors, the archive log history. There are also 2 RMAN specific areas. RMAN backup history is what automates RMAN, when a backup is taken a note of where the backup was saved is stored here. There is also RMAN configure settings, the RMAN preferences for this target database
* The ``list backup`` command shows the backups of the connected database
* You can set a retention period for backups. The ``delete obsolete`` command can be used to clear backups older than this period. This command is run when the FRA becomes full
* Expired backups are backups that RMAN can't find the file corresponding file for
* RMAN maintains it's own measure of how much space is used by backups, and this might differ from what the OS thinks. You can delete the files through Windows but RMAN won't know that the backup is deleted and won't see the newly freed space
* You can see which backups are obsolete with ``report obsolete``, this will not take any actions. ``delete obsolete`` will actually delete any backups in the report. This will both update the internal stats on space, and delete the file
* The ``report need backup`` command tells you which backups you need to meet your set retention policy
* You can issue more complicated RMAN commands with ``run {}``. Some commands require this run block, for example ``allocate channel``
* If you need to access the files in an ASMinstance manually, you can use ``asmcmd``
* Restore defaults to the file's previous location, but you can put the file elsewhere my using ``set newname for datafile X to '/new/path'``, but this must be contained withing a ``run {}`` block to keep scope
* If you have a run block with several commands, such as backup db, then delete obsolete; then if an earlier command doesn't complete, then later commands don't execute. You'd use scheduler to build more complex workflows
* Backupset: zipped backup - compressed to save space
* Image copy: backup uncompressed, will be the same size as the db
* RMAN backs up all data files, control file and spfile
* RMAN can be cnfigured to use parrallelism to backup and restore by using different channels, this will create a backupset for each channel however
* An RMAN command worth setting from the start is ``CONFIGURE CONTROLFILE AUTOBACKUP ON;``. This will back up the control file any time a change is made to it. This also means that we'll get a backup fo the controlfile and spfile whenever you do a regular backup
* Compressed backups can be done with the ``as compressed backupset`` keywords. Tis should reduce backup sizes to somewhere between a quarter and a third of the uncompressed size
* use the ``show all;`` command in RMAN to see all the options you can configure
* **Specifying a retention policy:** 
	* There are two types of retention policy, either a recovery window, or redundancy (number of backups to keep)
	* ``CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 5 DAYS`` will ensure you will always have a backup that can take you back to 5 days ago
	* The two options are exclusive, you can only have redundancy or recovery window
	* You can't set all the settings you'd like using the CONFIGURE options. You can create more complicated policies using ``backup [whatever] keep until time 'sysdate+60';``, this will override the retention policy and only be marked obsolete in 60 days
	* Backups exempted like this cannot be in the FRA, they can be in ASM however
* Practice 2-3 and 2-4

## RMAN Recovery Catalog
* An Rcat isn't necessary
* In RMAN hot backups (online) are much easier to do than cold, as RMAN needs access to the control file. The process would be:
	* shutdown immediate
	* startup mount
	* backup using rman
	* open 
* RMAN will not let you do a hot backup, unless you're in archivelog mode
* An Rcat is a schema inside an exisiting databse that contains a copy of all the RMAN details stored in control files copied to the db
* The processof adding a DB to an Rcat is called registering
* This is useful if you lose your controlfiles
* You can register multiple databases with a single RCAT
* The RCAT can store RMAN scripts
* RMAN only keeps backup history in the control file for a number of days specified by CONTROL_FILE_RECORD_KEEP_TIME, the default being 7 days. RCAT will keep this information for as long as you need
* If you have an RCAT or not, it's worth changing the CONTROL_FILE_RECORD_KEEP_TIME to as great as the retention policy says the odest backup should be or longer since there's no real downside barring  slightly bigger control file
* Setup process:
	* Create a tablespace for RCAT, maximum size is 50MB per database managed
	* Create user with default tablespace set to the one we created above with 
	* This user will need connect privs, and the role "Recovery Catalog owner"
	* Set up the catalog by first loggin in with ``rman catalog rcatowner/pass``
	* then run ``create catalog;`` which will create the tables and views necessary for RMAN
	* These table will remain empty unti ldatabases are registered
* RCATs have  version based on the RMAN version used to create them
* If you upgrade Oracle the RCAT become obsolete. 
* To register a database ``rman target / catalog rcatowner/password`` then issue ``register database;``
* You can supply EM the RCAT details so that EM backups are recorded there too
* If your RCAT becomes stale, there's no real need to worry as you can resync it manually with ``RESYNC CATALOG;``, and this will be run automatically if you run a command that requires an up to date catalog
* RCAT should be resynced as often as your controlfile_record_keep setting specifies
* Practice 4-1 to 4-3

## Backup Concepts and Strategies
* 3 options for backups:
	* RMAN - obv.
	* Oracle Secure Backup - Cost option. Iterfaces with tape MML to run tape backups
	* User Managed Backup - Using cp or ftp to make copies of the files
* DBV - a tool to find corrupt blocks, you can then use RMAN to recover them
* You can choose:
	* Whole - entire db
	* Partial - Part of the db
* Backup types:
	* All blocks in chosen files (full)
	* Only changed blocks (incremental)
		* Cumulative (changes since last level 0)
		* Differential (changes since last incremental)
* Backup mode:
	* Online (hot) - inconsistent
	* Offline (cold) - consistent
* A backup can be stored in 2 formats:
	* Image copy:- Ilke copying the raw files, backs up all blocks as raw data even empty blocks (use keywords ``as copy``)
	* Backupset:- copies files in a proprietary Oracle format, tars the files and doesn't store empty blocks (``as backupset``)
* Backupsets can also be compressed, which only backs up the used space in each used block
* Image copies can only be created on disk, not on tape
* The only advantage of an image copy is that they don't have to be restored, you can just switch the image copy to live
* Incremental backups:
	* Level 0 incremental backup backs up the whole shebang
	* A cumulative incremental level 1 backup contains blocks changed since the last level 0
	* A differential incremental level 1 backup contains blocks changed sincethe last backup of any type
* You can't have full and incremental backups on a single database, instead we call full backups on an incremental system level 0 backups
* 

# Notes

### Flashback
There are 3 types of flashback; table, database and drop. All work in different ways

###Shutdown abort
PMON kills all other processes, closes the files (not cleanly), writes a signle line to the log and kills itself

###Overview
Say we have 2 sessions. Someone updates smith to jones. We have a server process that will either read the block from the buffer, or pull it into the buffer. This will generate redo and undo, so it allocates a block from the undo tablespace (undo will not need to be read from disk, as it will already be in memory). The process wites the pre-update value to the undo buffer (in memory), then updates the row in the table buffer (with an exclusive lock), then generates a redo entry which stores both the old and new values to the redo log buffer. This is not commited yet, and all changes are only in memory. Session 2 now updates another row in the same table, and the same process is carried out. Still nothing has been written to disk. Commits are quick as only 2 things happen, an entry is written to the redo log buffer, and LGWR (log writer) is invoked to write the redo buffer to disk (the current log file). When a commit happens, it will write the whole redo log buffer to disk, as a single large IO is quicker than picking just the bits that need to be written. At a later point DBWn (database writer) will write the changes to the data files. Say session 1 commits, then the instance crashes. At this point the data files may be inconsistent with what we have commited. When we start the instance again, we startup nomount, then startup mount and reads the control files. The control file will have a flag that says 'needs emergency recovery' and is set to YES when the instance starts. A normal shutdown changes the flag to NO. If the control file lfag is set, the instance will read the online redo logs on disk serially, reading the changes. The reader can't know if a change is comitted, as it can't read ahead. The recovery process will begin to write changes to disk, keeping previous values to the undo (on disk, NOT memory). Session 1's change is committed, so it will dump the before value from the undo file. Session 2's changes have not been comitted, so at the end of the recovery session 2's changed value will be on the live data file, and the old values in undo, so the recovery process will pull the old value form the undo file and squirt it into the live data files. At this point recovery is complete. Non of this needs archive logs or backups.

###Checkpoint###
Really only comes into things on a log switch. It writes the ending SCN of the outgoing log to the header of all the data files. This helps automate recovery. It will compare the file's SCN to the current SCN to decide if it needs recovery, and which archive logs will need applying to the file as well.

###to_char###
Use ``select to_char(current_scn) from v$database;`` to return the literal value of the SCN, rather than 4.35e+8 or what have you.

###Alert log###
Can be found by parameter background_pump_dest. Super handy for looking up what's going off on the instance.

###Take-Away Questions for CACI###
* Why not multiplex the spfile & online redo logs?
* Are we in maxperform or maxprotect?
* 