---
title: Adding an Uncacheable Attribute to NFSv4.2
abbrev: Uncacheable Attribute
docname: draft-haynes-nfsv4-uncacheable-latest
category: std
date: {DATE}
consensus: true
ipr: trust200902
area: General
workgroup: Network File System Version 4
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping, comments]

author:
 -
    ins: T. Haynes
    name: Thomas Haynes
    organization: Hammerspace
    email: loghyr@gmail.com

normative:
  RFC2119:
  RFC4506:
  RFC4949:
  RFC7204:
  RFC7862:
  RFC7863:
  RFC8174:
  RFC8178:
  RFC8881:
  I-D.ietf-nfsv4-delstid:

informative:
  open:
    title: open and create files.
    seriesinfo: Linux Programmer's Manual
  POSIX.1:
    title: The Open Group Base Specifications Issue 7
    seriesinfo: IEEE Std 1003.1, 2013 Edition
    author:
      org: IEEE
    date:  2013
  RFC1813:
  Samba:
    target: https://www.samba.org/
    title: Samba.org. Samba Project Website. 
  SMB2:
    title:  Server Message Block (SMB) Protocol Versions 2 and 3
    author:
      org: Microsoft Learn

--- abstract

The Network File System version 4.2 (NFSv4.2) allows a client to
cache both metadata and data for file objects, as well as metadata
for directory objects.  While caching directory entries (dirents) can
improve performance, it can also prevent the server from enforcing access
control on individual dirents.  Similarly, caching file data can lead to
performance issues if the cache hit rate is low.  This document introduces
a new uncacheable attribute for NFSv4.  Files and dirents marked as
uncacheable MUST NOT be stored in client-side caches.
This ensures data consistency and integrity by requiring clients to
always retrieve the most recent data directly from the server. This
document extends NFSv4.2 (see RFC7862).

--- note_Note_to_Readers

