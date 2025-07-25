---
author: "Marco Pessotto"
date: 2025-07-24
title: "A rusty web? An excursion of a Perl guy into the Rust land"
tags:
 - web
 - rust
---

In my programmer's career, centered around the web applications, I've
always used dynamic, interpreted languages. Perl *in primis*, but also
Javascript, Python and Ruby. However, I've always been curious about
compiled, strongly typed languages and if they can be useful to me and
to my clients. [Rust](https://www.rust-lang.org/) would be the first
choice. It's a modern language, has an [excellent
documentation](https://doc.rust-lang.org/stable/book/) and it's quite
popular. However, it's something *very* different from the languages I
know.

I read most of *the book* a couple of years ago, but given that I
didn't do anything with it, my knowledge evaporated very quickly. This
time I read the book and immediately after that I started to work on a
non-trivial project, involving downloading XML data from different
sources, database operations, indexing and searching documents, and
finally serving JSON over HTTP. My goal was to replace at least part
of a Django application which *seemed* to have performance problems.
The Django application uses Xapian (which is written in C++) via its
[bindings](https://xapian.org/docs/bindings/python3/) to provide the
core functionality. Reindexing documents would be delegated to a
[Celery](https://docs.celeryq.dev/en/stable/index.html) task queue.

Unfortunately Xapian does [not](https://xapian.org/docs/bindings/)
have bindings for Rust so far. 

My reasoning was: I could use the [PostgreSQL full text search
feature](https://www.postgresql.org/docs/current/textsearch.html)
instead of Xapian, simplifying the setup (updating a row would trigger
an index update, instead of delegating the operation to Celery) and
down the road I could also deepen my PostgreSQL knowledge. That was the
plan.

Reading the Rust book I truly liked the language. Its main feature is
that it (normally) gives you no room for nasty memory management bugs
which plague languages like C. Being compiled to machine code, it's
faster than interpreted languages by an order of magnitude. However,
having to state the type of variables, arguments and return values, at
the beginning it was a bit of a cultural shock, but I got used to it
quickly.

When writing Perl, I'm used to construct like these:

```perl
if (my $res = download_url($url)) {
    ...
}
```

which are not possible any more. Instead you have to use the `match`
[construct](https://doc.rust-lang.org/stable/book/ch06-02-match.html)
and extract values from `Option` (`Some`/`None`) and `Result` (`Ok`,
`Err`) enumerations. This is the standard way to handle errors and
variables which may or may not have values. There is nothing like an
`undef` and this is one of the main Rust features. Instead, you need
to cover all the cases with something like this:

```rust
match download_url(url.clone()) {
    Ok(res) => {
       ...
    },
    Err(e) => println!("Error {url}: {e}"),
}
```

Which can also be written as:

```rust
if let Ok(res) = download_url(url.clone()) {
    ...
}
```

You must be consistent with the values you are declaring and
returning, and take care of the mutability and the borrowing of the
values. In Rust you can't have a piece of memory which can be modified
in multiple places. If you pass the value to a function, *normally*
you can't use it any more. This is without a doubt a *good thing*. For
example, when in Perl you pass a reference of hash to a function, you
don't know what happens to it. Things can be modified as a side
effect, and you are going to realize later at debugging time why that
piece of data is not what you expect.

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

Bottom line: I like the language. It's *very* different to what I was
used to, but I can see all its advantages. The downside is that you
can't write all those “quick and dirty” scripts which are the daily
bread of the sysadmin.

Once I got acquainted with the languages, I went shopping for crates
(which is how the modules are called in Rust) here:
[https://www.arewewebyet.org/](https://www.arewewebyet.org/).

Lately I have a bit of a dislike for object–relational mappings (ORM),
so I didn't go with [diesel](https://diesel.rs/) nor
[sqlx](https://docs.rs/sqlx/latest/sqlx/), but I went straight for
[tokio_postgres](https://docs.rs/tokio-postgres/latest/tokio_postgres/).

This saved me quite a bit of documentation reading and gave me direct
access to the database. Nothing weird to report here. It feels like
using any other DB driver in any other language, with a statement, the
placeholders and the arguments. The difference, of course, is that you
need to care about the data types which are coming out of the DB
(again the `Option` Enum is your friend and the error messages are
helpful).

To get data from the Internet,
[reqwest](https://crates.io/crates/reqwest) did the trick just fine
without any surprise.

For XML deserialization, [serde](https://serde.rs/) was paired with
[quick-xml](https://docs.rs/quick-xml/latest/quick_xml/de/). This is
one of the interesting bits.

You start defining your data structures like this:

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct OaiPmhResponse {
    #[serde(rename = "responseDate")]
    response_date: String,
    request: String,
    error: Option<ResponseError>,
    #[serde(rename = "ListRecords")]
    list_records: Option<ListRecords>,
}
// more definitions follow, to match the structure we expect
```

Then you feed the XML string to the `from_str` function like this:

```rust
use quick_xml::de::from_str;

fn parse_response (xml: &str) -> OaiPmhResponse {
    match from_str(xml) {
        Ok(res) => res,
        // return a dummy one with no records in it in case of errors
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
inside the data structure you defined, with the tags properly mapped,
or an error. The structs can have methods so this provides a nice
OOP-like encapsulation.

Once the data collection was successful, I moved to the web
application itself.

I chose the [Axum](https://github.com/tokio-rs/axum) framework,
maintained by the [Tokio project](https://tokio.rs/) and glued all the
pieces together.

With the core being something like this:

```rust
#[derive(Serialize, Debug)]
struct Entry {
    entry_id: i32,
    rank: f32,
    title: String,
}

async fn search(
    State(pool): State<ConnectionPool>,
    Query(params): Query<HashMap<String, String>>,
) -> (StatusCode, Json<Vec::<Entry>>) {
    let conn = pool.get().await.expect("Failed to get a connection from the pool");
    let sql = r#"
SELECT entry_id, title, ts_rank_cd(search_vector, query) AS rank
FROM entry, websearch_to_tsquery($1) query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 10;
"#;
    let query = match params.get("query") {
        Some(value) => value,
        None => "",
    };
    let out = conn.query(sql, &[&query]).await.expect("Query should be valid")
        .iter().map(|row|
                    Entry {
                        entry_id: row.get(0),
                        title: row.get(1),
                        rank: row.get(2),
                    }).collect();
    tracing::debug!("{:?}", &out);
    (StatusCode::OK, Json(out))
}
```

Which simply runs the query using the input provided by the user, runs
the full text search, and returns the serialized data as JSON.

During development it *felt* fast. The disappointment came when I
populated the database with about 30,000 documents of various size.
The Django application, despite returning more data and facets, was
still way faster. With the two applications running on the same
machine I got for 925 ms the Rust application, and 123 ms for the
Django one!

Now, obviously the problem here is not Python vs. Rust, but Xapian vs.
PostgreSQL. Most of the time is consumed in the SQL query, so here the
benchmark is actually between PostgreSQL and Xapian, with Xapian
winning by a large measure. Even if the Axum application is as fast as
it can get, because it's stripped to the bare minimum (it has no
sessions, no authorization, no templates), the time saved is not
enough to compensate the lack of a dedicated and optimized full text
search engine like Xapian (where Python is just providing an interface
to fast C++ code). Of course I shouldn't be surprised.

Beside the failure of the initial plan, this was really a nice and
constructive excursion, as I could learn a new language, using its
libraries to do common tasks like downloading data, making web
applications, interfacing with the database. Rust appears to have
plenty of quality crates.

To actually compete with Django + Xapian, I should probably use
[Tantivy](https://github.com/quickwit-oss/tantivy), instead of relying
on the PostgreSQL full text search. But that would be another
adventure...
