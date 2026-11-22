---
name: docx-to-apkg
description: "Turn one Chinese-reading Word .docx into one Anki .apkg. Extract the underlined words, write a 7-column vocabulary TSV named YYYY_MM_DD_STT.tsv, then build a single .apkg of the same name whose cards sit in Anki's Default deck and create no extra decks. Use this skill whenever the user supplies a .docx or .doc and wants Anki cards, an apkg, a vocabulary TSV, underlined-word extraction, docx sang apkg, làm bộ thẻ, lọc từ gạch chân, or the underlined Chinese words from a Word file turned into flashcards, even if they never say the word skill."
compatibility: Python 3 standard library only. Reads template/sample.apkg for the SpoonFed++ note type. Does not open the user's Anki collection.
---

# DOCX underlined words to one Anki package

One Word file becomes one vocabulary table and one `.apkg`. The package is named after the table. Cards belong to Anki's built-in Default deck, because a card has to sit somewhere and this flow does not create a deck tree.

`<skill-dir>` is the directory containing this SKILL.md. Quote every path. File names may contain spaces.

## Layout

The workspace shows two folders. This skill is one. The other is `output/`, sitting beside the skill folder, and it is the only place a finished `.apkg` is written. Tables stay inside the skill so the import folder never fills up with working files.

```
<parent>/
├── docx-to-apkg/                      this skill
│   ├── SKILL.md
│   ├── scripts/
│   │   ├── docx_underlined_words.py
│   │   └── build_apkg.py
│   ├── references/chinese-vocab-tsv.md
│   ├── template/sample.apkg           SpoonFed++ mold, read only
│   ├── extract/YYYY_MM_DD_STT.txt     underlined words, one per line
│   └── tsv/YYYY_MM_DD_STT.tsv         7-column vocabulary table
└── output/YYYY_MM_DD_STT.apkg         the only finished file
```

`<skill-dir>` is `docx-to-apkg/`. `<output-dir>` is `<parent>/output`. A reading uses the same `YYYY_MM_DD_STT` for the extract, the TSV, and the apkg. `output/` receives nothing except `.apkg` files.

## What the user gives you

- Path to one `.docx`. A `.doc` must be converted first (`textutil -convert docx FILE.doc` on macOS, or `soffice --headless --convert-to docx FILE.doc`), then run on the `.docx`.
- Optional reading date. Otherwise use today's date in the user's timezone.
- Optional request for traditional characters. Otherwise write simplified Chinese.

Do not ask for a deck name. This skill does not create decks.

## Step 1 — Choose the reading id

Look at existing file names in `<skill-dir>/extract/`, `<skill-dir>/tsv/`, and `<output-dir>/` for today's date (or the date the user gave). `STT` is the next two-digit number after the highest one already used that day, or `01` when none exist. Never reuse or overwrite an id.

Example: `tsv/2026_11_23_01.tsv` already exists, so the next reading is `2026_11_23_02`.

## Step 2 — Extract underlined words

```bash
python3 "<skill-dir>/scripts/docx_underlined_words.py" "/path/to/file.docx" -o "<skill-dir>/extract/<id>.txt"
```

The script keeps the first occurrence of each word, merges adjacent underlined runs into one phrase, and prints the list. It only reads the document body, so words underlined in headers, footnotes, or comments are absent.

Empty list: stop. Delete the empty extract file if the script created one. Do not invent words, and do not write a TSV or an apkg.

## Step 3 — Drop words already packaged

Read the `Vocabulary` column of every `tsv/*.tsv`. Drop any extracted line that already appears there, and keep the original order of the rest. Anki identifies a note by note type plus Hanzi, so a repeated character would not become a new card on import.

If every line was dropped, stop and report the list. Do not write an empty table.

## Step 4 — Write the TSV

Read `references/chinese-vocab-tsv.md` and follow its column rules. The path rules in this file win over that reference: write `tsv/<id>.tsv`, not a file under `output/`.

- One row per remaining extract line, same order, no added or removed words. Keep a multi-character phrase as one row.
- Header, tab-separated, exactly these seven columns: `Vocabulary`, `Pronunciation`, `English`, `Vietnamese`, `Example`, `Sentence Pinyin`, `Sentence Translation`.
- `Vocabulary` is Chinese characters only. Pronunciation and sentence pinyin use tone marks. The example sentence must contain the vocabulary word.
- No tab characters inside a cell. `Audio` and `SP` stay empty so HyperTTS can fill them later in Anki.

Check the file before building. Row count must equal the kept word list plus the header, and every row must have 7 fields:

```bash
awk -F'\t' 'NF!=7{bad++} END{print NR" rows, "(bad+0)" bad"}' "<skill-dir>/tsv/<id>.tsv"
```

Fix the table when this check fails. Do not run the builder on a bad table.

## Step 5 — Build the package

```bash
python3 "<skill-dir>/scripts/build_apkg.py" "<skill-dir>/tsv/<id>.tsv"
```

The script writes `<output-dir>/<id>.apkg` and leaves `Audio` and `SP` empty. It copies the SpoonFed++ note type from `template/sample.apkg` and keeps that note type's id, so Anki reuses the note type the user already has. GUID stays `guid_for(note type id, Hanzi)`. Changing that function would make every imported word look new.

Leave `template/sample.apkg` untouched. It is a real export. If the user later changes SpoonFed++ in Anki, they replace this mold by exporting one deck that uses only that note type, with "Support older Anki versions" ticked.

If `<output-dir>/<id>.apkg` already exists, stop and ask before building again. A newer package imported with Anki's default "Update notes: If newer" overwrites the note and clears audio already added with HyperTTS. Rebuilding is safe only when the user says to rebuild, and the import should then use "Update notes: Never" so old audio survives.

## Report

Reply briefly with:

- how many underlined items were extracted
- which items were dropped as duplicates, if any
- path of the extract file, the TSV, and the apkg
- the 7-column check result
- one import line: new characters import normally; characters already in Anki stay one note, so choose "Update notes: Never" when those notes already have audio
