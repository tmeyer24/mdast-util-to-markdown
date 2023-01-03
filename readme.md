# mdast-util-to-markdown

[![Build][build-badge]][build]
[![Coverage][coverage-badge]][coverage]
[![Downloads][downloads-badge]][downloads]
[![Size][size-badge]][size]
[![Sponsors][sponsors-badge]][collective]
[![Backers][backers-badge]][collective]
[![Chat][chat-badge]][chat]

**[mdast][]** utility that turns a syntax tree into markdown.

## Contents

*   [What is this?](#what-is-this)
*   [When should I use this?](#when-should-i-use-this)
*   [Install](#install)
*   [Use](#use)
*   [API](#api)
    *   [`toMarkdown(tree[, options])`](#tomarkdowntree-options)
    *   [`Handle`](#handle)
    *   [`Handlers`](#handlers)
    *   [`Info`](#info)
    *   [`Join`](#join)
    *   [`Map`](#map)
    *   [`Options`](#options)
    *   [`State`](#state)
    *   [`Unsafe`](#unsafe)
*   [List of extensions](#list-of-extensions)
*   [Syntax](#syntax)
*   [Syntax tree](#syntax-tree)
*   [Types](#types)
*   [Security](#security)
*   [Related](#related)
*   [Contribute](#contribute)
*   [License](#license)

## What is this?

This package is a utility that takes an [mdast][] syntax tree as input and turns
it into serialized markdown.

This utility is a low level project.
It’s used in [`remark-stringify`][remark-stringify], which focusses on making it
easier to transform content by abstracting these internals away.

## When should I use this?

If you want to handle syntax trees manually, use this.
For an easier time processing content, use the **[remark][]** ecosystem instead.

You can combine this utility with other utilities to add syntax extensions.
Notable examples that deeply integrate with it are
[`mdast-util-gfm`][mdast-util-gfm],
[`mdast-util-mdx`][mdast-util-mdx],
[`mdast-util-frontmatter`][mdast-util-frontmatter],
[`mdast-util-math`][mdast-util-math], and
[`mdast-util-directive`][mdast-util-directive].

## Install

This package is [ESM only][esm].
In Node.js (version 14.14+, or 16.0+), install with [npm][]:

```sh
npm install mdast-util-to-markdown
```

In Deno with [`esm.sh`][esmsh]:

```js
import {toMarkdown} from 'https://esm.sh/mdast-util-to-markdown@1'
```

In browsers with [`esm.sh`][esmsh]:

```html
<script type="module">
  import {toMarkdown} from 'https://esm.sh/mdast-util-to-markdown@1?bundle'
</script>
```

## Use

Say our module `example.js` looks as follows:

```js
import {toMarkdown} from 'mdast-util-to-markdown'

/** @type {import('mdast').Root} */
const tree = {
  type: 'root',
  children: [
    {
      type: 'blockquote',
      children: [
        {type: 'thematicBreak'},
        {
          type: 'paragraph',
          children: [
            {type: 'text', value: '- a\nb !'},
            {
              type: 'link',
              url: 'example.com',
              children: [{type: 'text', value: 'd'}]
            }
          ]
        }
      ]
    }
  ]
}

console.log(toMarkdown(tree))
```

…now running `node example.js` yields:

```markdown
> ***
>
> \- a
> b \![d](example.com)
```

> 👉 **Note**: observe the properly escaped characters which would otherwise
> turn into a list and image respectively.

## API

This package exports the identifier `toMarkdown`.
There is no default export.

### `toMarkdown(tree[, options])`

Turn an **[mdast][]** syntax tree into markdown.

###### Parameters

*   `tree` ([`Node`][node])
    — tree to serialize
*   `options` ([`Options`][options], optional)
    — configuration

###### Returns

Serialized markdown representing `tree` (`string`).

### `Handle`

Handle a particular node (TypeScript type).

###### Parameters

*   `node` (`any`)
    — expected mdast node
*   `parent` ([`Node`][node], optional)
    — parent of `node`
*   `state` ([`State`][state])
    — info passed around about the current state
*   `info` ([`Info`][info])
    — info on the surrounding of the node that is serialized

###### Returns

Serialized markdown representing `node` (`string`).

### `Handlers`

Handle particular nodes (TypeScript type).

Each key is a node type (`string`), each value its corresponding handler
([`Handle`][handle]).

###### Type

```ts
Record<string, Handle>
```

### `Info`

Info on the surrounding of the node that is serialized (TypeScript type).

###### Fields

*   `now` ([`Point`][point])
    — current point
*   `lineShift` (`number`)
    — number of columns each line will be shifted by wrapping nodes
*   `before` (`string`)
    — characters before this (guaranteed to be one, can be more)
*   `after` (`string`)
    — characters after this (guaranteed to be one, can be more)

### `Join`

How to join two blocks (TypeScript type).

“Blocks” are typically joined by one blank line.
Sometimes it’s nicer to have them flush next to each other, yet other times
they cannot occur together at all.

Join functions receive two adjacent siblings and their parent and what they
return defines how many blank lines to use between them.

###### Parameters

*   `left` ([`Node`][node])
    — first of two adjacent siblings
*   `right` ([`Node`][node])
    — second of two adjacent siblings
*   `parent` ([`Node`][node])
    — parent of the two siblings
*   `state` ([`State`][state])
    — info passed around about the current state

###### Returns

How many blank lines to use between the siblings (`boolean`, `number`,
optional).

Where `true` is as passing `1` and `false` means the nodes cannot be
joined by a blank line, such as two adjacent block quotes or indented code
after a list, in which case a comment will be injected to break them up:

```markdown
> Quote 1

<!---->

> Quote 2
```

> 👉 **Note**: abusing this feature will break markdown.
> One such example is when returning `0` for two paragraphs, which will result
> in the text running together, and in the future to be seen as one paragraph.

### `Map`

### `Options`

Configuration (TypeScript type).

##### Fields

The following fields influence how markdown is serialized.

###### `options.bullet`

Marker to use for bullets of items in unordered lists (`'*'`, `'+'`, or `'-'`,
default: `'*'`).

###### `options.bulletOther`

Marker to use in certain cases where the primary bullet doesn’t work (`'*'`,
`'+'`, or `'-'`, default: depends).

There are three cases where the primary bullet cannot be used:

*   When three list items are on their own, the last one is empty, and `bullet`
    is also a valid `rule`: `* - +`.
    This would turn into a thematic break if serialized with three primary
    bullets.
    As this is an edge case unlikely to appear in normal markdown, the last list
    item will be given a different bullet.
*   When a thematic break is the first child of one of the list items, and
    `bullet` is the same character as `rule`: `- ***`.
    This would turn into a single thematic break if serialized with primary
    bullets.
    As this is an edge case unlikely to appear in normal markdown this markup is
    always fixed, even if `bulletOther` is not passed
*   When two unordered lists appear next to each other: `* a\n- b`.
    CommonMark sees different bullets as different lists, but several markdown
    parsers parse it as one list.
    To solve for both, we instead inject an empty comment between the two lists:
    `* a\n<!---->\n* b`, but if `bulletOther` is given explicitly, it will be
    used instead

###### `options.bulletOrdered`

Marker to use for bullets of items in ordered lists (`'.'` or `')'`, default:
`'.'`).

###### `options.bulletOrderedOther`

Marker to use in certain cases where the primary bullet for ordered items
doesn’t work (`'.'` or `')'`, default: none).

There is one case where the primary bullet for ordered items cannot be used:

*   When two ordered lists appear next to each other: `1. a\n2) b`.
    CommonMark added support for `)` as a marker, but other markdown parsers
    do not support it.
    To solve for both, we instead inject an empty comment between the two lists:
    `1. a\n<!---->\n1. b`, but if `bulletOrderedOther` is given explicitly, it
    will be used instead

###### `options.closeAtx`

Whether to add the same number of number signs (`#`) at the end of an ATX
heading as the opening sequence (`boolean`, default: `false`).

###### `options.emphasis`

Marker to use for emphasis (`'*'` or `'_'`, default: `'*'`).

###### `options.fence`

Marker to use for fenced code (``'`'`` or `'~'`, default: ``'`'``).

###### `options.fences`

Whether to use fenced code always (`boolean`, default: `false`).
The default is to use fenced code if there is a language defined, if the code is
empty, or if it starts or ends in blank lines.

###### `options.incrementListMarker`

Whether to increment the counter of ordered lists items (`boolean`, default:
`true`).

###### `options.listItemIndent`

How to indent the content of list items (`'one'`, `'tab'`, or `'mixed'`,
default: `'tab'`).
Either with the size of the bullet plus one space (when `'one'`), a tab stop
(`'tab'`), or depending on the item and its parent list (`'mixed'`, uses `'one'`
if the item and list are tight and `'tab'` otherwise).

###### `options.quote`

Marker to use for titles (`'"'` or `"'"`, default: `'"'`).

###### `options.resourceLink`

Whether to always use resource links (`boolean`, default: `false`).
The default is to use autolinks (`<https://example.com>`) when possible
and resource links (`[text](url)`) otherwise.

###### `options.rule`

Marker to use for thematic breaks (`'*'`, `'-'`, or `'_'`, default: `'*'`).

###### `options.ruleRepetition`

Number of markers to use for thematic breaks (`number`, default: `3`, min: `3`).

###### `options.ruleSpaces`

Whether to add spaces between markers in thematic breaks (`boolean`, default:
`false`).

###### `options.setext`

Whether to use setext headings when possible (`boolean`, default: `false`).
The default is to always use ATX headings (`# heading`) instead of setext
headings (`heading\n=======`).
Setext headings cannot be used for empty headings or headings with a rank of
three or more.

###### `options.strong`

Marker to use for strong (`'*'` or `'_'`, default: `'*'`).

###### `options.tightDefinitions`

Whether to join definitions without a blank line (`boolean`, default: `false`).

The default is to add blank lines between any flow (“block”) construct.
Turning this option on is a shortcut for a [`Join`][join] function like so:

```js
function joinTightDefinitions(left, right) {
  if (left.type === 'definition' && right.type === 'definition') {
    return 0
  }
}
```

###### `options.handlers`

Handle particular nodes ([`Handlers`][handlers], optional).

###### `options.join`

How to join blocks ([`Array<Join>`][join], optional).

###### `options.unsafe`

Schemas that define when characters cannot occur ([`Array<Unsafe>`][unsafe],
optional).

###### `options.extensions`

List of extensions (`Array<Options>`, default: `[]`).
Each extension is an object with the same interface as `Options` itself.

### `State`

Info passed around about the current state (TypeScript type).

###### Fields

*   `stack` (`Array<string>`)
    — stack of constructs we’re in
*   `indexStack` (`Array<number>`)
    — positions of child nodes in their parents
*   `enter` (`(construct: string) => () => void`)
    — enter a construct (returns a corresponding exit function)
*   `options` ([`Options`][options])
    — applied user configuration
*   `unsafe` ([`Array<Unsafe>`][unsafe])
    — applied unsafe patterns
*   `join` ([`Array<Join>`][join])
    — applied join handlers
*   `handle` ([`Handle`][handle])
    — call the configured handler for the given node
*   `handlers` ([`Handlers`][handlers])
    — applied handlers
*   `bulletCurrent` (`string` or `undefined`)
    — list marker currently in use
*   `bulletLastUsed` (`string` or `undefined`)
    — list marker previously in use

### `Unsafe`

Schema that defines when a character cannot occur (TypeScript type).

###### Fields

*   `character` (`string`)
    — single unsafe character
*   `inConstruct` (`string`, `Array<string>`, optional)
    — constructs where this is bad
*   `notInConstruct` (`string`, `Array<string>`, optional)
    — constructs where this is fine again
*   `before` (`string`, optional)
    — `character` is bad when this is before it (cannot be used together with
    `atBreak`)
*   `after` (`string`, optional)
    — `character` is bad when this is after it
*   `atBreak` (`boolean`, optional)
    — `character` is bad at a break (cannot be used together with `before`)

## List of extensions

*   [`syntax-tree/mdast-util-directive`](https://github.com/syntax-tree/mdast-util-directive)
    — directives
*   [`syntax-tree/mdast-util-frontmatter`](https://github.com/syntax-tree/mdast-util-frontmatter)
    — frontmatter (YAML, TOML, more)
*   [`syntax-tree/mdast-util-gfm`](https://github.com/syntax-tree/mdast-util-gfm)
    — GFM
*   [`syntax-tree/mdast-util-gfm-autolink-literal`](https://github.com/syntax-tree/mdast-util-gfm-autolink-literal)
    — GFM autolink literals
*   [`syntax-tree/mdast-util-gfm-footnote`](https://github.com/syntax-tree/mdast-util-gfm-footnote)
    — GFM footnotes
*   [`syntax-tree/mdast-util-gfm-strikethrough`](https://github.com/syntax-tree/mdast-util-gfm-strikethrough)
    — GFM strikethrough
*   [`syntax-tree/mdast-util-gfm-table`](https://github.com/syntax-tree/mdast-util-gfm-table)
    — GFM tables
*   [`syntax-tree/mdast-util-gfm-task-list-item`](https://github.com/syntax-tree/mdast-util-gfm-task-list-item)
    — GFM task list items
*   [`syntax-tree/mdast-util-math`](https://github.com/syntax-tree/mdast-util-math)
    — math
*   [`syntax-tree/mdast-util-mdx`](https://github.com/syntax-tree/mdast-util-mdx)
    — MDX
*   [`syntax-tree/mdast-util-mdx-expression`](https://github.com/syntax-tree/mdast-util-mdx-expression)
    — MDX expressions
*   [`syntax-tree/mdast-util-mdx-jsx`](https://github.com/syntax-tree/mdast-util-mdx-jsx)
    — MDX JSX
*   [`syntax-tree/mdast-util-mdxjs-esm`](https://github.com/syntax-tree/mdast-util-mdxjs-esm)
    — MDX ESM

## Syntax

Markdown is serialized according to CommonMark but care is taken to format in
such a way that the resulting markdown should work with most markdown parsers.
Extensions can add support for custom syntax.

## Syntax tree

The syntax tree is [mdast][].

## Types

This package is fully typed with [TypeScript][].
It exports the additional types `Handle`, `Handlers`, `Info` `Join`, `Map`,
`Options`, `State`, and `Unsafe`.

## Security

`mdast-util-to-markdown` will do its best to serialize markdown to match the
syntax tree, but there are several cases where that is impossible.
It’ll do its best, but complete roundtripping is impossible given that any value
could be injected into the tree.

As markdown is sometimes used for HTML, and improper use of HTML can open you up
to a [cross-site scripting (XSS)][xss] attack, use of `mdast-util-to-markdown`
and parsing it again later could potentially be unsafe.
When parsing markdown afterwards and then going to HTML, use something like
[`hast-util-sanitize`][hast-util-sanitize] to make the tree safe.

## Related

*   [`syntax-tree/mdast-util-from-markdown`](https://github.com/syntax-tree/mdast-util-from-markdown)
    — parse markdown to mdast
*   [`micromark/micromark`](https://github.com/micromark/micromark)
    — parse markdown
*   [`remarkjs/remark`](https://github.com/remarkjs/remark)
    — process markdown

## Contribute

See [`contributing.md`][contributing] in [`syntax-tree/.github`][health] for
ways to get started.
See [`support.md`][support] for ways to get help.

This project has a [code of conduct][coc].
By interacting with this repository, organization, or community you agree to
abide by its terms.

## License

[MIT][license] © [Titus Wormer][author]

<!-- Definitions -->

[build-badge]: https://github.com/syntax-tree/mdast-util-to-markdown/workflows/main/badge.svg

[build]: https://github.com/syntax-tree/mdast-util-to-markdown/actions

[coverage-badge]: https://img.shields.io/codecov/c/github/syntax-tree/mdast-util-to-markdown.svg

[coverage]: https://codecov.io/github/syntax-tree/mdast-util-to-markdown

[downloads-badge]: https://img.shields.io/npm/dm/mdast-util-to-markdown.svg

[downloads]: https://www.npmjs.com/package/mdast-util-to-markdown

[size-badge]: https://img.shields.io/bundlephobia/minzip/mdast-util-to-markdown.svg

[size]: https://bundlephobia.com/result?p=mdast-util-to-markdown

[sponsors-badge]: https://opencollective.com/unified/sponsors/badge.svg

[backers-badge]: https://opencollective.com/unified/backers/badge.svg

[collective]: https://opencollective.com/unified

[chat-badge]: https://img.shields.io/badge/chat-discussions-success.svg

[chat]: https://github.com/syntax-tree/unist/discussions

[npm]: https://docs.npmjs.com/cli/install

[esmsh]: https://esm.sh

[license]: license

[author]: https://wooorm.com

[health]: https://github.com/syntax-tree/.github

[contributing]: https://github.com/syntax-tree/.github/blob/main/contributing.md

[support]: https://github.com/syntax-tree/.github/blob/main/support.md

[coc]: https://github.com/syntax-tree/.github/blob/main/code-of-conduct.md

[esm]: https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c

[typescript]: https://www.typescriptlang.org

[xss]: https://en.wikipedia.org/wiki/Cross-site_scripting

[hast-util-sanitize]: https://github.com/syntax-tree/hast-util-sanitize

[point]: https://github.com/syntax-tree/unist#point

[mdast]: https://github.com/syntax-tree/mdast

[node]: https://github.com/syntax-tree/mdast#nodes

[mdast-util-gfm]: https://github.com/syntax-tree/mdast-util-gfm

[mdast-util-mdx]: https://github.com/syntax-tree/mdast-util-mdx

[mdast-util-frontmatter]: https://github.com/syntax-tree/mdast-util-frontmatter

[mdast-util-math]: https://github.com/syntax-tree/mdast-util-math

[mdast-util-directive]: https://github.com/syntax-tree/mdast-util-directive

[remark]: https://github.com/remarkjs/remark

[remark-stringify]: https://github.com/remarkjs/remark/tree/main/packages/remark-stringify

[handle]: #handle

[handlers]: #handlers

[info]: #info

[join]: #join

[options]: #options

[state]: #state

[unsafe]: #unsafe
