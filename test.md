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

[gsutil-docs]: https://cloud.google.com/storage/docs/gsutil
[persistent-docs]: https://cloud.google.com/compute/docs/disks/persistent-disks
[ssd-docs]: https://cloud.google.com/compute/docs/disks/local-ssd
[service-account]: https://cloud.google.com/storage/docs/authentication#service_accounts
[semantics]: https://github.com/GoogleCloudPlatform/gcsfuse/blob/master/docs/semantics.md
