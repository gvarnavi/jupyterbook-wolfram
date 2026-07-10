# jupyterbook-wolfram

A [MyST / Jupyter Book](https://mystmd.org) site with **interactive Wolfram Language widgets**, rendered client-side from the Wolfram Cloud — readers need no Wolfram Engine, and neither do co-authors or CI.

## How it works

The executable MyST plugin [plugins/wolfram.py](plugins/wolfram.py) registers two directives and a document transform:

- `{wolfram}` — a Wolfram Language cell. All `{wolfram}` cells on a page share one kernel session at deploy time and are evaluated in document order, so later cells see earlier definitions. Tag the expression to deploy with a full-line comment marker:

  ```
  :::{wolfram}
  :echo: true

  (* #| label: app:my-widget *)
  Manipulate[Plot[Sin[a x], {x, 0, 10}], {a, 1, 5}]
  :::
  ```

  `(* #| deploy *)` deploys without a label; a label makes the widget a MyST cross-reference target, and adding `:caption:` wraps it in a numbered figure. Untagged cells are definitions-only (no widget).

- `{wolfram-notebook}` — embed a full notebook, either a public Wolfram Cloud URL or a repo-local `.nb` file.

Each deployable cell resolves to a public Wolfram Cloud object recorded in the committed manifest [.jupyter-book-wolfram/deployments.json](.jupyter-book-wolfram/deployments.json), keyed by a content digest. The widget itself is rendered in the browser by [widgets/wolfram-notebook.mjs](widgets/wolfram-notebook.mjs) via the wolfram-notebook-embedder.

## Building

```sh
myst start        # or: myst build
```

No Wolfram tooling is needed for a normal build: cached deployments come from the manifest, and any cache miss renders a warning admonition instead of failing.

### Deploying changed cells

Deployment is strictly opt-in and needs a licensed `wolframscript` on PATH:

```sh
JUPYTER_BOOK_WOLFRAM_DEPLOY=1 myst build
```

New/changed cells are published with `CloudPublish` and the manifest is updated — commit it. Cell digests are *chained* (cell N's digest covers cells 1..N on the page), so editing an earlier cell invalidates later deployments on that page.

You can also record an existing cloud object by hand:

```sh
plugins/wolfram.py --record <url> --digest <digest-from-placeholder-box>
```

## Authoring: converting a notebook to a page

`--scaffold` extracts a `.nb` into an editable MyST page (needs `wolframscript`):

```sh
plugins/wolfram.py --scaffold notebooks/classical-mechanics.nb -o 02.my_page.md
```

The conversion is best-effort and made for hand-finishing:

- Headings, prose, and list cells become markdown; the first title/chapter heading becomes the page title.
- Code cells become `{wolfram}` directives — add a `(* #| deploy/label *)` marker to the ones you want deployed.
- **TeX-assistant equations** (`TemplateBox[..., "TeXAssistantTemplate"]`) are replaced with their LaTeX source: inline `$...$` when mixed into a sentence, a `:::{math}` block when the equation is the entire cell.
- **Opaque payloads** (`GraphicsBox`, `CompressedData`, `FrameBox`, and friends) are elided and replaced with a comment saying what was omitted — pasted graphics can be tens of kilobytes of box data that no human should scroll past.
- Typeset cells and unknown styles are kept under a `<!-- TODO -->` comment for review, never silently dropped.

## Repository layout

| Path | What it is |
| --- | --- |
| `myst.yml` | MyST project config; registers the plugin |
| `plugins/wolfram.py` | Directives, transform, deploy logic, and authoring CLI |
| `widgets/wolfram-notebook.mjs` | Client-side anywidget renderer |
| `.jupyter-book-wolfram/deployments.json` | Committed manifest of deployed cloud objects |
| `notebooks/` | Source `.nb` notebooks for scaffolding/embedding |
| `01.classical_mechanics.md` | Example chapter |

## Environment variables

| Variable | Effect |
| --- | --- |
| `JUPYTER_BOOK_WOLFRAM_DEPLOY=1` | Enable cloud deployment during build |
| `JUPYTER_BOOK_WOLFRAM_CLOUD_PATH` | Cloud object path prefix (default `myst-widgets`) |
| `JUPYTER_BOOK_WOLFRAM_CLOUD_BASE` | Alternate `CloudBase` |
| `JUPYTER_BOOK_WOLFRAM_ESM` | Override the widget ESM module path |
