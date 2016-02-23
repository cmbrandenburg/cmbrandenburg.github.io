---
date: 2016-02-23
title: Rethinking CouchDB in Rust
categories:
    - software design
tags:
    - chill-rs
    - CouchDB
    - couchdb-rs
    - open source
    - Rust
---

My knowledge of Rust has surpassed my knowledge of CouchDB. I now think
less about _how_ to abstract the CouchDB API and more about _what_ to
abstract. Additionally, I believe my original strategy for the [couchdb
crate][couchdb_github] is a bad idea.

That strategy is to provide a thin abstraction whereby the application
has fine-grained control over each HTTP request it sends to the CouchDB
server. Here's an example request with one query parameter:

```rust
// GET /stuff/my_doc?rev=<some_revision>
client.get_document("/stuff/my_doc")
      .rev(&some_revision)
      .run();
```

There's not much abstraction here. The application explicitly sets the
URI path and `rev` query parameter. This level of abstraction leads to
two problems:

1. The CouchDB API provides many ways of doing the same thing, so the
   library would also provide many ways of doing the same thing—i.e.,
   bloat.

1. The library is less useful to most applications.

Let's look at the first problem: **bloat**.

Here are some examples how CouchDB has two ways of doing the same thing:

* Are you creating a document? You can PUT the document _or_ POST it to
  the database.

* Are you deleting a document? You can specify its revision via the
  `If-Match` header _or_ the `rev` query parameter.

* Are you uploading an attachment? You can embed the attachment content
  as JSON using base64-encoding _or_ use a multipart message to separate
  the attachment from its document and avoid the overhead of base64.

A good CouchDB library will hide meaningless choices and use a
reasonable default. For example, the library should use multipart to
upload attachment content because multipart uses significantly less
bandwidth than base64 in real-world cases. Application programmers
shouldn't be bothered about this detail.

There's even more to be said about attachments, and that brings us to
the second point: **being useful**. But before I explain this, you need
a basic, two-minute understanding of CouchDB attachments.

A CouchDB attachment is a MIME-typed blob added to a document—think
<q>email attachment.</q> But, unlike an email, a CouchDB document is
revision-controlled, hence each attachment has a history. Imagine the
following sequence of events:

1. You create a document with an attachment containing the content
   <q>Hello</q> at document revision 1.

1. You update the attachment with the content <q>Goodbye</q> at document
   revision 2.

After updating the attachment, you can retrieve the original
<q>Hello</q> content by explicitly requesting revision 1.

But what happens to an attachment when you update the document itself?
That depends on how much info you send in your update:

1. If you send the full attachment, including content, then the server
   will overwrite the existing attachment with the new content.

1. Or, if you send an attachment <q>stub</q> containing only the
   attachment's name, then the server will make no changes to the
   existing attachment.

1. Or, if you send no attachment info at all, the server will delete the
   existing attachment.

If the library requires the application to explicitly provide attachment
info—as the couchdb crate does—then deletion is the default. But
deletion is a bad default. A better default would be to send a stub and
make no changes.

However, a CouchDB library that does automatic stub-sending would take
more control over the outgoing HTTP request. Such a library would also
need to know about all attachments without the application telling the
library about them. Is this even possible? Yes.

Imagine the following pseudocode:

```rust
struct Meta {
    id: DocumentId,
    revision: Revision,
    attachments: HashMap<String, Attachment>,
}

struct Speech {
    transcript: String,
}

let doc1 = db.read_document("gettysburg");

let (meta, mut content): (Meta, Speech) = doc1.into_content();

if content.transcript == "Four score and *eight* years ago…" {

    // Oops! Need to correct Lincoln's speech.
    content.transcript = "Four score and *seven* years ago…".to_owned();

    let doc2 = Document::from_content(meta, content);
    db.write_document(doc2); // sends a stub for any existing attachment
}
```

A key fact is that the CouchDB server sends attachment info as part of
any document. Hence, in the code above, the `doc1` variable holds
all attachment info, and it transfers the info to `meta`, with `meta`
later transferring the info to `doc2`. When the application sends `doc2`
to the server, the library knows enough to send a stub for any existing
attachment.

Suppose instead the application adds (or modifies) an attachment:

```rust
let doc3 = db.read_document("washington_farewell");

let (mut meta, content): (Meta, Speech) = doc3.into_content();
meta.attachments.insert("manuscript.png",
                        Attachment::new("image/png", load_image()));
let doc4 = Document::from_content(meta, content);

db.write_document(doc4); // sends a stub for any preexisting attachment
```

In this case, the `doc4` variable has enough info to send the full
content of the new `manuscript.png` attachment and a stub for any other,
preexisting attachment. Waste not, want not.

I've been exploring this and other ideas in my new project,
[Chill][chill_github].

Chill hides more HTTP headers, URI query parameters, and JSON content of
the HTTP messages. With Chill, the application declares _what_ to do and
Chill figures out _how_ to do it.

<hr>

My thanks go to Jeremy Wright for editing early drafts of this article
and making it better.

[couchdb_github]: https://github.com/couchdb-rs/couchdb
[chill_github]: https://github.com/cmbrandenburg/chill-rs
