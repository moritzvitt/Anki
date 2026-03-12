# Pseudocode Overview

## Progress Utility

FUNCTION print_progress(prefix, done, total)
PRINT prefix + “: “ + done + “/” + total (overwrite same line)
IF done >= total
PRINT newline
END

---

## Batched Note Updates

FUNCTION run_batched_note_updates(client, updates, label, batch_size)

written = 0
total = length(updates)

IF total == 0
    RETURN

FOR each batch IN chunk(updates, batch_size)

    actions = []
    FOR each update IN batch
        actions.ADD(updateNoteFields action with update)
    END

    client.invoke_multi(actions)

    written += size(batch)
    print_progress(label, written, total)

END

END

---

## Chunked Tag Actions

FUNCTION run_chunked_note_tag_action(client, note_ids, action, tags, label, batch_size)

done = 0
total = length(note_ids)

IF total == 0
    RETURN

FOR each batch IN chunk(note_ids, batch_size)

    client.invoke(action, notes=batch, tags=tags)

    done += size(batch)
    print_progress(label, done, total)

END

END

---

## Tag Sanitization

FUNCTION sanitize_tag_component(value)

normalized = lowercase(trim(value))

replace whitespace with "_"
replace non-word characters with "_"

IF normalized is empty
    RETURN "unknown_field"

RETURN normalized

END

---

# Cloze Generation

## `format_single_lemma_cloze_notes`

FUNCTION format_single_lemma_cloze_notes(client, notes, dry_run)

success = 0
failed = 0
skipped_multi = 0
skipped_existing = 0

updates = []
success_ids = []
failed_ids = []

FOR each note WITH index i

    cloze = note.fields["Cloze"]
    lemma = note.fields["Lemma"]
    definition = note.fields["Word Definition"]

    IF cloze matches CLOZE_PATTERN_RE
        skipped_existing += 1
        CONTINUE

    (new_cloze, generated) = make_cloze(cloze, lemma, definition)

    IF generated IS NULL
        skipped_multi += 1
        CONTINUE

    IF NOT dry_run
        updates.ADD(update Cloze field)

    IF generated == TRUE
        success += 1
        success_ids.ADD(note id)
    ELSE
        failed += 1
        failed_ids.ADD(note id)
    END

END

IF NOT dry_run
    run_batched_note_updates(...)
    add/remove success/fail tags
END

RETURN statistics

END

---

# Duplicate Detection

## `suspend_new_duplicates_by_lemma`

FUNCTION suspend_new_duplicates_by_lemma(client, notes, dry_run)

map note_by_id
map note_cards

FOR each note
    store its card ids
END

IF card info missing
    fetch from client
END

all_card_ids = union of all card ids

IF empty
    RETURN

new_cards_by_note = {}

FOR each card
    IF card.queue == NEW
        group card under its note id
END

IF no new cards
    RETURN

new_notes = notes belonging to those ids
grouped = group_notes_by_lemma(new_notes)

duplicate_note_ids = notes where group size > 1

### Choosing the Best Card

FUNCTION best_new_card_key(cards)

choose card with smallest due
tie-break with smallest card id

RETURN (due, id)

END

### Suspension Logic

cards_to_suspend = set
notes_to_tag = set

FOR each lemma group

keep_nid = note whose card has best_new_card_key

FOR each other note in group
    notes_to_tag.ADD(note)
    cards_to_suspend.ADD(all its cards)
END

END

### Apply Changes

IF NOT dry_run

suspend cards_to_suspend

tag notes_to_tag

END

RETURN statistics

---

# Heisig Linking

## `link_heisig_with_media`

FUNCTION link_heisig_with_media(client, media_notes, heisig_notes, dry_run)

media_kanji = map note_id → set of kanji
all_kanji = set

FOR each media note
    chars = extract_kanji_chars(media_field)
    store in media_kanji
    add to all_kanji
END

updates = []
written = 0

FOR each heisig note

    chars = extract_kanji_chars(heisig_field)
    shared = intersection(chars, all_kanji)

    IF none
        CONTINUE

    linked_media = all media notes sharing those kanji

    IF empty
        CONTINUE

    links = HTML links to those notes

    IF NOT dry_run
        updates.ADD(update link field)

    written += 1

END

IF NOT dry_run
    run_batched_note_updates(client, updates)
END

RETURN statistics

END

---

# Furigana Generation

## `add_furigana_to_field`

FUNCTION add_furigana_to_field(client, notes, dry_run)

updated = 0

skipped_missing_source = 0
skipped_missing_target = 0
skipped_missing_reading = 0
skipped_existing_furigana = 0

FOR each note

    source_name = resolve source field

    IF missing
        skipped_missing_source += 1
        CONTINUE

    target_name = resolve target field

    IF missing
        skipped_missing_target += 1
        CONTINUE

    IF target already contains square furigana
        skipped_existing_furigana += 1
        CONTINUE

    IF source is inline text
        furigana = build_furigana_text(source)
    ELSE
        reading_name = resolve reading field

        IF missing
            skipped_missing_reading += 1
            CONTINUE

        furigana = build_furigana(source, reading)
    END

    IF furigana empty OR unchanged
        CONTINUE

    IF NOT dry_run
        schedule update

    updated += 1

END

IF NOT dry_run
    run_batched_note_updates(client, updates)

RETURN statistics

END

---

# Loading Notes

## From Note Query

FUNCTION load_notes(client, query)

note_ids = client.findNotes(query)

RETURN client.get_notes_info(note_ids)

END

---

## From Card Query

FUNCTION load_notes_from_cards(client, query)

card_ids = client.findCards(query)

cards = client.get_cards_info(card_ids)

note_ids = unique note ids from cards

RETURN client.get_notes_info(note_ids)

END

---

# Tag Notes Containing Furigana

## `tag_notes_containing_square_furigana`

FUNCTION tag_notes_containing_square_furigana(client, notes, dry_run)

tagged_ids = []
skipped_missing_target = 0

FOR each note

    target_field = resolve target field

    IF missing
        skipped_missing_target += 1
        CONTINUE

    value = field value

    IF contains square furigana
        tagged_ids.ADD(note id)

END

IF NOT dry_run AND tagged_ids NOT empty

    tag = "meta::contains_furigana_in_<field>"

    add tag to notes

END

RETURN statistics

END

