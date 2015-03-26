## Introduction

gcsfuse is an open-source tool that enables you to interact with Cloud Storage objects using traditional Unix tools and existing software. Specifically, gcsfuse allows you to mount a Google Cloud Storage bucket as a local file system that supports simple read/write use cases and most POSIX semantics. You can use gcsfuse on any Unix-like environment that supports Filesystem in Userspace (FUSE) including Google Compute Engine instances.

You can simultaneously read and write to Cloud Storage through gcsfuse and tools like [gsutil][gsutil-docs]. For example, if you write an object using gcsfuse, it will be available to read with gsutil, or vice versa, without the need to re-mount the bucket or reboot the Compute Engine instance.

gcsfuse is designed for uses cases that involve global aggregation and distribution of mostly-read content:

* Distributing content to many instances, in particular, stateless instances. The content can be read simultaneously and random I/O is supported.

* Aggregating the output from many instances, for example, log files, into one shared location. The shared location supports many writers of data, each writing to individual files.

* Transferring data into or out of Google Cloud Storage can be done by using gcsfuse on both a machine on-premises and a Google Compute Engine instance.

* Improving interoperability between older non-cloud software with newer cloud software.  The older software can use the file based interface while the newer software can use the native REST interface.

For in-cloud, single node file storage use case, we recommend that you attach a [persistent disk][persistent-docs] or [local SSD][ssd-docs] to your VM instead of using gcsfuse.

## Scope

gcsfuse supports all file-oriented POSIX semantics: open, close, read, write, list, stat, and rename. However, there are some use cases where gcsfuse currently works best. Use the following characteristics as guidelines:

<dl>
<dt>Random Reads Performance</dt>
<dd>gcsfuse works best with less seeks and a large amount of data read as compared to many seeks and small amounts of data read. If you are using gcsfuse in the later scenario, you can use local cache to improve performance.</dd>

<dt>Random Writes</dt>
<dd>gcsfuse simulates random writes by downloading the entire object into a cache, letting you edit the local copy, and writing the entire object back. This is functional, but can have low performance for small edits to large
objects.</dd>

<dt>Write Cache</dt>
<dd>Because writes are cached until the file is closed, you can lose data that’s been written and even flushed but not yet saved to Google Cloud Storage, such as for a VM crash after some writes but before file closes.  Readers will not see writes until they are committed which can be arbitrarily far off.</dd>

<dt>Write Concurrency</dt>
<dd>gcsfuse provides close-to-open consistency, but concurrent modifications by other processes, including from the Google Cloud Storage XML or JSON API to the same file (object) result in last write wins.</dd>

<dt>Operation Atomicity</dt>
<dd>Some operations like rename() are more than one non-atomic call that can be impacted by a system failure and leave data in an unexpected state.</dd>

<dt>Identity</dt>
<dd>Authentication will be for the Google account mounting the filesystem, not the UID of the user.  Authentication is through a [service account][service-account] credential.</dd>

<dt>POSIX Support</dt>
<dd>You can use a subset of POSIX functions that align with file and directory support, for example, opendir(), readdir(), rmdir() and closedir(). For a detailed discussion of semantics, see the [Semantics][semantics].</dd>
</dl>

## Billing

gcsfuse is available free of charge, but the storage, metadata, and network IO it generates to and from Google Cloud Storage is charged like any other Cloud Storage interface. You should be aware of the following charges related to using gcsfuse:

* Normal buckets operations (create, delete, and list) incur charges as described in the [Operations][pricing-ops] section of the Cloud Storage pricing page.

* Nearline Storage buckets have costs associated with retrieval and early deletion. See [Nearline Storage][pricing-nearline] on the Cloud Storage pricing page.

* Network egress charges and data transfer between regions and continents incur costs. See the [Network][pricing-network] section on the Cloud Storage pricing page.

Also, if you are moving data into or out of Google cloud, data transfer across regions and continents, depending on the scenario, might incur transfer costs. For more information, see the Google Cloud Platform
[pricing][pricing-cloud] page.

## Installing

gcsfuse works on  Linux and OS X operating systems. For detailed help on satisfying these prerequisites see [Prerequisites][install-prereq].

Prerequisites:

*   gcsfuse is distributed as source code in the [Go][go] language. If you do
    not yet have Go installed, see [here][go-install] for instructions. Be sure
    to follow the linked [setup instructions][go-setup], in particular setting
    the `GOPATH` environment variable and ensuring `$GOPATH/bin` is in your
    `PATH` environment variable.

*   OS X only: Before using gcsfuse, you must have [FUSE for OS X][osxfuse].
    Installing FUSE is not necessary on Linux, since modern versions have kernel
    support built in.

*   The `go get` command below will need to fetch source code from GitHub,
    which requires [Git][git]. If the `git` binary is not installed on your
    system, download it [here][git-download] or install it by some other means
    (for example on Google Compute Engine Debian instances you can run
    `sudo apt-get update && sudo apt-get install git-core`).

*   A service account credential to allow gcsfuse to authenticate. For more information, see
    [service accounts][service-account].
        
After you have satisfied the prerequisites, you can install gcsfuse by running:

    $ go get github.com/googlecloudplatform/gcsfuse

