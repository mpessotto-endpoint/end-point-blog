---
author: "Marco Pessotto"
date: 2025-07-24
title: "A rusty web? An excursion"
tags:
 - web
 - rust
---

In my programmer's career, centered around the web applications, I've
always used dynamic, interpreted languages. Perl *in primis*, but also
Javascript, Python and Ruby. However, I've been curious about
compiled, strongly typed languages and if they can be useful to me and
my clients. [Rust](https://www.rust-lang.org/) would be the first
choice. It's a modern language, has an [excellent
documentation](https://doc.rust-lang.org/stable/book/) and it was also
accepted in the Linux kernel (which sounds like a blessing). However,
it's something *very* different from the languages I know.

I read most of the book a couple of years ago, but given that I didn't
do anything with it, my knowledge evaporated very quickly. This time I
read the book and immediately after that I started to work on a
non-trivial project, involving downloading serialized data in XML from
different sources, database operations, indexing and searching
documents, serving JSON from a web application. My goal was to replace
at least part of a Django application which *seemed* to have
performance problems. The Django application uses Xapian (which is
written in C++) via its
[bindings](https://xapian.org/docs/bindings/python3/) to provide the
core functionality. Reindexing documents would be delegated to a
[Celery](https://docs.celeryq.dev/en/stable/index.html) task queue.

Unfortunately Xapian so far does
[not](https://xapian.org/docs/bindings/) have bindings for Rust. So my
reasoning was: I could use the [PostgreSQL full text search
feature](https://www.postgresql.org/docs/current/textsearch.html)
instead of Xapian, simplifying the setup (updating a row would trigger
an index update, instead of delegating the operation to Celery) and
refining my Postgres knowledge. That was the plan.

Reading the Rust book I truly liked the language, which itself feels
amazing. Its main feature is that it normally gives you no room for
nasty memory management bugs which plague languages like C. Being
compiled to machine code, it's faster than interpreted languages by an
order of magnitude.

If the memory safety comes for free and your program runs fast,
writing the application itself is another story. Coming from dynamic
languages, having to state the type of variables, arguments and return
values is a bit of a cultural shock, but you get used to it quickly.

When writing Perl, I'm very used to construct like these:

```perl
if (my $res = download_url($url)) {
    ...
}
```

which are not possible any more. Instead you have to use the `match`
[construct](https://doc.rust-lang.org/stable/book/ch06-02-match.html)
and extract values from `Option` and `Result` enumerations. This is
the standard way to handle errors and variables which may or may not
have values. There is nothing like an `undef` and this is one of the
main Rust features. So you have to do something like this:

```rust
match download_url(url.clone()) {
    Ok(res) => {
       ...
    },
    Err(e) => println!("Error {url}: {e}"),
}
```

Which could become 

```rust
if let Ok(res) = download_url(url.clone()) {
    ...
}
```

You need be careful to be consistent with the values you are declaring
and returning, and take care of the mutability and the borrowing of
the values. In Rust you can't have a piece of memory which can be
modified in multiple places. If you pass the value to a function,
*normally* you can't use it any more. This is without a doubt a *big
and good thing*. When in Perl for example you pass a reference of hash
to a function, you don't know what happens to it. Things can be
modified without noticing, and you are going to realize later at
debugging time why that piece of data is not what you expect.

In the Rust land, everything feels under strict control, and the
compiler throws errors at you which most of the times are making
sense. It explains to you why you can't use that variable at that
point, and even suggests a fix. It's amazing the amount of work behind
the language and its ability to analyze the code.

The [string
management](https://doc.rust-lang.org/stable/book/ch08-02-strings.html)
feels a bit weird because it's normally anchored to the UTF-8
encoding, while e.g. Perl has an [abstract
way]((https://www.endpointdev.com/blog/2025/04/encoding-in-perl/) to
handle it, so I'm used to think differently about it.

The `async` feature is nice, but present in my most of the modern
languages (Perl included!), so I don't think that should be considered
the main reason to use Rust.

Bottom line I like the language. It's *very* different to what I was
used, but I can see all its advantages. The downside is that you can't
write all those "quick and dirty" scripts which are the daily bread of
the sysadmin.

So, being acquainted with the languages, I went shopping for crates
(which is how the modules are called in Rust) here:
[https://www.arewewebyet.org/](https://www.arewewebyet.org/).

Lately I have a bit of a dislike for object–relational mappings (ORM),
so I didn't go with [diesel](https://diesel.rs/) nor
[sqlx](https://docs.rs/sqlx/latest/sqlx/), but I went straight for
[tokio_postgres](https://docs.rs/tokio-postgres/latest/tokio_postgres/).

This saved me quite a bit of documentation reading and gave me direct
access to the database. Nothing weird to report here. It feels like
using any other DB driver in any other language, with a statement, the
placeholder and the arguments. The difference, of course, is that you
need to care about the data types which are coming out of the DB
(again the `Option` Enum is your friend and the error messages are
helpful).

To get data from the Internet,
[reqwest](https://crates.io/crates/reqwest) did the trick just fine
without any surprise. It works as expected like other user agents
around in other languages.

For XML deserialization, [serde](https://serde.rs/) was paired with
[quick-xml](https://docs.rs/quick-xml/latest/quick_xml/de/). This is
one of the interesting bits.

You start defining your data structures like this:

```rust
#[derive(Debug, Deserialize)]
struct OaiPmhResponse {
    #[serde(rename = "responseDate")]
    response_date: String,
    request: String,
    error: Option<ResponseError>,
    #[serde(rename = "ListRecords")]
    list_records: Option<ListRecords>,
}
```

Then you feed the XML string to the `from_str` function like this:

```
fn parse_response (xml: &str) -> OaiPmhResponse {
    match from_str(xml) {
        Ok(res) => res,
        Err(e) => OaiPmhResponse {
            response_date: String::from("NOW"),
            request: String::from("Invalid"),
            error: Some(ResponseError {
                code: String::from("Invalid XML"),
                message: e.to_string(),
            }),
            list_records: None,
        },
    }
}
```

which takes care of the parsing and gives you back either an `Ok` with
inside your data structure, or an error, and you are supposed to
account for both cases. The structs can have methods so this provides
a nice encapsulation. And super fast, too.

[Axum](https://github.com/tokio-rs/axum)







