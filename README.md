# skink-vox

Natural-language processing for Skink: tokenization, sentence segmentation,
part-of-speech tagging, named-entity extraction, case conversion, and
readability/usage statistics.

A behavioral port of the Go `prose` toolkit (jdkato/prose v3), verified
byte-identical against its shipped test corpus — see `dump.skink` and the
differential harness in `../../tooling/prose-harness/` (outside this module;
the module itself is pure Skink).

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

## Quick start

```skink
import "mod/<owner>/skink-vox"

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
import "mod/<owner>/skink-vox/tokenize"
import "mod/<owner>/skink-vox/tag"
import "mod/<owner>/skink-vox/ner"

tk := tokenize.NewTokenizer()
toks := tk.Tokenize(text)
tg, _ := tag.New()
tg.TagTokens(toks)          // NER features need POS tags first
ex, _ := ner.New()
ex.Label(toks)
ents := ex.Chunk(text, toks)
```

Run the demo (`demo/main.skink`) for a full tour.

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

```sh
# Go reference dump (requires the Go toolchain and the prose source)
cd ../../tooling/prose-harness/go-dump && go build -o /tmp/go-dump . && /tmp/go-dump > /tmp/go.txt

# Skink dump — run from the module directory so ./models resolves
cd ../../modules/skink-vox && skink -o /tmp/sk-dump dump.skink && /tmp/sk-dump > /tmp/sk.txt

diff /tmp/go.txt /tmp/sk.txt   # expect no output
```

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
output for every corpus input; regenerate them with `../../tooling/prose-harness/go-golden`.
Skink has no coverage instrumentation yet, so coverage is approximated by
exhaustive branch tests plus the golden corpora — effectively every public
function and every internal code path exercised in the reference test suite
is asserted here.