This will fetch the gcsfuse sources, build them, and install a binary named gcsfuse to `$GOPATH/bin`.

## Using gcsfuse

### Manually mounting

Use gcsfuse interactively for testing or debugging. In general, you will most likely auto-mount buckets as shown in the next section.

1. Create a directory.

        $ mkdir gcs

2. Mount a bucket.

        $ gcsfuse --key_file /path/to/client_secrets.json --bucket example-bucket gcs

3. Start working with the mounted bucket.

        $ ls gcs

4. Stop the gcsfuse process when finished with CTRL+C.


### Automatically mounting

Automatically mounting buckets means configuring the `/etc/fstab` so that buckets are automatically mounted after instance reboots or failures.

1. Create service account credential as is shown above and make sure the key is in a location that TBD.

2. [TBD] Edit the `/etc/fstab` file and put the following:

        # <file system>   <dir>  <type>    <options>   <dump> <pass>
        TBD

3. Test `/etc/fstab` file:

        $ mount -a

## Best Practices

Here are some best practices when working with gcsfuse.

<dl>
<dt>Persisting mount through reboots</dt>
<dd>
  If your instance fails or if you reboot it and you manually mounted your
  bucket, you need to remount the bucket. You can remount buckets
  automatically by modifying the `/etc/fstab` file (see
  [Automatically mounting](#user-content-automatically-mounting)).
</dd>
<dt>NFS mounts<dt>
<dd>
  If you use the NFS mount as your working directory, you will receive stale
  file handles when you open more than 10,000 files, or when your instance is
  routed to a new NFS server endpoint because of NFS termination, which can
  happen periodically. To avoid stale file handles, use a working directory
  outside of the NFS mount, and use absolute paths to work with files in the
  mount.
</dd>
<dt>Synchronizing writes</dt>
<dd>
  A file opened for write is not synchronized with Google Cloud Storage until it is
  closed. This means intermediate writes to the file will not be readable by
  other gcsfuse users or Google Cloud Storage users. This also means that
  applications should be careful to inspect the return code from the POSIX
  close() function, as applications often ignore it.
</dd>
<dt>Renaming directories</dt>
<dd>
  You may not rename directories. Instead, you must delete the old directory
  and create a new one. For example, you can use `cp -lR old_dir new_dir`,
  followed by `rm -Rf old_dir`.
</dd>
<dt>Object rename atomicity</dt>
<dd>
  Unlike POSIX, the gcsfuse rename() implementation is not
  atomic. It is implemented using a copy-in-the-cloud followed by a delete.
  Therefore, note that both objects will exist for a short period of time and
  may both continue to exist in the event of failure.
</dd>
</dl>

## FAQ

**Where can I go to discuss gcsfuse ?**

In the [gcsfuse-users][] group.

**Can I continue to work with my buckets using other tools?**

Yes. You can work with any other tools you use with Google Cloud Storage, including [gsutil][gsutil-docs] and [Google Developers Console][console].

**Can I multiple buckets?**

Use {{feat_name}} multiple times to mount multiple buckets.

**Can I use gcsfuse on Google Compute Engine instances?**

Yes. gcsfuse will be supported on all Google Compute Engine [linux operating systems][compute-linux]. If it isn't currently installed, you can follow the [instructions](#user-content-installing) above to install it.

**How can I update my version of gcsfuse?**

Run:

    $ go get -u github.com/googlecloudplatform/gcsfuse

**How can I troubleshoot gcsfuse?**

If you run `gcsfuse` with no arguments, usage information is displayed, including options for showing debug output. One option you can use to help troublshoot is `fuse.debug`.

    $ gcsfuse --fuse.debug --key_file /path/to/client_secrets.json --bucket example-bucket gcs

**Can I set object permissions or ACLs with gcsfuse?**

Setting an object's permissions (e.g., using chmod) is not supported. Viewing and editing Google Cloud Storage Access Control Lists are also not supported.

**Can I create a symbolic link to a file?**

TBD.

**Can I create a hard link to file?**

No.


[gsutil-docs]: https://cloud.google.com/storage/docs/gsutil
[persistent-docs]: https://cloud.google.com/compute/docs/disks/persistent-disks
[ssd-docs]: https://cloud.google.com/compute/docs/disks/local-ssd
[service-account]: https://cloud.google.com/storage/docs/authentication#service_accounts
[semantics]: https://github.com/GoogleCloudPlatform/gcsfuse/blob/master/docs/semantics.md
[pricing-ops]: https://cloud.google.com/storage/pricing#operations-pricing
[pricing-nearline]: https://cloud.google.com/storage/pricing#nearline-pricing
[pricing-network]: https://cloud.google.com/storage/pricing#network-pricing
[pricing-cloud]: https://cloud.google.com/pricing
[go]: http://golang.org/
[go-install]: http://golang.org/doc/install
[go-setup]: http://golang.org/doc/code.html
[osxfuse]: https://osxfuse.github.io/
[git]: http://git-scm.com/
[git-download]: http://git-scm.com/downloads
[gcsfuse-users]: https://groups.google.com/group/gcsfuse-users
[console]: https://console.developers.google.com
[compute-linux]: https://cloud.google.com/compute/docs/operating-systems/linux-os
