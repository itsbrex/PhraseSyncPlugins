# Phase 5: Code Review Results

## Review Competition Results

| Metric | Agent Alpha | Agent Beta |
|--------|-------------|------------|
| **Critical** | 7 | 5 |
| **Major** | 10 | 10 |
| **Minor** | 12 | 9 |
| **Suggestions** | 5 | 5 |
| **Total** | **34** | **29** |

**Winner: Agent Alpha** (34 findings vs 29), with several unique catches including the ControllerMotion division-by-zero on `phraseRemaining`, the NoteFilter off-by-one variation numbering, and the deprecated JUCE parameter ID constructors. Agent Beta brought unique value with the CC range overflow, stuck-notes-on-variation-change scenario, transport-jump CC overshoot, and the fudge factor analysis.

---

## Combined Findings (Deduplicated)

### CRITICAL

- [x] **C1. Dangling pointer in `findNoteOnEvent()`** (LineToggler/Source/PluginProcessor.cpp:212-231)
  Returns `&metadata` where `metadata` is a stack-local copy. Caller dereferences a dangling pointer. UB -- crash or corrupt timing. *Both agents found this.*

- [x] **C2. `std::cout` on the audio thread** (NoteFilter/Source/PluginProcessor.cpp:307-309)
  `std::cout` with `std::endl` flush + `getDescription()` heap allocations on every note event. Guaranteed audio glitches. *Both agents found this.*

- [x] **C3. Division by zero when `tempoBpm` is 0** (ChannelFilter:172, ControllerMotion:253, NoteFilter:195)
  `60.0 / tempoBpm` -- hosts may report bpm=0 when transport is stopped. Produces Inf/NaN that corrupts all phrase calculations. *Both agents found this.*

- [x] **C4. Deprecated JUCE API `getCurrentPosition()`** (ChannelFilter:232, ControllerMotion:291, NoteFilter:272)
  Removed in JUCE 7+. Won't compile on modern JUCE. *Both agents found this.*

- [x] **C5. Negative MIDI note numbers / stuck notes** (NoteFilter/Source/PluginProcessor.cpp:246)
  `originalNote - variationStartNote` can go negative. JUCE masks to 0-127, producing garbage. Note-offs for notes playing during a variation change will mismatch their note-ons, causing stuck notes. *Both agents found this; Beta added the stuck-notes insight.*

- [ ] **C6. Division by zero when `phraseRemaining` is 0** (ControllerMotion/Source/PluginProcessor.cpp:319)
  `blockPhraseTime / phraseRemaining` when `currentPhrasePosition` is exactly 1.0. Produces Inf, corrupts CC output. *Alpha unique find.*

- [ ] **C7. NoteFilter off-by-one: variation 1 maps to startNote=12, not 0** (NoteFilter/Source/PluginProcessor.cpp:241)
  `variationStartNote = variation * variationHeight`. Variation 1 with height 12 means start=12. Notes 0-11 are unreachable. Should be `(variation - 1) * variationHeight`. *Alpha unique find.*

### MAJOR

- [ ] **M1. Unsafe C-style casts of parameter pointers** (All plugins)
  C-style casts from `RangedAudioParameter*` to `AudioParameterInt*` etc. No runtime check. *Both agents found this.*

- [ ] **M2. Parameter lookup by string on every processBlock()** (ControllerMotion:361-382)
  `parameters.getParameter("outPhrasePosCCNumber")` etc. called on audio thread every buffer. Should be cached. *Both agents found this.*

- [ ] **M3. ControllerMotion writes to input buffer, not outputMidiBuffer** (ControllerMotion:336-343)
  `outputMidiBuffer` is declared but never used. CCs added to input buffer. Inconsistent with other plugins. Potential iterator invalidation. *Both agents found this.*

- [ ] **M4. `createEditor()` returns `0` instead of `nullptr`** (All plugins)
  *Both agents found this.*

- [ ] **M5. Infinite loop risk in LineToggler processBlock()** (LineToggler:248-304)
  `controlNoteSearchStart` increments by 1 but is used as a sample position threshold. Alpha identified the specific infinite loop mechanism; Beta identified the `samplePosition == 0` edge case causing `processEndTime = -1`. *Both agents found this from different angles.*

- [ ] **M6. LineToggler drops all non-line MIDI events** (LineToggler:233-308)
  CCs, pitch bend, aftertouch, SysEx, and out-of-range notes are silently eaten. TODO comment acknowledges this. *Both agents found this.*

