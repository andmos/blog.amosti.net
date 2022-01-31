+++
author = "Andreas Mosti"
date = 2019-01-05T15:04:00Z
description = ""
draft = false
slug = "ensure-consistent-markdown-style-with-markdownlint"
title = "Ensure consistent Markdown style with Markdownlint"

+++


Markdown is great. It's easy and flexible, and provides a good markup language even non-technical people can understand and enjoy. But, that flexibility and customizability can come at a cost. Document buildup can be done in many ways, and it can be hard to ensure consitency when working with multiple documents and contributors.

I like to think of markup languages as code, and most code deserves a good style guide. [Markdownlint](https://github.com/DavidAnson/markdownlint) is a good alternative.

`markdownlint` provides [a nice set of standard rules](https://github.com/DavidAnson/markdownlint/blob/master/doc/Rules.md) when writing markdown, like:

* Heading levels should only increment by one level at a time
* Lists should be surrounded by blank lines
* First line in file should be a top level heading
* No empty links
* No trailing spaces
* No multiple consecutive blank lines

to name a few. It also ensures concistensy in headers, like:

```markdown
My Heading
===
```

vs.

```markdown
# My Heading
```

Another smart rule is ensuring language description when writing code blocks.

If some rules don't fit your style or project, they can be overrided with a `.markdownlint.json` file:

```markdown
{
    "MD013": false, // Disable line length rule.  
    "MD024": false // Allow Multiple headings with the same content.
}
```

The easiest way to start using `markdownlint` is to install the extention for [VSCode](https://marketplace.visualstudio.com/items?itemName=DavidAnson.vscode-markdownlint) or [Atom](https://atom.io/packages/linter-node-markdownlint), or integrated with builds using [Grunt](https://github.com/sagiegurari/grunt-markdownlint) or [markdownlint-cli](https://github.com/igorshubovych/markdownlint-cli).

For my [Coffee recipes](https://github.com/andmos/Coffee) I use a simple container with Travis:

```yaml
sudo: required
language: generic

services:
  - docker
script:
  - docker run --rm -v $(pwd):/usr/src/app/files -it andmos/markdownlint *.md
```

If any rules are broken, it breaks the build.