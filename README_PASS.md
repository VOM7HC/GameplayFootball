# Optional match reports (PASS format)

This feature adapts [Tijmen Krijnen's seven 2022 commits](https://github.com/tijmenkrijnen/GameplayFootball/compare/68159a2f0f96eec8ebba26ab7820130f36b922a7...0506df2b36ecb1fd1ed233ae4dc0a1495ad8320e).
His original author identity and source-commit references are retained in this
branch's history. The reporting implementation has been modified to preserve
current game behavior and handle output errors safely.

## Enable reporting

Reporting is off by default. Add a line to the `football.config` used by the game:

```text
"match_report_directory" "match-reports"
```

The directory is relative to the game's working directory; an absolute path also
works. Remove the setting, or leave its value empty, to disable collection and
export. Each match snapshots this setting when it is created.

Leaving a match writes `match-reports/match-<start-ms>-<unique-id>/report.json`.
Each match reserves its own directory, so games with the same start timestamp
cannot overwrite each other's reports. JSON is written to a temporary file and
renamed after closing successfully. A write error is logged as a warning and
does not prevent normal match cleanup. Reporting is synchronous at match exit,
so a slow output filesystem may delay that exit.

## Report contents

The three top-level arrays remain `MatchActions`, `MatchInfo`, and `MatchLineup`.
The inherited Dutch event labels, PASS codes, string-valued numbers/booleans, and
`/Date(milliseconds)/` Unix-epoch dates are retained. Empty lists are JSON arrays
(`[]`), not empty strings. This is a PASS-format adaptation; compatibility with an
external PASS installation has not been tested.

- Actions include kickoffs, goal kicks, free kicks, corners, penalties, throw-ins,
  goals/own goals, offsides, fouls, and cards. Events snapshot match time, phase,
  and score when recorded. This is not a complete pass/shot or player-position log.
- `n_HomeOrAway` identifies the actor's team (0 home, 1 away, -1 unknown).
  `n_BeneficiaryHomeOrAway` separately identifies the team awarded a goal or
  restart, when relevant; -1 means not specified. An own-goal scorer remains
  associated with their own team. Unknown scorers retain empty person fields
  and the known actor team rather than disappearing from the report.
- Cards are recorded when awarded. A second yellow produces the yellow event and
  a separate second-yellow dismissal. A direct red produces only a direct-red
  event. Existing card delays and send-off gameplay rules are unchanged.
- `MatchLineup` includes the roster, with starting roles captured from the actual
  formation at match creation. `b_Starter` distinguishes starters from the bench;
  bench/unknown roles have code 0. Sent-off starters remain in the roster and have
  `b_RedCard` set. Export-only role labels do not change the formation UI.
- `MatchInfo.c_Status` is `completed` after the engine's game-over condition,
  otherwise `abandoned`. Abandoning a game still produces a report. Halftime goals
  are captured once on transition to the second half. `b_HalfTimeKnown` is false
  until that transition; unknown halftime scores are not fabricated as 0-0.
- Start/end timestamps are 64-bit milliseconds since the Unix epoch. The legacy
  `Local` fields contain the same epoch instant, not a timezone-shifted timestamp.
  Match start here means match object creation, including loading.
- Valid UTF-8 player/team names are preserved and JSON-escaped. Invalid source
  bytes are replaced with U+FFFD in the report. The bundled database is unchanged;
  no player or team is renamed to work around encoding problems.

See [example_output.json](example_output.json) for a synthetic regression example.
Regenerate it with `match_report_tests --example example_output.json` after building.

## Validation

The serializer and file writer are independent of the renderer and window system.
With `BUILD_TESTING=ON` (the default), build and run:

```text
cmake --build <build-directory> --config Release --parallel
ctest --test-dir <build-directory> -C Release --output-on-failure
```

Tests cover action codes and attribution, direct-red versus second-yellow records,
score/date fields, abandoned/empty reports, roster roles, UTF-8 and JSON escaping,
repeated-match filenames, and invalid output paths. The Windows and macOS CI
workflows run these tests. Builds and serializer tests do not replace checking
real match events and exit behavior on each platform.
