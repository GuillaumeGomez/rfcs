Rustdoc: Bundle local images

- Feature Name: NONE
- Start Date: 2023-02-06
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#32104](https://github.com/rust-lang/rust/issues/32104)

# Summary
[summary]: #summary

This RFC proposes to allow the bundling of local images in rustdoc HTML output. A draft implementation is available as [#107640](https://github.com/rust-lang/rust/pull/107640).

# Motivation
[motivation]: #motivation

Currently, rustdoc does not allow for the inclusion of local images in the generated output. This limits users from customizing their rustdoc output by adding images to their documentation as they currently need to host them on a web server so they can be available from anywhere, but therefore requiring an access to the internet. Bundling local images would enable users to customize their rustdoc output and make it more visually appealing.

This would make the documentation more engaging and easier to understand while lowering the amount of effort required to achieve a better result.

# Guide-level explanationvide
[guide-level-explanation]: #guide-level-explanation

This RFC proposes to allow rustdoc to include local images in the generated documentation by copying them into the output directory.

This would be done by allowing users to specify the path of a local resource file in doc comments. The resource file would be stored in the `doc.files` folder. The `doc.files` folder will be at the "top level" of the rustdoc output level (at the same level as the `static.files` or the `src` folders).

The only local resources considered will be the ones in the markdown image syntax: `![resource title](path)`, where `<path>` is the path of the resource file relative to the source file.

The path could be either a relative path (`../images/my_image.png`) or an absolute path (`/images/my_image.png`):

```rust
/// Using a local image ![with absolute path](/local/image.png)
///
/// Using a local image ![with relative path](../local/image.png)
```

The local resources files are not affected by the `--resource-suffix`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A new rustdoc pass will be added which would go through all documentation to gather local resources into a map.

Then in HTML documentation generation, the local resources pathes will be replaced by their equivalent linking to the output directory instead.

The local resources files will be renamed as follows: `{original filename}-{hash}{extension}`. The `{hash}` information will be computed from the local resource file content.

You can look at what the implementation could look like in [#107640](https://github.com/rust-lang/rust/pull/107640).

# Drawbacks
[drawbacks]: #drawbacks

Allowing local resources in rustdoc output could lead to big output files if users include big resource files. This could lead to slower build times and increase the size of generated documentation (in particular in case of very big local resources!).

# Prior art
[prior-art]: #prior-art

- [sphinx](https://www.sphinx-doc.org/en/master/usage/configuration.html#confval-latex_additional_files)
- [haddock](https://haskell-haddock.readthedocs.io/en/latest/invoking.html?highlight=image#cmdoption-theme): it's mentioned in this command documentation that local files in the given directory will be copied into the generated output directory.
- [doxygen](https://doxygen.nl/manual/commands.html#cmdimage): supported through `\image`.

Another approach to this feature:

[ePUB packages](https://www.w3.org/publishing/epub3/epub-packages.html#sec-pkg-manifest) use explicitly-declared manifests. It allows to have a fallback chain mechanism (going through resources for an entry until an available resource is found).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Currently, to provide resources, users need to specify external URLs for resources or inline them (if possible like the `svg` image format) directly into the documentation. It has the advantage to avoid the problem of large output files, but it also requires users to upload their resources to a web server to make them available everywhere.

# Unresolved Questions
[unresolved-questions]: #unresolved-questions

- Should we put a size limit on the local resources?
- Should we somehow keep the original local resource filename instead of just using a number instead?
- Should we use this feature for the logo if it's a local file?

# Possible extensions
[possible-extensions]: #possible-extensions

This feature could be extended to DOM content using local resources. It would require to add parsing for HTML tags attributes. For example:

```html
/// <video src="../some-video.mp4">
```
