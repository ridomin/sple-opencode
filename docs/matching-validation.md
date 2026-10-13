# YouTube Music song matching validation

## Run summary

Validation performed on 2026-09-23 against the saved **FlamenRock** snapshot
(`0FRr10mglUR3E0Pq8TqlxL`) using the same `runMatchPhase`, Music adapter and
matcher used by the CLI. Refresh was enabled, with the default threshold 0.75,
auto ambiguity policy and explicit `en-US` catalog locale. The run used the
downloaded snapshot directly, so it did not refresh Spotify data.

The existing status contained two automatic unmatched rows and three added
rows. Only the two non-terminal automatic rows were searched. The match status
and v2 decision cache were updated locally; no playlist-creation or track-add
operation was invoked. The three added entries were compared before and after
and were identical, including the confirmed Los Chichos selection.

The pre-run status backup is at:

```text
%LOCALAPPDATA%\Temp\opencode\flamenrock-before-song-matching-1790192396018.json
```

The updated review artifact is at:

```text
%APPDATA%\sp2ytm\matches\0FRr10mglUR3E0Pq8TqlxL.json
```

## Retrieval, ranking and listening are separate checks

| Track | Expected Music song | Retrieval and ranking in this run | Recording review |
| --- | --- | --- | --- |
| Las Leyes De La Frontera | [doHMIFM90Ms](https://music.youtube.com/watch?v=doHMIFM90Ms) | First query retrieved the expected song; selected at rank 1, score **0.930**, no eligibility rejection | User explicitly supplied this preferred song before implementation |
| La lista de la compra | [7_GAEckFi5Y](https://music.youtube.com/watch?v=7_GAEckFi5Y) | First query did not retrieve the expected ID. Second, album-specific query retrieved it; selected at rank 1, score **1.000**, no eligibility rejection | User previously confirmed the same performance and edit by listening |
| Quiero Ser Libre - Single Version | [SM2zDCCiREM](https://music.youtube.com/watch?v=SM2zDCCiREM) | Already added, so not searched or ranked again. Stored selection was preserved exactly | User previously confirmed this remaster preserves the same performance and edit |

Both targeted unmatched rows became `matched`. This is **2/2 expected songs
retrieved and selected in this small live sample**, not a general accuracy
estimate. There was no new audio listening by the agent during implementation.
The Los Chichos row is a preserved completed result, not an additional live
retrieval pass; a separate offline test verifies manual-choice preservation.

### Queries and observed metadata

1. `Derby Motoreta’s Burrito Kachimba - las leyes de la frontera`
   - Expected ID present among twenty song results.
   - Catalog duration: 216 seconds; Spotify: 215.852 seconds (difference 0.148s).
   - Strong, unambiguous result stopped further discovery.
2. `La Cabra Mecanica - la lista de la compra`
   - Expected ID absent from the first twenty results.
   - A “Versión 2009” alternative scored 0.850, below the strong-match early-stop
     threshold. This allowed the album query to run.
3. `La Cabra Mecanica - la lista de la compra Vestidos De Domingo`
   - Expected ID retrieved with the correct album and featured-artist title.
   - Catalog duration: 222 seconds; Spotify: 221.373 seconds (difference 0.627s).
   - Earlier watch-page metadata reported 221 seconds for this ID; this run
     records the Music catalog's actual value, still within the three-second
     tolerance.
   - The catalog artist list named La Cabra Mecanica; title feature extraction
     supplied the María Jiménez credit for scoring.

No provider failures or region blocks were observed for these two searches.
Song catalog presence does not establish playback availability in every region.

## Automated checks

- `npm run typecheck`: passed.
- `npm test`: **148 tests passed** across 19 files.
- `npm run build`: passed.
- Built `match --help`: lists Music song matching, `--refresh`, `--threshold`
  and `--on-ambiguous`.
- `youtubei.js` import verified under Node **20.20.2**, in addition to the
  development runtime.
- Offline fixtures cover song/video filtering, the two expected automatic
  selections, remaster acceptance, uncertain edits, duration and artist gates,
  near ties, cache/policy invalidation, cancellation, provider errors,
  manual/terminal preservation and the CLI's no-mutation matching workflow.

## Repeating manual validation

### Follow-up: Mola spelling-suggestion response

The first pending Mola track, Jane's Addiction's “So What! - Jane's House;
Deep Red's Dancefloor Dub”, reproduced `unrecognized result shelf`. Its valid
Songs response contained an `ItemSection` with `DidYouMean` alongside a
`MusicShelf`. The adapter now accepts known `DidYouMean` and
`ShowingResultsFor` search hints as metadata, while rejecting unknown/malformed
containers and still selecting only song results.

All three queries for that track were re-run live after the fix and each
returned twenty songs without the parser error. This check performed search
only; it did not migrate Mola. The 26 adapter/matcher tests, typecheck and build
passed after the fix.

### Procedure

1. Inspect the snapshot and status file first. Note manual and added/skipped rows;
   these are deliberately excluded from automatic refresh.
2. Run `node dist/cli/index.js match --refresh "FlamenRock"`.
3. Use the printed status path to inspect `evidence.queries`, candidates,
   component scores, duration differences, version notes and rejection reasons.
4. For each new proposed recording, compare opening, vocals/chorus, arrangement,
   speed and ending/fade with Spotify. Record **same recording / acceptable
   remaster / wrong version / unsure** separately from search/ranking results.
5. For a confirmed manual choice, preserve the existing row/input identity and
   set `status: "matched"`, `manual: true` and the chosen `videoId`; update or
   remove obsolete selected-candidate evidence as described in README.
6. Report absent IDs, provider failures or region limitations explicitly.
   A high heuristic score alone is not a listening confirmation.

### Corroborated leading-article artist alias

The automatic eligibility gate accepts a primary-artist credit that differs
from the Spotify primary artist only by a leading article from `["the", "a",
"an"]` (case- and diacritic-insensitive, English only) **only when** the
candidate's normalized core title is identical, its album is near-identical
(album similarity at least 0.9) and its known duration is within tolerance.
Such an acceptance records a `Music: leading-article alias` note in
`versionNotes` while keeping the raw `primaryArtistScore` value in evidence.
An article-only difference **without** that album/title/duration corroboration
is not automatically eligible and remains a reviewable limitation of the gate.

The Toy Dolls rows on `We're Mad! The Anthology` are the corroborated example
used to validate this rule end-to-end.


### Corroborated leading-article artist alias

A primary-artist credit that differs from the Spotify primary artist only by a
leading English article (`the`, `a`, `an`; case- and diacritic-insensitive) is
accepted by the automatic eligibility gate **only when** the candidate's core
title is identical (`titleScore === 1`), its album is near-identical
(`albumScore >= 0.9`), it has a known duration within tolerance, and it is not
otherwise duration-rejected. Such an acceptance is recorded with a
`Music: leading-article alias` version note while the raw `primaryArtistScore`
stays in evidence. An **article-only** difference without that album/title/
duration corroboration is not automatically eligible and remains a reviewable
limitation of the gate.

The Toy Dolls rows on `We're Mad! The Anthology` are the corroborated example
used to validate this rule end-to-end.
