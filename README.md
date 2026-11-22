# docx-to-apkg

Turn one Chinese-reading Word file into one Anki package.

The skill extracts underlined words from a `.docx`, writes a 7-column vocabulary table, and builds a matching `.apkg`. Cards sit in Anki's built-in **Default** deck. The flow does not create extra decks.

Python 3 standard library only. It does not open your Anki collection.

## Layout

```
.
├── README.md
├── docx-to-apkg/                      Cursor skill
│   ├── SKILL.md
│   ├── scripts/
│   │   ├── docx_underlined_words.py
│   │   └── build_apkg.py
│   ├── references/chinese-vocab-tsv.md
│   ├── template/sample.apkg           SpoonFed++ mold, read only
│   ├── extract/                       underlined words, one per line
│   └── tsv/                           7-column vocabulary tables
└── output/                            finished .apkg files only
```

A reading uses the same `YYYY_MM_DD_STT` id for the extract, the TSV, and the apkg. `STT` is a two-digit sequence for that date (`01`, `02`, …). Never overwrite an existing id.

## Use as a Cursor skill

Point Cursor at `docx-to-apkg/SKILL.md`, then give it a `.docx` (or convert a `.doc` first). Optional: a reading date, or a request for traditional characters. Otherwise it uses today's date and simplified Chinese.

Do not supply a deck name. This skill does not create decks.

## Run the scripts by hand

Extract underlined words from the document body. Adjacent underlined runs become one phrase. The first occurrence of each word is kept.

```bash
python3 "docx-to-apkg/scripts/docx_underlined_words.py" "/path/to/file.docx" -o "docx-to-apkg/extract/2026_11_23_01.txt"
```

Words underlined only in headers, footnotes, or comments are skipped. If the list is empty, stop. Do not invent words.

Drop any extract line that already appears in the `Vocabulary` column of `docx-to-apkg/tsv/*.tsv`. Then write `docx-to-apkg/tsv/<id>.tsv` with this header:

```
Vocabulary	Pronunciation	English	Vietnamese	Example	Sentence Pinyin	Sentence Translation
```

Check that every row has seven tab-separated fields:

```bash
awk -F'\t' 'NF!=7{bad++} END{print NR" rows, "(bad+0)" bad"}' "docx-to-apkg/tsv/2026_11_23_01.tsv"
```

Build the package only after that check passes:

```bash
python3 "docx-to-apkg/scripts/build_apkg.py" "docx-to-apkg/tsv/2026_11_23_01.tsv"
```

That writes `output/<id>.apkg`. `Audio` and `SP` stay empty so HyperTTS can fill them later in Anki.

## Import into Anki

The package copies the **SpoonFed++** note type from `docx-to-apkg/template/sample.apkg` and keeps that note type's id, so Anki reuses the type you already have. A note is identified by note type plus Hanzi.

- New characters import as new notes.
- Characters already in Anki stay one note. If those notes already have audio, import with **Update notes: Never**. The default **If newer** overwrites the note and clears HyperTTS audio.

Leave `template/sample.apkg` untouched. To refresh the mold after you change SpoonFed++ in Anki, export one deck that uses only that note type, with **Support older Anki versions** ticked, and replace the file.

A `.doc` must be converted first:

```bash
textutil -convert docx FILE.doc
# or
soffice --headless --convert-to docx FILE.doc
```

## Requirements

- Python 3
- A `.docx` whose vocabulary is underlined in the document body
