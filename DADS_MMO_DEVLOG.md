# Dad's MMO Development Log

## 2026-08-20 — Qwen Reasoning Fix / Chat Delivery Investigation

### Completed

Investigated how mod-playerbots-characters accesses the local AI.

Confirmed architecture:

- Provider: Ollama
- Endpoint: http://host.docker.internal:11434/api/chat
- Model: qwen3:30b
- Ollama runs independently on Windows and automatically loads the model when PBC sends a request.
- The model unloads after inactivity but Ollama itself remains available.

### Qwen Reasoning Leak

Despite sending:

    "think": false

Qwen3:30b returned internal reasoning inside `message.content`, ending with:

    </think>

This reasoning was being stored in PBC history along with the actual character response.

Modified:

    src/llm/pbc_llm.cpp

The Ollama response handler now strips everything through the final `</think>` marker and trims leading whitespace before returning the response.

Example before:

    [large reasoning block] </think> Fine

Example after:

    Fine

Rebuilt the ac-worldserver Docker image from source.

Verified the fix in `mod_pbc_history`.

New Therinaya responses are clean:

    Hi

instead of containing Qwen reasoning.

### Git

Development branch:

    dads-mmo

Fix committed as:

    619b38cf2977a28795ab208dbfaf878b87ded621
    fix(llm): strip leaked Qwen reasoning from bot responses

Git author configured as:

    Thomas Rodriguez <162509959+goos3d@users.noreply.github.com>

### Current Problem

Therinaya generates replies successfully, but they are not appearing in the WoW client chat.

Confirmed working:

    WoW player message
        -> PBC receives event
        -> PBC calls Ollama
        -> qwen3:30b generates response
        -> reasoning-strip patch cleans response
        -> PendingReply created
        -> history written to database
        -> clean Therinaya response exists in mod_pbc_history

Example:

    Dadbod: hi
    Therinaya: Hi

However, Therinaya's `Hi` is not visible in WoW chat.

### Investigation So Far

`ProcessNormal()` successfully creates:

    PendingReply reply;
    reply.messageText = res.text;

and executes:

    replies.push_back(std::move(reply));

When narrator events are enabled, responses pass through:

    ParseNarratorSpans(res.text)
        -> PushReplySegments(...)

`PushReplySegments()` correctly creates a normal `PBC_PendingAction` for plain text such as `Hi` and pushes it into:

    g_PBC_PendingActions

Therefore the current suspected failure point is AFTER the pending action is queued.

Current pipeline:

    Qwen response             OK
        |
    reasoning cleanup         OK
        |
    PendingReply              OK
        |
    history/database          OK
        |
    ParseNarratorSpans        OK
        |
    PushReplySegments         OK
        |
    g_PBC_PendingActions      OK
        |
    pending-action consumer   INVESTIGATE NEXT
        |
    WoW chat packet/client    NOT DISPLAYING

### Resume Here

Find where `g_PBC_PendingActions` is consumed:

    grep -Rni "g_PBC_PendingActions" \
    modules/mod-playerbots-characters/src

Also:

    grep -RniE "PendingActions.*(front|pop|empty)|PendingAction.*action" \
    modules/mod-playerbots-characters/src

Likely location:

    src/core/pbc_world.cpp

Inspect approximately:

    sed -n '300,410p' modules/mod-playerbots-characters/src/core/pbc_world.cpp

Goal:

Trace the pending action from `g_PBC_PendingActions` through AzerothCore's chat system to the WoW client.

Do NOT modify the Qwen cleanup patch while investigating this. It has already been verified working.

### Other Notes

Observed warning:

    [PBC] [WARNING] Unknown template token '{rt1}' — replaced with empty string

This has not yet been shown to be related to the missing chat display.

`conf/playerbots_characters.conf` remains intentionally untracked and should be reviewed for local/private configuration before ever being committed.
