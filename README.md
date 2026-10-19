> [!NOTE]
> **Hackathon submission snapshot**
>
> This repository preserves the version submitted to **Reinvent the Wheel, 16–18 October 2026**, hosted by [Science Works](https://science.works) and [Inherent Laboratories](https://inherentlabs.ai/).
>
> **For the latest version, issues and contributions, visit the original project: [achu730/plotwise](https://github.com/achu730/plotwise).**
>
> Submitted version: [`ec3282c`](https://github.com/science-works/rtw-2026-plotwise/tree/ec3282cff0126c5ab66888c7c46f948e24885b2b). This notice was added after submission. Nothing else in the repository has changed.

---


# Plotwise

Turn an analysed dataset into a figure that follows your journal's or collaboration's style guide,
fix it by chatting instead of rerunning the analysis, and prove every plotted point still matches the data.

## Run it

**As a webpage, nothing to install:** host it once (free on Hugging Face Spaces, all in the browser) and share the
link. Step by step in [HOSTING.md](HOSTING.md).

**On your own computer:**

```bash
bash run.sh         # macOS / Linux  (Windows: double-click run.bat)
# then open http://localhost:5050
```

`run.sh` makes a virtual environment and installs `requirements.txt` the first time.
Without it: `pip install -r requirements.txt && python app.py`.

**Which AI writes the code** (shown top right in the app, picked automatically):

| You have | Plotwise uses |
|---|---|
| `ANTHROPIC_API_KEY` set | Claude API (`claude-opus-5-5`; override with `PLOTWISE_MODEL`) |
| Claude Code installed and logged in (`claude` on your PATH, works with a Pro/Max login) | Claude Code in print mode |
| neither, or no internet | Offline rules: understands "Y vs X", log axes, legend position, units, labels, ranges, error bars, fits, "Preliminary" |

Force one with `PLOTWISE_BACKEND=api|cli|offline`. If Claude fails partway through, Plotwise falls back to the offline engine and logs that it did.

## The seven steps

1. **Style guide**: upload the journal's or collaboration's style guide (DOCX, PDF, TeX, TXT, MD). The card shows
   "<guide name> uploaded." and a summary of its figure rules. Claude turns the guide into a style spec: fonts,
   width in mm, tick style, unit format, experiment label, colours. Until a guide is uploaded a plain default is used.
   You can also **type the style you want** ("Arial 8 pt, 85 mm wide, colour-blind palette, ticks inside on all
   sides"); what Plotwise understood is listed, and "Figure style now" shows the result with colour swatches.
2. **Previous plotting code or figures** (optional): drop scripts you plotted with before (Python, notebooks,
   ROOT macros, R, MATLAB) or figures you made (PNG, JPG, SVG, PDF). Plotwise copies their colours, fonts, sizes,
   markers and tick style. The colour palette of a figure is measured from its pixels (or read from the SVG/PDF)
   and shown as swatches. Layers apply in order: guide, then past plots, then what you typed, so the most
   specific wins. Each can be switched off or removed, and the current figure is redrawn straight away.
3. **Data**: drop a `.csv` or `.xlsx` (all sheets are read). Plotwise records its SHA-256, never edits it, and
   lists cells that can't be plotted. Tables can sit anywhere: headers on any row (or two rows: names, then units),
   several tables stacked or side by side, titles and notes around them, and a separate table of uncertainties that
   is matched back to its values. Labelled single values ("Frame rate (fps) | 50") anywhere on the sheet are listed
   as constants, each with its cell and any matching uncertainty, and the AI uses them by name through logged calculations.
   CSV tables downloaded from [HEPData](https://www.hepdata.net) work as they are (the `#:` header lines become the table title,
   and asymmetric `stat +` / `stat -` errors are checked side by side).
4. **Plot from a sentence**: "y axis (y displacement) vs x axis (time)". Claude writes matplotlib code, the app
   runs it in a separate process, and the style is applied.
5. **Edit by chatting** ("log y axis", "legend bottom left", "use cm", "add a quadratic fit") **or by hand** with
   **Format axes**, like Excel's Format Axis: axis titles, minimum and maximum, tick spacing, log scale, legend
   position, gridlines and a trendline. Hand changes need no AI and are checked like any other version.
   **Click any point** in the figure to see its x, y, error bar and the row of your file it came from.
6. **Check**: automatic checks on every version, plus a human spot check of the 2nd point and random ones.
   LaTeX in labels is shown as readable text in warnings ("$\pi^{-}$" reads "π⁻").
7. **Export**: PDF for LaTeX, PNG, SVG, the point inspector (HTML), the AI log, or a zip with everything
   including `reproduce.py`, which rebuilds the figure from the data without Plotwise. The **Code** tab shows the
   plotting code with syntax highlighting. **Save to your GitHub** (optional) commits the code, figure, checks,
   AI log and a `manifest.json` of SHA-256 hashes to your own repository in one commit; the PDF and PNG carry the
   session, version and data hash in their metadata, so a figure found later can be traced to its code and data.
   Use a fine-grained token with "Contents: Read and write" on that one repository; Plotwise never stores it.

## What the AI may do to the data

Three things, and each one is shown as a warning in Checks and recorded in the AI log:

1. **Scale by a factor**, declared in the code: a unit change with `pw.convert` (MeV/c → GeV/c, m → cm) or a
   constant from the file with `pw.transform` (pixels × metres per pixel). No offsets.
2. **Log axes** (`set_yscale("log")`); the values themselves stay as they are.
3. **Fits** drawn over the data with `pw.fit`: linear, polynomial (degree 2-6), exponential, logarithmic, power
   law and Gaussian, weighted by the error bars when there are any, optionally over an x range. The data points
   stay as they are. The checker confirms the fit used points from your file, re-does the fit itself and checks
   the line drawn is that model; the equation with uncertainties and R² or χ²/ndf are logged.

Everything else is refused: smoothing, other model curves, normalising, offsets, interpolating, filling gaps,
resampling and dropping rows. Code that calls such functions (including `np.polyfit` or `curve_fit` outside
`pw.fit`) is blocked before it runs, and any other change is caught by the checker: a factor the code didn't
declare, a curve that isn't in the file, or a point that doesn't match its row is a FAIL.

## How the integrity checks work

Plotwise does not trust the AI's code. After each run it reads the numbers back **out of the rendered
figure** (every marker, error bar, bar and curve) and matches each one to a row of your file:

- **Every point traced**: each plotted (x, y) must equal a source row, allowing only a clean unit factor
  (×100 for m → cm). A changed, invented or smoothed value is a FAIL with the row and the % difference.
- **Nothing dropped**: every row of the series (grouping columns such as "Trial" are detected) must be plotted.
  Empty or non-numeric cells are listed with their spreadsheet row so you can fix the source.
- **Error bars** must equal the uncertainty in the same row, with the same factor as the values (or a calculation declared in the code); a missing error bar is a warning.
- **Units**: if the axis says cm, the values must be the source column × 100. Label and data disagreeing is a FAIL.
- **Typed-in numbers**: a literal list of numbers in the code is flagged.
- **Style rules**: width in mm, no in-figure title, unit format, legend, font actually used, experiment label,
  missing glyphs.
- **Unsafe code** (file, network, shell access) is blocked before it runs. On a hosted copy the plot code also
  runs inside a guard that refuses to open files outside its own folder or start programs, and it never sees
  API keys or tokens.

**Saved sessions** (top right) lists every session that has a guide, data or a figure, newest first. Name the
current one there, and click any entry to reopen it with all its versions, checks and AI log. Sessions are
folders under `sessions/`, so they survive restarts and unzipping a new version over the old folder. On a hosted
copy each browser only sees the sessions it made.

The **Inject AI mistake** button secretly nudges one value by 5 % and drops a row inside the code, to show
the checker catches it. The **AI log** (terminal tab, and `ai_log.md` in the export) records every step by the
human, the AI and the app: prompts, Claude's replies, code diffs, check results and spot-check answers.

## Layout

```
app.py                  Flask server and API
plotwise/core.py        session, style layers, versions, format panel, AI log, export, GitHub deposit
plotwise/llm.py         Claude backends and prompts
plotwise/harness.py     runs generated code in a subprocess, reads points back, style checks, sandbox
plotwise/integrity.py   figure-vs-data checks, including fits
plotwise/pwstyle.py     style helpers (`pw`) and pw.fit given to the generated code; copied into reproduce.py
plotwise/styleinputs.py style from typed preferences, old plotting code and old figures (palettes)
plotwise/texttools.py   LaTeX to readable text for warnings (static/texttools.js is the same for the page)
plotwise/github.py      one-commit upload to the scientist's own repository
plotwise/offline.py     rule-based fallback engine
plotwise/inspector.py   standalone point-inspector HTML
plotwise/tables.py      reads messy spreadsheets (several tables, uncertainty tables, side notes)
plotwise/base_style.json  plain default style used before a guide is uploaded
Dockerfile, render.yaml hosting (see HOSTING.md)
```