Discussion of this draft takes place
on the NFSv4 working group mailing list (nfsv4@ietf.org),
which is archived at
[](https://mailarchive.ietf.org/arch/search/?email_list=nfsv4). Source
code and issues list for this draft can be found at
[](https://github.com/ietf-wg-nfsv4/uncacheable).

Working Group information can be found at [](https://github.com/ietf-wg-nfsv4).

--- middle

# Introduction

With a remote filesystem, the client typically caches both directory
entries (dirents) and file contents in order to improve performance.
Several assumptions are made about the rate of change in the dirents
and the number of clients trying to concurrently access a file.
With NFSv4.2, this could theoretically be mitigated by directory
delegations for the dirents and practically by file delegations
for the file contents.

There are prior efforts to bypass both the dirent and file caching.
Access Based Enumeration (ABE) is used in Server Message Block (SMB)
{{SMB2}} protocol to effectively limit the namespace visibility per
user. In Highly Parallel Computing (HPC) workloads, file caching
is bypassed in order to achieve consistent work flows.

This document introduces the uncacheable attribute to NFSv4.2 to
implement ABE and to bypass file caching on the client. As such,
it is an OPTIONAL to implement attribute for NFSv4.2. However, if
both the client and the server support this attribute, then the
client MUST follow the semantics of uncacheable.[^2]

[^2]: What about mixed modes?

A client can easily determine whether or not a server supports
the uncacheable attribute with a simple GETATTR on any
dirent. If the server does not support the uncacheable
attribute, it will return an error of NFS4ERR_ATTRNOTSUPP.

The only way that the server can determine that the client supports
the attribute is if the client sends either a GETATTR or a SETATTR
with the uncacheable attribute.

While some argument could be made to introduce two new attributes,
the functionality of the uncacheable attribute dictates which cache
is to be bypassed. As ABE is concerned with walking the namespace,
it is only applicable to be acted on dirents which are of type attribute value of 
NF4DIR. And as bypassing file caching is file based, it is only
applicable for dirents which are of type attribute value of  NF4REG.[^1]

[^1]: What about the other file types?

Using the process detailed in {{RFC8178}}, the revisions in this document
become an extension of NFSv4.2 {{RFC7862}}. They are built on top of the
external data representation (XDR) {{RFC4506}} generated from
{{RFC7863}}.

## Definitions

Access Based Enumeration (ABE)

: When servicing a READDIR or GETATTR operation, the server provides
results based on the access permissions of the user making the request.

dirent

: A directory entry, representing either a file or a subdirectory. In
the context of NFSv4, a dirent marked as uncacheable MUST NOT be cached
by clients.

dirent caching

: A client cache that is used to avoid looking up attributes.

file caching

: A client cache, normally called the page cache, which caches the
contents of a regular file. Typical usage would be to accumulate
changes to be bunched together for writing to the server.

Further, the definitions of the following terms are referenced as follows:

- directory delegations ({{Section 10.9 of RFC8881}})
- file delegations ({{Section 10.2 of RFC8881}})
- GETATTR ({{Section 18.7 of RFC8881}})
- hidden ({{Section 5.8.2.15 of RFC8881}})
- Mandatory Access Control (MAC) ({{RFC4949}})
- NF4DIR ({{Section 5.8.1.2 of RFC8881}})
- NF4REG ({{Section 5.8.1.2 of RFC8881}})
- NFS4ERR_ATTRNOTSUPP ({{Section 15.1.15.1 of RFC8881}}
- mode ({{Section 6.2.4 of RFC8881}})
- offline ({{Section 2 of I-D.ietf-nfsv4-delstid}})
- owner ({{Section 5.8.2.26 of RFC8881}})
- owner_group ({{Section 5.8.2.27 of RFC8881}})
- READDIR ({{Section 18.23 of RFC8881}})
- SETATTR ({{Section 18.30 of RFC8881}})
- system ({{Section 5.8.2.36 of RFC8881}})
- type ({{Section 5.8.1.2 of RFC8881}})

## Requirements Language

{::boilerplate bcp14-tagged}

# Caching of Dirents

With a remote filesystem, the client typically caches directory
entries (dirents) locally to improve performance. This cooperation
succeeds because both the server and client operate under POSIX
semantics ([POSIX.1]) and agree to interpretation of mode bits with
respect to the uid and gid in NFSv3 {{RFC1813}}. For NFSv4.2, these
would respectively be the mode, owner, and owner_group attributes
defined in {{Section 5 of RFC8881}}.  Note that this cooperation
does not apply to Access Control List (ACLs) entries as NFSv4.2
does not implement a strict POSIX style ACL.

NFSv4.2 does implement NFSv4.1 ACLs, which are enforced on the
server and not the client. As such, ACL enforcement requires the
client to bypass the dirent cache to have checks done when a new
user attempts to access the dirent.

Another consideration is that not all server implementations natively
support the SMB {{SMB2}}. Instead, they layer Samba {{Samba}} on
top of the NFSv4.2 service. The attributes of hidden, system, and
offline have already been introduced in the NFSv4.2 protocol to
support Samba.  The Samba implementation can utilize these attributes
to provide SMB semantics. While private protocols can supply these
features, it is better to drive them into open standards.

Another concept that can be adapted from SMB is that of Access Based
Enumeration (ABE). If a share or a folder has ABE enabled, then the
user can only see the files and sub-folders for which they have
permissions.

Under the POSIX model, this can be done on the client and not
the server. However, that only works with uid, gid, and mode bits.
If we consider identity mappings, ACLS, and server local policies,
then the determination of ABE MUST be done on the server.

Since cached dirents are shared by all users on a client, and the
client cannot determine access permissions for individual dirents,
all users are presented with the same set of attributes. To address
this, this document introduces the new uncacheable attribute. This
attribute instructs the client not to cache the dirent for a file
or directory object. Consequently, each time a client queries for
these attributes, the server's response can be tailored to the
specific user making the request.

## Uncacheable Dirents {#sec_dirents}

If a file object or directory has the uncacheable attribute set,
then the client MUST NOT cache its dirent attributes. This means
that even if the client has previously retrieved the attributes for
a user, it MUST query the server again for those attributes on
subsequent requests. Additionally, the client MUST NOT share
attributes between different users.

# Caching of File Data

In addition to caching metadata, clients can also cache file data.
The uncacheable attribute also instructs the client to bypass its
page cache for the file. This behavior is similar to using the
O_DIRECT flag with the open call ({{open}}). This can be beneficial
for files that are not shared or for files that do not exhibit
access patterns suitable for caching.

However, the real need for bypassing write caching is evident in
HPC workloads. In general, these involve massive data transfers and
require extremely low latency.  Write caching can introduce
unpredictable latency, as data is buffered and flushed later.

## Uncacheable Files {#sec_files}

If a file object is marked as uncacheable, all modifications to
the file MUST be immediately sent from the client to the server. In
other words, the file data is also not cacheable.

# XDR for Offline Attribute

~~~ xdr
///
/// typedef bool            fattr4_uncacheable;
///
/// const FATTR4_UNCACHEABLE            = 87;
///
~~~

# Extraction of XDR

This document contains the external data representation (XDR)
{{RFC4506}} description of the uncacheable attribute.  The XDR
description is presented in a manner that facilitates easy extraction
into a ready-to-compile format. To extract the machine-readable XDR
description, use the following shell script:

~~~ shell
#!/bin/sh
grep '^ *///' $* | sed 's?^ */// ??' | sed 's?^ *///$??'
~~~

For example, if the script is named 'extract.sh' and this document is
named 'spec.txt', execute the following command:

~~~ shell
sh extract.sh < spec.txt > uncacheable_prot.x
~~~

This script removes leading blank spaces and the sentinel sequence '///'
from each line. XDR descriptions with the sentinel sequence are embedded
throughout the document.

Note that the XDR code contained in this document depends on types from
the NFSv4.2 nfs4_prot.x file (generated from {{RFC7863}}).  This includes
both nfs types that end with a 4, such as offset4, length4, etc., as
well as more generic types such as uint32_t and uint64_t.

While the XDR can be appended to that from {{RFC7863}}, the code snippets
should be placed in their appropriate sections within the existing XDR.

# Security Considerations

For a given user A, a client MUST NOT make access decisions for
uncacheable dirents retrieved for another user B. These decisions
MUST be made by the server.  If the client is Labeled NFS aware
({{RFC7204}}), then the client MUST locally enforce the MAC security policies.

The uncacheable attribute allows dirents to be annotated such that
attributes are presented to the user based on the server's access
control decisions.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

Trond Myklebust and Thomas Haynes all worked on the prototype at Hammerspace.
