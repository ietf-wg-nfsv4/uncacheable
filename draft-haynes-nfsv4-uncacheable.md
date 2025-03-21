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
pi: [toc, sortrefs, symrefs]

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

informative:

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

In the Network File System version 4.2 (NFSv4.2) {{RFC7863}}, a client
queries for either a file's or directory's attributes via either GETATTR
({{Section 18.7 of RFC8881}}) or READDIR ({{Section 18.23 of RFC8881}})
to the server. These directory entries (dirents) can be cached locally
by the client.

Since cached dirents are shared by all users on a client, and the
client cannot determine access permissions for individual dirents, all
users are presented with the same set of attributes. To address this,
this document introduces the new uncacheable attribute. This attribute
instructs the client not to cache the dirent for a file or directory
object. Consequently, each time a client queries for these attributes,
the server's response can be tailored to the specific user making
the request, based on factors such as Access Control Lists (ACLs) on
the file or directory object ({{Section 6 of RFC8881}})
or proprietary policies.

In addition to caching metadata, clients can also cache file data. The
uncacheable attribute also instructs the client to bypass its page cache
for the file. This behavior is similar to using the O_DIRECT flag with
the open call. This can be beneficial for files that are not shared or
for files that do not exhibit access patterns suitable for caching.

Using the process detailed in {{RFC8178}}, the revisions in this document
become an extension of NFSv4.2 {{RFC7862}}. They are built on top of the
external data representation (XDR) {{RFC4506}} generated from {{RFC7863}}.

## Definitions

Access Based Enumeration

: When servicing a READDIR or GETATTR operation, the server provides
results based on the access permissions of the user making the request.

dirent

: A directory entry, representing either a file or a subdirectory. In
the context of NFSv4, a dirent marked as uncacheable MUST NOT be cached
by clients.

## Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Uncacheable Dirents {#sec_dirents}

If a file object or directory has the uncacheable attribute set,
then the client MUST NOT cache its dirent attributes. This means
that even if the client has previously retrieved the attributes
for a user, it MUST query the server again for those attributes
on subsequent requests. Additionally, the client MUST NOT share
attributes between different users.

# Uncacheable Files {#sec_files}

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
({{RFC7204}}), then the client will locally enforce the Mandatory
Access Control (MAC) (as defined in {{RFC4949}}).

The uncacheable attribute allows dirents to be annotated such that
attributes are presented to the user based on the server's access
control decisions.

# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

Trond Myklebust and Thomas Haynes all worked on the prototype at Hammerspace.