- [ ] **M7. LineToggler O(slots * events) complexity** (LineToggler:241-305)
  Outer loop over slots, inner loop over all events. A single-pass approach would be more efficient. *Both agents found this.*

- [ ] **M8. Deprecated JUCE parameter ID constructors** (All plugins)
  String-based parameter constructors deprecated in JUCE 7. Need `ParameterID` objects. *Alpha unique find.*

- [ ] **M9. State restore fragility** (All plugins)
  `replaceState()` relies on JUCE not recreating parameter objects. Cached raw pointers could dangle if internals change. *Beta unique find.*

- [ ] **M10. ControllerMotion CC overshoot on first processBlock / transport jump** (ControllerMotion:307)
  `blockTimeSamples = playheadTimeSamples - lastBufferTimestamp` can be enormous on first call, causing huge `blockPhraseTime` and CC value overshoot. *Beta unique find.*

- [ ] **M11. ControllerMotion firstCCNumber + 3 can exceed CC 127** (ControllerMotion:317)
  Parameter range 1-127, but with 4 CCs, values 125-127 produce CC 128+. Invalid MIDI. *Beta unique find.*

### MINOR

- [ ] **m1. All `.jucer` files share the same project ID `otJdPT`** -- potential host-side conflicts. *Both.*
- [ ] **m2. Duplicate internal element IDs across all `.jucer` files** -- copy-paste artifacts. *Both.*
- [ ] **m3. macOS-only build (XCODE_MAC only)** -- no Windows/Linux exporters. *Both.*
- [ ] **m4. Stale comment "12 lines" but define is 4** (LineToggler/Source/PluginProcessor.h:14). *Both.*
- [ ] **m5. `shouldPlayMidiMessage` takes `MidiMessage` by value** (ChannelFilter/Source/PluginProcessor.h:63) -- unnecessary copy. *Both.*
- [ ] **m6. Duplicate class name `MIDIClipVariationsAudioProcessor`** in ChannelFilter and NoteFilter. *Both.*
- [ ] **m7. `displaySplashScreen="1"` in all `.jucer` files** -- should be disabled for release. *Alpha unique.*
- [ ] **m8. Inconsistent naming: folder vs plugin names** (ChannelFilter -> ClipVariations-Channel). *Alpha unique.*
- [ ] **m9. Double-to-int narrowing in `addEvent()` timestamp** (ChannelFilter:256, NoteFilter:298). *Alpha unique.*
- [ ] **m10. No `#include <sstream>` for `std::ostringstream`** -- relies on transitive include. *Alpha unique.*
- [ ] **m11. Fudge factor analysis** -- the +0.001 in `timeRangeStraddlesPhraseChange()` is benign but the reasoning comment is misleading. *Beta unique.*
- [ ] **m12. ChannelFilter passes non-note-on messages from all channels** -- CCs/pitch bend leak through unfiltered. *Beta unique.*
- [ ] **m13. Unnecessary GUI modules loaded** -- `juce_gui_basics`/`juce_gui_extra` included but no editor exists. *Beta unique.*

### SUGGESTIONS

- [ ] **S1. Extract shared phrase/beat logic** into a common base class or utility (triplicated across 3 plugins). *Both.*
- [ ] **S2. Use `GenericAudioProcessorEditor`** for a free basic UI. *Both.*
- [ ] **S3. Use `std::atomic` for cross-thread member variables.** *Alpha.*
- [ ] **S4. Add parameter listeners** instead of polling in processBlock. *Alpha.*
- [ ] **S5. Migrate from Projucer to CMake.** *Both.*
- [ ] **S6. Clamp CC number to 0-127** in ControllerMotion loop. *Beta.*

---

## Priority Fix Order

1. **C1** -- Dangling pointer (crash/UB, trivial to fix: return by value or index)
2. **C2** -- Remove `std::cout` from audio thread
3. **C5 + C7** -- NoteFilter negative notes + off-by-one (causes stuck notes)
4. **C3 + C6** -- Division by zero guards
5. **M5** -- Infinite loop in LineToggler
6. **M11** -- CC range overflow
7. **M6** -- Pass through non-line MIDI events in LineToggler
8. **C4 + M8** -- Migrate to modern JUCE API (compile fix)
9. **M1** -- Replace C-style casts with dynamic_cast
10. **M2** -- Cache parameter pointers
