# Converting an OCR'd PDF into CB-Essay chapters

These scripts turn the text layer of an OCR'd book PDF into one Markdown file per chapter for the `_essay/` folder. They were built for *In Nez Perce Country*, but everything specific to that book lives in a YAML config file, so the same scripts can be reused for another book.

| File | What it does |
|------|--------------|
| `ocr-pages.rb` | Lists every PDF page with its word count and first two lines. Use the list to fill in the config. |
| `ocr-configs/*.yml` | One config per book: chapter list and start pages, running header, pages to skip, OCR corrections, and structure patterns. Start a new one from `template.yml`. |
| `convert-ocr.rb` | Reads a config plus the extracted text and writes the chapter files. |

The output is a strong first draft, not a finished edition. Plan on a hand-editing pass afterwards.

## Requirements

- **A PDF with a text layer.** The scripts read existing OCR text and don't run OCR themselves. To check, open the PDF and try to select a word. If you can't, run it through OCR first (for example Adobe Acrobat's *Recognize Text*, or `ocrmypdf`).
- **`pdftotext`**, from Xpdf or Poppler. It comes with Git for Windows. Run `pdftotext -v` to check.
- **Ruby**, already needed for Jekyll. Run `ruby -v` to check.

Run every command below from the repository root, in Git Bash on Windows or a terminal on Mac/Linux.

## Step 1: Set up a work folder

Choose a short name for the book. These examples use `my-book`; replace it with your own name everywhere.

```bash
mkdir -p ocr-work/my-book
cp /path/to/my-book.pdf ocr-work/my-book/book.pdf
```

Add `ocr-work/` to `.gitignore` so extracted text and scratch output never get committed.

## Step 2: Extract the text

Run both commands, because the converter needs both files:

```bash
pdftotext -enc UTF-8 ocr-work/my-book/book.pdf ocr-work/my-book/book.txt
pdftotext -enc UTF-8 -layout ocr-work/my-book/book.pdf ocr-work/my-book/book-layout.txt
```

- `book.txt` is the main source. On most pages each paragraph comes out as one line.
- `book-layout.txt` keeps the printed layout. The converter uses it only for pages where the plain extraction breaks text into single printed lines, and rebuilds paragraphs from the indentation.
- Keep `-enc UTF-8`. The script fails on the default encoding.

## Step 3: Survey the pages

```bash
ruby scripts/ocr-pages.rb ocr-work/my-book/book.txt > ocr-work/my-book/pages.txt
```

Each row of `pages.txt` shows the PDF page number, the word count and the page's first two lines:

```
12    410w    II | Trails to the Buffalo Country
13    520w    6 • In Nez Perce Country | "Coyote the Expeditioner" ...
15    18w     00 | ~
```

Write down:

1. **The running header:** the text repeated at the top of most pages (row 13). Copy it exactly as the OCR spelled it, mistakes included.
2. **Chapter start pages:** rows *without* the running header whose second line is a chapter title (row 12). The first line is usually the chapter number, often garbled.
3. **Image-only pages:** very low word counts or garbled text (row 15), such as maps, plates and photos.
4. **The end page:** the last page before the index or other back matter you want to leave out.

The book's printed contents page, usually around PDF pages 3–6, is the easiest source for chapter titles.

> Use the **PDF page numbers** from `pages.txt`, not the numbers printed on the pages. They often drift apart because of plates, blank pages or missing pages.

## Step 4: Write the config

Start from the template:

```bash
cp scripts/ocr-configs/template.yml scripts/ocr-configs/my-book.yml
```

Then replace every placeholder (`0` page numbers, the sample chapter titles, the header text). The template has comments throughout, and the optional blocks are commented out. For a filled-in example, see `in-nez-perce-country.yml`. A finished minimal config looks like this:

```yaml
running_header: 'My Book Title'

chapters:
  - { title: "Preface", page: 9, slug: preface, part: Front Matter }
  - { title: "The First Chapter", page: 14, slug: first-chapter }
  - { title: "The Second Chapter", page: 31, slug: second-chapter }

end_page: 240
skip_pages: [22, 23, 87]

footnote_numbering: per_chapter

text_fixes: []
keep_apart: []
```

### Config reference

| Key | Required | Meaning |
|-----|----------|---------|
| `running_header` | yes | Ruby regex for the page header. Use `\s*` where the OCR adds stray spaces, e.g. `'In Nez Perce Co\s*un\s*try'`. Keep regexes in single quotes. |
| `chapters` | yes | Chapters in reading order. Each runs from its `page` up to the page before the next chapter's `page`. |
| `chapters[].title` | yes | Becomes the essay's `title`. |
| `chapters[].page` | yes | PDF page where the chapter starts. |
| `chapters[].slug` | yes | The filename becomes `NN-slug.md`, where `NN` is the chapter's position in the list. |
| `chapters[].order` | no | Front-matter `order`. Defaults to position × 10 (10, 20, 30…). |
| `chapters[].part` | no | Front-matter `part`, which groups entries in the table of contents. Defaults to `Chapter N`. |
| `end_page` | yes | Last PDF page of the final chapter. |
| `skip_pages` | no | PDF pages to ignore entirely. |
| `footnote_numbering` | no | `per_chapter` (the default) if notes restart at 1 in each chapter; `continuous` if they run through the whole book. |
| `text_fixes` | no | `{ find, replace }` regex pairs applied to every paragraph and footnote, in order, for OCR errors specific to this book. `replace` can use `\1`. |
| `keep_apart` | no | Words the split-word rejoiner must never create. See Step 6. |
| `quoted_sections` | no | For books with labelled primary-source excerpts (e.g. `Account 5.1: … Source: …`). See below. |
| `reading_list_pattern` | no | Regex for a paragraph that starts a further-reading list. Group 1 becomes a `####` heading, and the short citation paragraphs after it become bullet items. |

**`quoted_sections`** has two keys:

- `pattern`: a regex matching the paragraph that opens a quoted section. Group 1 becomes a `###` heading and group 2 an italic description.
- `source_label`: optional, e.g. `'Source:'`. Text after this label becomes a bold **Source:** line.

Each section starts with `{% include essay/new-section.html %}`, and the paragraphs after it are block-quoted until the next section. Both `quoted_sections` and `reading_list_pattern` are commented out in the template. **Uncomment them only if the new book has labelled excerpts or reading lists, and adapt the regex to match.** The sample values match *In Nez Perce Country*.

## Step 5: Run the converter

Always write to a scratch folder, never straight into `_essay/`:

```bash
ruby scripts/convert-ocr.rb scripts/ocr-configs/my-book.yml \
  ocr-work/my-book/book.txt ocr-work/my-book/book-layout.txt ocr-work/my-book/out
```

The arguments are, in order: config, plain text, layout text, output folder.

The script prints one line per chapter, plus a warning for each footnote it couldn't link into the text:

```
01-preface.md: 12 paras, 3 notes (1-3)
first-chapter: no reference found for note 4
02-first-chapter.md: 88 paras, 9 notes (1-9)
```

It also writes `ocr-work/my-book/joins.txt`, which lists every split word it rejoined.

### What the converter does automatically

- Removes running headers, page numbers, `Figure N.` captions, and lines with no real words (map and image debris).
- Rejoins paragraphs that cross a page break.
- Fixes common OCR damage:
  - **Words run together:** `ofthe` → `of the`, `ofMontana` → `of Montana`.
  - **Letters read as digits:** `I994` → `1994`.
  - **Space before punctuation:** `word ,` → `word,`.
  - **Ordinals:** `21 s1` → `21st`.
- Rejoins words the OCR split in two (`enteri ng` → `entering`). It joins only when the combined word appears elsewhere in the book and at least one fragment doesn't look like a real word.
- Turns footnotes into Markdown footnotes (`[^3]`) and links each one to its reference in the text. If the OCR lost a reference mark, the note is listed under a **Notes** heading at the end of the chapter instead.
- Escapes characters that Markdown or Liquid would misread.
- Writes front matter with `title`, `order` and `part`.

## Step 6: Review and adjust

Open several output files next to the PDF and check them. Fix problems **in the config**, then repeat Step 5. Each run rebuilds the output from scratch, so edits made to the output files are lost.

| Problem | Fix |
|---------|-----|
| A chapter starts or ends in the wrong place, or a paragraph count is far off | Correct that chapter's `page` |
| The first paragraph of a chapter is missing, or part of the title is left in the text | See *Chapter title pages* below |
| Garbage text from a map or photo | Add the page to `skip_pages` |
| Running headers left in the text | Loosen the `running_header` regex |
| The same OCR error repeats (`tbe` for `the`) | Add `{ find: '\btbe\b', replace: 'the' }` to `text_fixes` |
| A wrong join in `joins.txt` (e.g. `per cent` → `percent`) | Add the joined word to `keep_apart` |
| Footnotes run together or are missing | Check `footnote_numbering`. Some notes will always need hand fixing. |

**Chapter title pages.** On a chapter's first page, the script drops the first line (the chapter number) and then any following short lines that don't end in punctuation (the title). A book laid out differently, for example with an epigraph or a byline under the title, may need a hand fix on that page.

## Step 7: Install into the site

1. **Commit first.** That way the current `_essay/` files can be recovered.
2. **Remove the old chapter files** from `_essay/`, but keep `_essay/README.md`.
3. **Copy in the new files:**

   ```bash
   cp ocr-work/my-book/out/*.md _essay/
   ```

4. **Preview the site:**

   ```bash
   bundle exec jekyll serve
   ```

   At http://localhost:4000, check the table of contents, chapter order and a few chapters, including their footnotes.

## Step 8: Finish by hand

Some things the converter can't do well:

- **Tables** come out as runs of numbers or garbage. Rebuild them as Markdown tables or remove them.
- **Figures and maps** are dropped. To add images back, make them collection items and use the essay feature includes, such as `{% include essay/feature/aside.html objectid="..." %}` or `image-gallery.html`.
- **Leftover OCR typos** (`retum`, `comer` for `corner`). A spell-check pass catches most of them.
- **Unlinked footnotes** under the **Notes** heading. Find the spot in the text, add `[^N]`, and turn the note into `[^N]: …`.
- **Quoted sections** that the OCR merged into the end of a previous paragraph. Split them out by hand.

**Once you start editing files in `_essay/`, don't re-run the converter into that folder.** It would overwrite your edits. Treat the script as a one-time import, and make all later changes directly in the essay files.

## Quick reference

```bash
mkdir -p ocr-work/my-book
pdftotext -enc UTF-8          ocr-work/my-book/book.pdf ocr-work/my-book/book.txt
pdftotext -enc UTF-8 -layout  ocr-work/my-book/book.pdf ocr-work/my-book/book-layout.txt
ruby scripts/ocr-pages.rb ocr-work/my-book/book.txt > ocr-work/my-book/pages.txt
cp scripts/ocr-configs/template.yml scripts/ocr-configs/my-book.yml   # then fill it in
ruby scripts/convert-ocr.rb scripts/ocr-configs/my-book.yml \
  ocr-work/my-book/book.txt ocr-work/my-book/book-layout.txt ocr-work/my-book/out
# review, adjust the config, re-run; then copy out/*.md into _essay/ and edit by hand
```
