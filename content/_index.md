+++
template = "default.html"
+++

# Reproducing Zola Issues & Questions

This is a basic site used to replicate issues and questions that arise from using the [Zola](https://getzola.org) static site generator.

## File Searching Logic

Zola's handling of local paths can be frustrating at times due to subtle nuances. Here are the relevant [Zola docs](https://www.getzola.org/documentation/templates/overview/#file-searching-logic).

In particular, the docs say:

> For functions that are searching for a file on disk *other than through get_page and get_section*, the following logic applies.
> 1. The base directory is the Zola root directory, where the config.toml is
> 1. For the given path: if it starts with @/, replace that with content/ instead and trim any leading /
> 1. Search in the following locations in this order, returning the first where the file exists:
>     1. $base_directory + $path
>     1. $base_directory + "static/" + $path
>     1. $base_directory + "content/" + $path
>     1. $base_directory + $output_path + $path
>     1. $base_directory + "themes" + $theme + "static/" + $path (only if using a theme)
>
> It will error if the path is outside the Zola directory.
>
> In practice this means that @/some/image.jpg, /content/some/image.jpg and content/some/image.jpg will point to the same thing.

### Within Markdown

 Ex. # | Case                                                    | Actual Result                                                                                                          | Comments
-------|---------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|-----------
 1     | `[Home](/)`                                             | [/](/) |
 2     | `[Home](@/_index.md)`                                   | [@/_index.md](@/_index.md) |
 3     | `[Home](/index.html)`                                   | [/index.html](/index.html) |
 3     | `[Home](@/index.html)`                                   | [/index.html](@/index.html) |
 4     | `[Home](/does-not-exist/)`                              | [/does-not-exist](/does-not-exist/) |
 5     | `[Home](@/does-not-exist/index.md)`                     | <!--{% raw() %}[@/does-not-exist/index.md](@/does-not-exist/index.md){% end %}--> Error |
 6     | `[page](@/page.md)`                                     | [@/page.md](@/page.md) |
 7     | `[dir-w-date](@/dir-w-date/index.md)`                   | <!--{% raw() %}[dir-w-date](@/dir-w-date/index.md){% end %}--> Error | See Ex. 8
 8     | `[dir-w-date](@/2025-01-29_dir-w-date/index.md")`       | [@/2025-01-29_dir-w-date/index.md](@/2025-01-29_dir-w-date/index.md)| MUST include date for directories, when addressing via an  internal, validated links
 8     | `[dir-w-date](/dir-w-date/)`                            | [dir-w-date](/dir-w-date/) | See Ex. 9
 9     | `[dir-w-date](/2025-01-29_dir-w-date/)`                 | [/2025-01-29_dir-w-date/](/2025-01-29_dir-w-date/) | 404 - Must NOT include date for directories, when addressing via an internal, non-validated link
 10    | `![img](/2025-01-29_dir-w-date/img.png)`                | ![img](/2025-01-29_dir-w-date/img.png) | 404 - MUST NOT include date for directories, when addressing via an internal, non-validated link
 11    | `![img](@/2025-01-29_dir-w-date/img.png)`               | ![img](@/2025-01-29_dir-w-date/img.png) | 404 - internal, non-validated links for images are allowed, but have no effect, see `get_url` Ex
 12    | `![img](/dir-w-date/img.png)`                           | ![img](/dir-w-date/img.png)|
 13    | `[#file-searching-logic](#file-searching-logic)`        | [#file-searching-logic](#file-searching-logic) |
 14    | `[#does-not-exist](#does-not-exist)`                    | <!-- {% raw() %}[#does-not-exist](#does-not-exist){% end %} --> ERROR | Document fragment link validation is enforced within the same document, not the same template

There are several inconsistencies here, but the behavior is consistent with the documentation.

The most important two things to improve for me would be

1. unify local path and url path for internal link validation
2. internally validate

### get_url

As far as I can tell, there are only only a few functions that this applies to: `get_url` ([docs](https://www.getzola.org/documentation/templates/overview/#get-url)), `get_hash` ([docs](https://www.getzola.org/documentation/templates/overview/#get-hash)), `get_image_metadata` ([docs](https://www.getzola.org/documentation/templates/overview/#get-image-metadata)), and `load_data` ([docs](https://www.getzola.org/documentation/templates/overview/#load-data)). The remaining functions that take a `path` argument are `get_page` and `get_section`, which are excluded from the **file searching logic**.

Let's examine each one more closely:

The `get_url` docs specify:

> If the path starts with @/, it will be treated as an internal link to a Markdown file, starting from the root content directory as well as validated.

Meaning that link validation via `get_url` **only happens if the link is to a markdown file**.

 Ex. # | Function                                                | Actual Result                                                                                   | Comments |
-------|---------------------------------------------------------|-------------------------------------------------------------------------------------------------| ---------|
 1     | `get_url(path="/index.html")`                           | {{ get_url(path="/index.md") }}                                                                 |
 2     | `get_url(path="@/_index.md")`                           | {{ get_url(path="@/_index.md") }}                                                               |
 3     | `get_url(path="@/page.md")`                             | {{ get_url(path="@/page.md") }}                                                                 |
 4     | `get_url(path="@/dir-w-date/index.md")`                 | <!-- {% raw() %}get_url(path="@/page-with-date-in-directory/index.md"){% end %}-->  ERROR       |
 5     | `get_url(path="@/2025-01-29_dir-w-date/index.md")`      | {{ get_url(path="@/2025-01-29_dir-w-date/index.md") }}                                          |
 6     | `get_url(path="@/2025-01-29_dir-w-date/img.png")`       | <!-- {% raw() %}{{ get_url(path="@/2025-01-29_dir-w-date/img.png") }}{% end %} --> ERROR        |
