# Git Hooks for mysql

General system description
=======================

The idea of this project is to integrate mysql database into git, in such a way that any time the client make a commit, the changes that have been made in a defined mysql DB get recorded in the repo as migration files. It should include functionality to restore the database to a specific state when the user make checkout, either automatically or manually.

The approach taken, making migration files that just store the changes in the DB instead of recording it's full state each commit, is motivated by the fact that GIT store complete objects (loose objects) in each commit, without taking care of applying delta compression from version to version. 

It's true that git provides functionality for applying delta compression on a "whole repo / multi-commit scope" with the "gc" command, and that the loose objects stored in each commit do get compressed; besides that, considering the files involved in this use case are DB dump files (text format with very repetitive patterns, so get very good compression ratios), it could be apparent that the case for this approach is not that rational, but the idea of creating migration files that just store the changes to the DB is appealing from the point of view of minimizing invasive interaction with the mysql server in the process of restoring system state. In a "full state dump / restore" approach each time the repo is restored to a certain state the DB tables should be dropped and recreated. In a "full state dump / restore with CREATE TABLE IF NOT EXIST and REPLACE instead of INSERT" approach, the DB tables need not to be drop every time state is restored, but the sql server has to act upon every table and record in the DB, which in fact is quite prone to error. Using migration files have the advantage of messing with the actual DB just the needed amount each time, passing the heavy workload to an external process (this software been created), so the risk of introducing errors in the original system, or breaking it all together, diminish, provided that this external process been created get to a stable, trustable state as software.

Recalling GIT issues with objects compression, a valid argument for migration files (other than "it adds a improvement to existing GIT compression mechanisms") would be that the use of "gc" command is neither well known nor readily applied with each commit, so it could perfectly be the case in many user repositories that it never get applied.

Ref:
- https://gist.github.com/matthewmccullough/2695758
- https://git-scm.com/docs/git-gc

The first version of this solution gets help from DBDiff CLI php system, so it depends on a working installation of this software (and of course of php).  Subsequent versions could be improved implementing an algorithm to extract the differences between DB version right in the sql file dumps, using text comparisons, so there will be no need for external dependencies.

The checkout functionality is dependent of the kind of checkout: if it's a linear one, the DB changes to restore will be automatically integrated into the working DB. The term linear checkout means that there is a linear relation between the state of the working copy and the commit been checked out (both are in the same branch, and there is no intermediate branches creation / merging in the history). If it's not a linear checkout then the system should present the user with options to:

1- restore the DB state to one specific version of the DB dump files from one of the merging commits (this to be done automatically), or

2- merge the sql dump files by hand. In this last case is not enough the diff functionality natively existing in GIT, because the history of both branches could had altered the DB state independently, so the only solution would be to let the decision making process in hands of the user.


Use
===

Up to this first version the system is composed by only two hooks files: post-commit & pre-commit. They must be copied in the hooks folder of the git repo that will use this funcionallity. There is a configuration file (dbdiff.conf) that needs to be copied in the root of the git repo. The variable are quite self-explanatory and in any case the help of DBDiff cover them.

Other variables of the system are inside the post-commit script, and are:

_usr="root"                         - The user to access mysql server.

_pass=""                            - The password to access mysql server. Not recommended to store the password here, it's better to use my.cnf as especified in mysql documentation. Anyway the option is given to the user.

_host="localhost"                   # The host where the mysql server is running.

_db="d02bcad8"                      # The DB to check for changes each commit.

_tmpdb=${_db}_mysql_migr_tmp_db     # The name that will be given to the tmp db that will be used for comparison. This is an internal variable and the user is simple given the option to change the format of the last part of that name.

_migrations_path="DB/"              # Where the migration files generated will be stored in the file structure of the git repository.

_dbdiff="../DBDiff/dbdiff"          # Location of the dbdiff program (this generally will be in some central part of the system)

_dbdiff_conf="./dbdiff.conf"        # Location of the dbdiff configuration file (this file contains credentials, so it should be keep in a secure location and with the appropiate access rights)