# skink-vox

Natural-language processing for Skink: tokenization, sentence segmentation,
part-of-speech tagging, named-entity extraction, case conversion, and
readability/usage statistics.

A behavioral port of the Go `prose` toolkit (jdkato/prose v3), verified
byte-identical against its shipped test corpus. The differential harness
lives in the separate `prose-harness` repository (not part of this
checkout); `demo/main.skink` gives a local tour of the pipeline. The
module itself is pure Skink.

## Modules

| module      | contents                                                        |
|-------------|-----------------------------------------------------------------|
| `vox`       | `NewDocument` pipeline: tokens, sentences, tags, entities        |
| `tokenize`  | `Tokenizer`, `TreebankWordTokenizer`, `WordBoundaryTokenizer`, `WordPunctTokenizer`, `BlanklineTokenizer`, `RegexpTokenizer` |
| `segment`   | Punkt sentence `Segmenter`                                       |
| `tag`       | Perceptron POS `Tagger`, `Lexical` dictionary tagger, registry   |
| `ner`       | Maximum-entropy `Extracter` — `Label` + `Chunk`                  |
| `strcase`   | `Camel`/`Pascal`/`Snake`/`Constant`/`Dash`/`Dot`/`Simple`, AP & Chicago title converters, sentence case |
| `summarize` | `Document` metrics, readability formulas, `Syllables`            |
| `token`     | Shared `Token` type                                              |

## Installation

vox is distributed through `scales`, the Skink package manager. Inside a
project directory (one containing a `skink.toml` — create one with
`scales init` if needed):

```sh
scales add mod/markoxley/skink-vox            # latest, records it in skink.toml
scales add mod/markoxley/skink-vox@v1.0.0     # pin a specific tag
```

This fetches the package into the shared scales cache
(`~/.local/skink/scales/packages/...`) and writes `skink.toml` +
`skink.lock`. `scales install` restores the cache on a fresh machine —
no sources are copied into your project.

## Quick start

```skink
import "mod/markoxley/skink-vox"

fn main() {
    doc, err := vox.NewDocument("Marie Curie studied in Paris.")
    if err.message != "" {
        println(err.message)
        return
    }
    for i := 0; i < len(doc.Entities()); i = i + 1 {
        e := doc.Entities()[i]
        println(e.Label, ": ", e.Text)   // PERSON: Marie Curie
    }
}
```

Individual stages import directly:

```skink
import "mod/markoxley/skink-vox/tokenize"
import "mod/markoxley/skink-vox/tag"
import "mod/markoxley/skink-vox/ner"

tk := tokenize.NewTokenizer()
toks := tk.Tokenize(text)
tg, _ := tag.New()
tg.TagTokens(toks)          // NER features need POS tags first
ex, _ := ner.New()
ex.Label(toks)
ents := ex.Chunk(text, toks)
```

Run the demo for a full tour:

```sh
skink -o /tmp/vox-demo demo/main.skink && /tmp/vox-demo
```

## Models

The model files under `models/` (`tag/en.bin`, `ner/en.bin`,
`punkt/en.json`) are baked into the compiled binary by `embed "models"`
declarations in the model-loading modules (`vox`, `tag`, `ner`, `segment`),
so a binary that links any of them is fully self-contained — like the Go
original, which embeds the same models. Note the binary grows by ~8 MB.

At load time `internal/modelpath` resolves each model by searching, in
order:

1. `$SKINK_VOX_MODELS` — a directory containing the files; an explicit
   override for swapping in other models
2. The embedded filesystem — `int:models/...`
3. `./models/` or `./skink-vox/models/` — running from the repo
4. `mod/` or `vendor/` copies in the current directory
5. The scales package cache (`$SCALES_CACHE_DIR`, `$SKINK_PREFIX/scales`, or
   `~/.local/skink/scales`)

Steps 3–5 exist so builds that bypass the embed declarations (for example,
tools compiled from a partial checkout) still find the data files.

## Verifying parity

Byte-for-byte parity with the Go reference is checked by the differential
harness in the separate `prose-harness` repository: `go-dump` dumps the
reference pipeline's output, a matching Skink dump program emits the same
records for every corpus input, and the two outputs are `diff`ed. That
harness is not part of this checkout; the same coverage is exercised here
by the `*_golden.json` corpora under `testdata/` (see Tests below).

## Tests

Run the full unit-test suite (10 files) from this directory:

```sh
for f in *_test.skink; do skink -t "$f"; done
```

The suite covers every public module plus the internal helpers:

- `token_test.skink` — offsets, `End()`/`In()`, repeated/Unicode tokens
- `tokenize_test.skink` — all tokenizer variants, options, the
  `offsets_golden.json`/`treebank_golden.json` corpora
- `segment_test.skink` — golden rules + corpus goldens, offset invariants
- `tag_test.skink` — model loading, registry, lexicons, `Lexical`,
  `tag_golden.json` corpus
- `ner_test.skink` — labeling, chunking, `ner_golden.json` corpus
- `strcase_test.skink` — mechanical + sentence/title converters, AP/Chicago
- `summarize_test.skink` — readability metrics, syllables (golden +
  accuracy), density/keywords
- `vox_test.skink` — document pipeline, stage toggles, accessors
- `internal_test.skink` — mini-regex, unicode, punkt internals, flat/maxent
  readers, model-path lookup, error paths
- `skink_vox_test.skink` — end-to-end regression

The `*_golden.json` files in `testdata/` record the exact reference-implementation
output for every corpus input; they are generated by the `go-golden` tool
in the `prose-harness` repository. Coverage is approximated by exhaustive
branch tests plus the golden corpora — effectively every public function
and every internal code path exercised in the reference test suite is
asserted here (`skink -cov` + `tools/skinkcov` can also instrument runs).
