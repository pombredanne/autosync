Ever have a workflow that requires a copy of a directory be maintained
in another location?  Perhaps you created a cronjob task that looks like
this::

* * * * * s3put -w -b my_bucket /home/me/stuff

Copying entire filesystems from point A to point B is a common system
anti-pattern.  Most tools that solve this are procedural:
their semantics state that files are to be copied from target to
source.

Better would be declarative approach.  We should only need to specific
that a target needs to be kept in sync with a source and trust the
underlying tool to figure out how that will happen.

Autosync is such a declarative tool copy.  Functionally it is simialr
to ``while true; do s3put ... ; sleep 1; done`` but far more efficent.
efficient.  It uses inotify to track changes instead of making
multiple passes; a full directory scan is only performed once upon
start instead of iteratively.  Multiple threads and concurrent
transfers are used to hide network I/O latency.

Using autosync
===================

Installation
------------

Autosync is a registered package in the pypi repository; it can be instlled with pip::

pip install autosync

Running
-------

Running autosync requires only a source directory path and infromation
about the desired target.  By default autosync writes to S3; the
configuration for an S3 uploader can be as simple as::

autosync --target_container=my_s3_bucket --source_prefix=/var/data /var/data/files

In this example files are copied from `/var/data/files`, `/var/data`
is stripped from each file's absolute path and this file is written to
my_s3_bucket.

Other targets can be specified with the actor parameter.

 --actor: The full name of the module and class use to generate a
   connection to the target. So if you created a connection object
   called my.foobar.Connection you would type
   --actor=my.foobar.Connection. Moduled included with autosync
   (i.e. autosync.actors.*.Connection) can be abbreviated with the
   base module name (i.e. --actor=s3) (default: 's3')

Source filter restricts which of the files in the source directory are copied.

 --source_filter: Only accept files that match this regex (default:
   '^.*$')

Most backend systems have some notion of drives, containers, buckets,
repos - whatever its called, it goes into this parameter.

 --target_container: The remote bucket to write into. This would be an
    S3 bucket, rackspace container, rsync host and module or maybe
    Drive letter in Windows


Configuration
-------------

Autosync has no configuraiton files.  It does use boto and depends upon the credentials in `~/.boto`.  

Prefixes
============

One of the most important concepts to keep track of when configuring
autosync is the "source_prefix".  Folks who use s3put a lot will
already know about the -P flag, but I'll review it anyway.

Imagine you have a directory structure hanging off of your root
directory that looks like this::

/home/adeprince/mystuff/1
/home/adeprince/mystuff/a/2
/home/adeprince/mystuff/a/b/3
/home/adeprince/mystuff/c/4

If you type this::

# cd 
# tar -cvzf mystuff.tar.gz mystuff

you will create a tar file laid out like this::

mystuff/1
mystuff/a/2
mystuff/a/b/3
mystuff/c/4

But if you type this instead::

# cd ~/mystuff
# tar -cvzf mystuff.tar.gz * 

you will create a tar file with a layout that looks like this::

1
a/2
a/b/3
c/4

With tar and zip the currect directory defines the root of your tree.

In autosync this root is made explicit.  When running autosync if you say::

autosync mystuff

the resulting tree structure will look this::

/home/adeprince/mystuff/1
/home/adeprince/mystuff/a/2
/home/adeprince/mystuff/a/b/3
/home/adeprince/mystuff/c/4

Probably not what you want.

To stop this you add what is called a "source prefix".   So if you say::

$ autosync --source_prefix="/home/" mystuff

the resulting file system will look like this::

adeprince/mystuff/1
adeprince/mystuff/a/2
adeprince/mystuff/a/b/3
adeprince/mystuff/c/4

You can get the same automatic behavior of more traditional tools; if
you are working from bash you might say::

autosync --source_prefix=$PWD/mystuff/ mystuff

which will give you a file layout that looks like this::

1
a/2
a/b/3
c/4

Note the trailing slash.  If you omit the trailing slash and write::

autosync --source_prefix=$PWD/mystuff/ mystuff

which will give you a file layout that looks like this

/1
/a/2
/a/b/3
/c/4

Depending on your backend this could mean something very different.
It doesn't matter for S3, but it does for WAAA because inside of puppy
these values might be passed to os.pathjoin and used to generate urls.
If the second parameter in os.path.join starts with a /, the first
parameter is omitted.  So::

>>> import os.path
>>> os.path.join('a', 'b') 
'a/b'
>>> os.path.join('a', '/b') 
'/b'

Don't accidentally convert something to an absolute path if you don't
mean to.

While you might not want your files to be in the /home/adeprince
directory on S3, you might also not want them to be in the root
directory.  No problem, autosync also supports target prefixes.  These
are added to the front of your path, so if you said::

autosync --source_prefix=$PWD/mystuff/ --target-prefix=/foo/bar/ mystuff

your file structure would look like this

/foo/bar/1
/foo/bar/a/2
/foo/bar/a/b/3
/foo/bar/c/4


Extending autosync
==================

Now lets look at how autosync is extended.  There there three steps to
extending autosync:

Configuration parameters.
-------------------------

You want autosync's main method to know about any parameters that
might be specific to your application.  Fortunately gflags make this
really painlessly easy.

Recall in the above example we need to add two additional parameters
beyond the defaults in autosync: url and secret.  To do this we write::

import gflags
FLAGS = gflags.FLAGS
gflags.DEFINE_string('url', None, 'Base url to connect to (i.e. www.wnyc.org/api/v1/media_file')
gflags.DEFINE_string('secret', None, 'Secret API key')

Now the next step is to create a a class that implements the autosync
actor interface.  This interface requires three methods::

list returns a generator that produces autosync.files.File compatible
objects describing what's already in your datastore.  Recall autosync
is threaded -- if this data is sorted by primary key autosync is able
to perform a merge and determine which files need to be added or
removed before the download of file listings is complete.  To indicate
this the returned generate should have an attribute called
already_sorted and it should be true.

delete accepts a single parameter, key, that is also a File compatible
object.  The primary key of the object to delete is in key.key

upload accepts a single parameter, key, that is also a File compatible
object.  This is used to indicate that the file should be created or
updated.

From this email I omit the complete implementation of
meda_file_actor.py's MediaFileActor class, but you are welcome to look
in puppy/scripts.

Lastly you need to "register" your actor with autosync and run
autosync.  You do this with the following code snippet::

if __name__ == "__main__":
    main(actor=MediaFileActor)
