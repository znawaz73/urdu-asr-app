# Use `auto` language detection and display raw model output

The backend's `LANG_TO_ID` map has no `ur` (Urdu) entry — sending `language: "ur"` returns
an error. Rather than falling back to `hi` (Hindi/Devanagari) or adding a client-side
transliteration layer, we use `language: "auto"` and display whatever script the model
produces verbatim. This keeps the client simple and honest: the app evaluates what the
model actually does with Urdu speech, which is the stated purpose.

## Considered options

- **`ur` → `hi` fallback**: attempt `ur`, catch the error, retry with `hi`. Adds a failed
  connection on every session start with no benefit since `ur` will never succeed.
- **Client-side transliteration (Devanagari → Latin)**: would require a lookup table or
  external library, contradicts the zero-dependency constraint, and the output would be
  approximate anyway.
- **`auto` + raw output** *(chosen)*: no fallback logic, no post-processing, no extra
  dependencies. The label shown in the UI (`language_name` from the `ready` event) tells
  the user what the model detected.

## Consequences

The app no longer enforces RTL layout or Nastaliq/Naskh fonts. If the model produces
Arabic-script output for some audio it detects as Arabic/Urdu, that text will render
left-to-right in a standard system font — readable but not typographically correct.
Acceptable for an evaluation tool; revisit if the app is ever used for display purposes.
