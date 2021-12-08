Rustdoc: jump to definition

- Feature Name: `jump_to_definition`
- Start Date: 2021-12-09
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#89095](https://github.com/rust-lang/rust/issues/89095)

# Summary
[summary]: #summary

Generate links on idents in the rustdoc source code pages which links to the item's definition and documentation (if available).

# Motivation
[motivation]: #motivation

This RFC proposes to generate links in the rustdoc source code pages (the ones you end up on when clicking on the `source` links) to allow you to jump to an item definition or back to its documentation page.

The goal of this RFC is to greatly improve the browing experience of the source page (especially when looking for private items).

Since a video is worth a thousand words:

https://user-images.githubusercontent.com/3050060/114622354-21307b00-9cae-11eb-834d-f6d8178a37bd.mp4

You can also give it a try with the following docs:

 * [askama](https://rustdoc.crud.net/imperio/jump-to-def-askama/src/askama/lib.rs.html)
 * [regex](https://rustdoc.crud.net/imperio/jump-to-def-regex/src/regex/lib.rs.html)
 * [rand](https://rustdoc.crud.net/imperio/jump-to-def-rand/src/rand/lib.rs.html)

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

We take as given that "source view" is a valuable part of rustdoc. Sometimes, it's necessary to go beyond the documentation and look at the implementation. That's why source view is a common feature in documentation tooling for a variety of languages (like Go for example, you can access this [source code](https://cs.opensource.google/go/go/+/go1.16.5:src/database/sql/sql.go;l=44) from [here](https://pkg.go.dev/database/sql#Register), or python docs or [hexdocs.pm](https://preview.hex.pm/preview/k6/0.0.1/show/lib/k6/template/web_socket.ex) for Elixir and Erlang languages). This is not a rare feature.

Rustdoc's source view often runs into a particular problem: to properly understand the implementation, you need to make reference to identifiers defined elsewhere. For instance, consider the common newtype pattern:

```
pub MyStruct(OtherStruct)
```

If you clicked through on `MyStruct`'s src link, expecting to see the private implementation details, you'd be disappointed: you actually need to see the contents of `OtherStruct`. However, `OtherStruct` might be in another file entirely. Tracking it down can be a tedious process:

 - Use Ctrl-F to search the page for either a definition of `OtherStruct` or a `use` statement that imports it.
 - If you found a `use` statement, figure out which file it maps to.
 - Open the sidebar and navigate to that file.
 - Ctrl-F again to find the actual definition of `OtherStruct`.
 
If `OtherStruct` itself is defined in terms of additional structs, you may need to repeat this process many times to get even a cursory understanding of the implementation of `MyStruct`. It can also be error prone. If we implement "go to definition", we can turn this whole process into a single click, and make the result reliable.

The same applies when reading a source code when type annotations are not available:

```
let var = some_other_var.do_something();
let var2 = a_function();
```

It would require to look for each method in the various files, etc.

So overall, this would greatly improve the source code browsing experience.

The other goal is to allow to go back from the source code pages directly to the item documentation to be able to read the rendered content and not just the "raw" doc comments.

So in short, this feature is split in two parts:

 * Jump to the definition of an ident.
 * Jump to an item documentation from the source view.

Adding this information directly into the documentation allows it to improve the documentation value: if you want to go a bit further what is written in the documentation (to see some implementation details for example), you can currently go to the source code. However, you're quickly limited to the current file if it's using private items not from this file.

Once you found what you're looking for, having a link to an item's documentation from the source code allows to add the missing connection between the source code viewer and the doc pages.

Another important note is that if documentation is split in multiple places or requires an external tool for the source code navigation, it's actually slowing down our users quite a lot.

A good example where this would be very helpful is for the [rust compiler source code](https://doc.rust-lang.org/1.57.0/nightly-rustc/src/rustc_middle/hir/map/mod.rs.html#872-877): even if a fully set up IDE or github, it's very hard to go around without previous knowledge or help from someone else. It's also very common to have undocumented items or to just want to see how something is implemented. With the "jump to definition", it already improved quite a lot the browsing experience by allowing to jump super quickly by simply clicking on a item. The missing part is now to go back from the source code into the item documentation to make it complete.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

It doesn't require javascript since it's only generating links, making it work with or without javascript enabled.

For the link generation, we use the already existing system in rustdoc. Meaning their generation will follow the same rules as the link in the documentation pages.

All items with documentation should have a link to their documentation page (like impl blocks, methods, trait/impl associated items, etc).

So on the technical side, there is no big issues (based on what is described in this RFC). A few parts of this RFC are implemented in the following pull requests:

 * [#84176](https://github.com/rust-lang/rust/pull/84176)
 * [#88033](https://github.com/rust-lang/rust/pull/88033)
 * [#91264](https://github.com/rust-lang/rust/pull/91264)

## UI

For the UI, we have the following constraints:

 * Must work on mobile and desktop: we cannot rely on mouse hover events.
 * It's better if it works without javascript (it's important for accessibility, but also for maintenance reasons too: less JS code, less maintenance).

Some extra explanations about the statement "less JS code, less maintenance": To have this information in the front-end, the back-end has to generate it in any case (either as a link or any other format). So deciding to use JS to handle this would mean that we still have code in the back-end but we would also have code in the front-end to treat it.

With this in mind, the suggested UI would be like this:

 * Links generated on idents in source code will jump to the ident definition.
 * Links generated on items' definition will jump to their documentation page (if any, private items don't have one by default).
 * On mobile, these links will have dotted underline (because you don't have a cursor which change when hovering them).

Another thing which would be nice: to make it more obvious if a link points to source code or to documentation, they should have different visual markers (different background/underline colors for example).

Basically, the source code pages' UI doesn't change.

# Drawbacks
[drawbacks]: #drawbacks

**The biggest concern is actually about "scope expansion"**. If we add this feature, are we doing too much? Rustdoc is supposed to be about documentation, so is providing more information into the source code page really necessary?

An answer about this is that it actually makes source code browsing much more pleasant and convenient. Multiple other documentation tools provide this feature for this reason so it doesn't seem like it would be out of scope in this regard.

**Another concern is**: "why should we implement this in rustdoc? Aren't there already existing tools doing it?"

A good answer to this is actually the following scenario: you are on <docs.rs> and you're looking at a crate documentation. You want to see how something is implemented and click on the `source` link. At this point, you encounter an item used in this page but not defined in this page. Do you want to clone the crate locally to check it out or go to github/gitlab to check it there or do you prefer having the links directly available?

**Another concern is**: "impact on the size of the generated pages"

Since we generate links, it will increase the size of the source code pages. Here are a few examples with the currently implemented parts of this feature:

| crate name | without the feature (in bytes) | with the feature (in bytes) | diff |
| ---------- | ------------------------------ | --------------------------- | ---- |
| std | 11.788.288 | 13.172.736 | 11.7% |
| tract_core | 4.276.224 | 5.554.176 | 29.9% |
| stm32f4 | 22.335.488 | 26.984.448 | 20.8% |
| image | 4.927.488 | 6.045.696 | 22.7% |

The impact on the number of DOM nodes now (I took random pages with enough code for it to be significant). To compute it, I used `document.getElementsByTagName('main')[0].getElementsByTagName('*').length ` in the browser console:

| file | without the features | with the feature | diff |
|-|-|-|-|
| askama_shared/parser.rs.html | 8414 | 8677 | 3.1% |
| regex/re_bytes.rs.html | 4386 | 4386 | 0% |
| rand/distributions/uniform.rs.html | 6838 | 6838 | 0% |

**Last concern is about the maintenance**.

Any feature has a maintenance cost, this one included. It was suggested to put this feature in its own crate. The only part that can be extracted is the span map builder. But it would very likely be more of a problem than a solution considering that rustdoc API doesn't allow it easily. An important reminder: this feature is less than 200 lines of code in rustdoc and doesn't require any extra dependency.

One more argument about this is that the feature actually requires not that much code. It is split in two parts:

 * https://github.com/rust-lang/rust/blob/master/src/librustdoc/html/render/span_map.rs (which is basically a visitor gathering `span`)
 * https://github.com/rust-lang/rust/blob/master/src/librustdoc/html/highlight.rs#L725-L753 (for generating links)

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

[Github Code navigation feature](https://docs.github.com/en/repositories/working-with-files/using-files/navigating-code-on-github): Relying on external (and private) tools doesn't seem like a good idea. If they decide to shut it down, we can't do anything about it. In addition to that, a lot of projects are not on github, so it would exclude them. It also requires an internet connection. And finally, the biggest setback: it would very likely not support linking to items from other crates.

[Google Code Search](https://cs.opensource.google/fuchsia/fuchsia/+/main:src/factory/factory_store_providers/src/config.rs;l=62): Same issues as listed for `Github Code Navigation`.

rust-analyzer: This is a public opensource project. However, the setup isn't super easy (but it's a minor issue) and when using it on the rust compiler source code, it's quite resource-heavy. Also, if you want to look at a crate source code on <docs.rs>, you'd need to clone it locally to be able to use rust-analyzer on it.

rust-analyzer in the browser: Another possibility would be to run it with wasm. However, it brings a few issues: it requires JS to work (obviously, but it's not a big problem), it would be quite heavy to run and it would complexify rustdoc build quite a lot.

External library (like `cpan` or `tree-sitter`): They would work to generate links to another source code location. However, they bring their downsides as well: they would need to be kept up-to-date with rust syntax evolution (if any) and making links back to documentation would be very tricky because some information is only internal to rustdoc (like how to differentiate between different `impl blocks` or different blanket trait implementations like `impl Trait<T: Clone> for T` and `impl Trait<T: Copy> for T`).

# Prior art
[prior-art]: #prior-art

As mentionned above, it is actually quite common for documentation tools to have a source viewer. Implementations vary a bit between them:

| tool/language | source code viewer | jump to definition | go back to documentation |
| ------------- | ------------------ | ------------------ | ------------------------ |
| [doxygen](https://eigen.tuxfamily.org/dox/Matrix_8h_source.html) | yes | no | yes |
| [Elixir](https://elixir.bootlin.com) (used for linux kernel) | yes | yes | no |
| [Go](https://cs.opensource.google/go/go/+/go1.16.5:src/database/sql/sql.go;l=44) | yes | yes | no |
| [hexdocs](https://preview.hex.pm/preview/k6/0.0.1/show/lib/k6/template/web_socket.ex) | yes | no | no |
| python (sphynx/pydoc) | no (links to github) | no | no |
| [ruby](https://ruby-doc.org/stdlib-3.1.2/libdoc/delegate/rdoc/Delegator.html#method-i-__raise__) | yes (directly on the page) | yes/no (it is sometimes in examples) | no |
| [perl](https://metacpan.org/dist/DBD-SQLite/source/lib/DBD/SQLite.pm) | yes | yes | yes |
| [haskell](https://devdocs.io/haskell~9/libraries/binary-0.8.9.0/data-binary-builder#v:append) | yes | yes | no |
| [opal](https://opalrb.com/docs/api/v1.4.1/stdlib/Buffer) | yes (directly on the page) | no | yes |
| [cargo-src](https://github.com/rust-dev-tools/cargo-src) | yes | yes | yes |

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Is it the best way to make the links to go to an item documentation like this? Maybe something adding a small icon to click right besides it should be better?

# Potential extensions

A potential extension would be "find references". It would allow users to check where the given item is being used. We already have all the information needed for this feature, so it's mostly how to use it and show it to the end users. However, if we decide to move forward with it, it will be part of another RFC.
