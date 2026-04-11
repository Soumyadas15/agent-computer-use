# agent-computer-use — Computer use Skill

You have access to `agent-computer-use`, a CLI tool that controls desktop applications. You can click buttons, type text, read screens, scroll, drag files, move windows — all from the terminal.

## How to think

**Think like a human sitting at the computer.** Before you act, ask yourself: what would I see on screen? What would I click? What would I type?

A human:

1. Looks at the screen (snapshot)
2. Finds what they need (identify refs)
3. Does one action (click, type, key)
4. Checks what changed (re-snapshot)

You must do the same. Never skip steps. Never assume the UI didn't change after an action.

## Core loop

```
snapshot → identify → act → verify
```

```bash
agent-computer-use snapshot -a Music -i -c          # what's on screen?
# read the output, find the right @ref
agent-computer-use click @e5                         # do one thing
agent-computer-use snapshot -a Music -i -c          # what changed?
```

**Every action changes the UI.** Your previous refs are now stale. Always re-snapshot.

## Opening apps

Always wait for the app to be ready before doing anything:

```bash
agent-computer-use open Safari --wait
agent-computer-use snapshot -a Safari -i -c
```

Never interact with an app you haven't opened and snapshotted first.

## Finding elements

**Step 1**: Snapshot with `-i -c` (interactive + compact):

```bash
agent-computer-use snapshot -a Calculator -i -c
```

This shows only clickable/typeable elements with refs like `@e1`, `@e5`, `@e12`.

**Step 2**: Read the output. Find the element you need by its name, role, or id.

**Step 3**: Use the ref. Refs are the fastest and most reliable way to target elements.

If elements are missing, increase depth:

```bash
agent-computer-use snapshot -a Safari -i -c -d 8
```

## Clicking

For buttons, links, menu items — use `click`:

```bash
agent-computer-use click @e5                         # single click (AXPress, headless)
agent-computer-use click @e5 --count 2               # double-click (opens files, plays songs)
```

`click` tries AXPress first (background, no focus steal). Only falls back to mouse simulation for double-click or right-click.

For elements with stable IDs (won't change between snapshots):

```bash
agent-computer-use click 'id="play"' -a Music
agent-computer-use click 'id~="track-123"' -a Music  # partial id match
```

## Typing

**With a target element** (preferred — uses AXSetValue, headless):

```bash
agent-computer-use type "hello world" -s @e3
```

**Into the focused field** (keyboard simulation, needs app focus):

```bash
agent-computer-use type "hello world" -a Safari
```

Always prefer `-s @ref` when you have a ref. It's more reliable.

## Key presses

```bash
agent-computer-use key Return -a Calculator
agent-computer-use key cmd+k -a Slack
agent-computer-use key cmd+a -a TextEdit
agent-computer-use key Escape -a Safari
```

## Scrolling

```bash
agent-computer-use scroll down -a Music              # scroll main content area
agent-computer-use scroll down --amount 10 -a Music  # scroll more
agent-computer-use scroll-to @e42                    # scroll element into view (headless)
```

Scroll needs the app to be focused. Use `scroll-to` for headless.

## Reading content

```bash
agent-computer-use text -a Calculator                # all visible text
agent-computer-use get-value @e5                     # one element's value/state
agent-computer-use get-value 'id="title"' -a Music   # by selector
```

Use `get-value` on specific elements instead of `text` on large apps.

## Window management

```bash
agent-computer-use move-window -a Notes --x 100 --y 100
agent-computer-use resize-window -a Notes --width 800 --height 600
agent-computer-use windows -a Finder                 # get window positions and sizes
```

These are instant and headless — use AXSetPosition/AXSetSize.

## Drag and drop

Drag needs the app to be focused and two visible, non-overlapping areas.

**Think like a human**: you need to see both the source and destination.

```bash
# Step 1: Set up windows side by side
agent-computer-use move-window -a Finder --x 0 --y 25
agent-computer-use resize-window -a Finder --width 720 --height 475
# (open a second Finder window for destination)

# Step 2: Snapshot to find the file
agent-computer-use snapshot -a Finder -i -c -d 8

# Step 3: Get the file's position
agent-computer-use get-value @e32                    # check position

# Step 4: Drag to destination
agent-computer-use drag @e32 @e50 -a Finder         # drag by refs
# or by coordinates:
agent-computer-use drag --from-x 300 --from-y 55 --to-x 1000 --to-y 200 -a Finder
```

## Selector syntax

### Refs (always prefer these)

```bash
@e1, @e2, @e3                         # from most recent snapshot
```

### DSL

```bash
'role=button name="Submit"'            # role + exact name
'name="Login"'                         # exact name
'id="AllClear"'                        # exact id (most stable)
'id~="track-123"'                      # id contains (case-insensitive)
'name~="Clear"'                        # name contains (case-insensitive)
'button "Submit"'                      # shorthand: role name
'"Login"'                              # shorthand: just name
'role=button index=2'                  # 3rd match (0-based)
'css=".my-button"'                     # CSS selector (Electron apps only)
```

### Chains

```bash
'id=sidebar >> role=button index=0'    # first button inside sidebar
'name="Form" >> button "Submit"'       # submit button inside form
```

## Electron apps (CDP)

Electron apps (Slack, Cursor, VS Code, Postman, Discord) are automatically detected. agent-computer-use relaunches them with CDP support on first use.

Everything works headless — no window activation, no mouse, no focus steal:

```bash
agent-computer-use snapshot -a Slack -i -c           # full DOM tree via CDP
agent-computer-use click @e5                         # JS element.click()
agent-computer-use key cmd+k -a Slack                # CDP key dispatch
agent-computer-use type "hello" -a Slack             # CDP insertText
agent-computer-use scroll down -a Slack              # JS scrollBy()
agent-computer-use text -a Slack                     # document.body.innerText
```

**Typing in Electron apps**: `insertText` goes to the focused element. If you need to type into a specific input:

```bash
agent-computer-use snapshot -a Slack -i -c           # find the input ref
agent-computer-use click @e18                        # click to focus it
agent-computer-use key cmd+a -a Slack                # select all
agent-computer-use key backspace -a Slack            # clear
agent-computer-use type "your text" -a Slack         # now type
```

## Verification

Never assume an action worked. Always verify:

```bash
# After clicking:
agent-computer-use snapshot -a Safari -i -c          # check what changed

# After typing:
agent-computer-use get-value @e3                     # verify the value

# Inline verification:
agent-computer-use click @e5 --expect 'name="Done"'  # click then wait for "Done"

# Idempotent typing:
agent-computer-use ensure-text @e3 "hello"           # only types if value differs
```

## Waiting

When UI takes time to load:

```bash
agent-computer-use wait-for 'name="Dashboard"'       # poll until element appears
agent-computer-use wait-for 'role=button' --timeout 15
sleep 2                                # simple delay after navigation
```

## Batch operations

Chain multiple commands to avoid per-command startup:

```bash
echo '[["click","@e5"],["key","Return","-a","Music"]]' | agent-computer-use batch --bail
```

## Real-world patterns

### Search and play a song

```bash
agent-computer-use open Music --wait
agent-computer-use snapshot -a Music -i -c
# find search — click it
agent-computer-use click @e1
agent-computer-use snapshot -a Music -i -c
# find search field
agent-computer-use type "Kiss of Life" -s @e31
agent-computer-use key Return -a Music
sleep 3
agent-computer-use snapshot -a Music -i -c -d 8
# find the track, double-click to play
agent-computer-use click 'id~="604771089"' -a Music --count 2
sleep 2
agent-computer-use get-value 'id="title"' -a Music
# → "Kiss of Life"
```

### Open a DM in Slack and send a message

```bash
agent-computer-use key cmd+k -a Slack
sleep 1
agent-computer-use snapshot -a Slack -i -c
# find and click the search input
agent-computer-use click @e18
agent-computer-use key cmd+a -a Slack
agent-computer-use key backspace -a Slack
agent-computer-use type "Vukasin" -a Slack
sleep 1
agent-computer-use key Return -a Slack
sleep 2
agent-computer-use type "hey, check this out" -a Slack
agent-computer-use key Return -a Slack
```

### Check calendar events

```bash
agent-computer-use open Calendar --wait
agent-computer-use snapshot -a Calendar -i -c -d 6
# read the visible dates
agent-computer-use text -a Calendar
# navigate to next month
agent-computer-use click @e3                         # next month button
sleep 1
agent-computer-use text -a Calendar
```

### Fill a web form

```bash
agent-computer-use open Safari --wait
agent-computer-use snapshot -a Safari -i -c
agent-computer-use type "https://example.com/form" -s @e34
agent-computer-use key Return -a Safari
sleep 3
agent-computer-use snapshot -a Safari -i -c -d 8
agent-computer-use type "John Doe" -s @e5
agent-computer-use type "john@example.com" -s @e6
agent-computer-use type "Hello world" -s @e7
agent-computer-use click @e8                         # submit button
agent-computer-use snapshot -a Safari -i -c          # verify submission
```

### Drag a file between Finder windows

```bash
# set up side-by-side windows
agent-computer-use move-window -a Finder --x 0 --y 25
# (ensure two windows open, Downloads left, Desktop right)
agent-computer-use snapshot -a Finder -i -c -d 8
# find the file (look for textfield with val="filename")
agent-computer-use click @e32                        # select the file
agent-computer-use drag --from-x 300 --from-y 55 --to-x 1000 --to-y 200 -a Finder
```

### Browse App Store

```bash
agent-computer-use open "App Store" --wait
agent-computer-use snapshot -a "App Store" -i -c -d 10
agent-computer-use click 'id="AppStore.tabBar.discover"' -a "App Store"
sleep 2
agent-computer-use scroll down --amount 5 -a "App Store"
agent-computer-use snapshot -a "App Store" -i -c -d 10
agent-computer-use text -a "App Store"
```

## Rules

1. **Always snapshot before acting.** You cannot interact with what you cannot see.
2. **Always re-snapshot after acting.** The UI changed. Your refs are stale.
3. **Use refs, not selectors.** Refs are fast and unambiguous. Selectors search the tree.
4. **Use `-i -c` on snapshots.** Interactive + compact reduces noise by 10x.
5. **Use `id=` selectors when available.** IDs are the most stable across UI changes.
6. **Wait after navigation.** `sleep 2-3` after opening pages, switching tabs, submitting forms.
7. **Verify after typing.** Use `get-value` to confirm the text was set correctly.
8. **One action at a time.** Don't chain multiple actions without checking state between them.
9. **Use `type -s @ref`** over `type -a App`. The selector path uses AXSetValue (reliable). The app path uses keyboard simulation (fragile).
10. **Use `scroll-to @ref`** when you know the element. It's headless. `scroll down` needs focus.

## Troubleshooting

### Element not found

Re-snapshot. Your refs are stale.

```bash
agent-computer-use snapshot -a Safari -i -c
agent-computer-use click @e3                         # use the NEW ref
```

### Ambiguous selector

Multiple elements match. Use refs instead, or add `index=`:

```bash
agent-computer-use click 'role=button index=0' -a Music
```

### Click didn't work

Try double-click (some apps need it to activate items):

```bash
agent-computer-use click @e5 --count 2
```

Or use `scroll-to` first if the element might be offscreen:

```bash
agent-computer-use scroll-to @e5
agent-computer-use click @e5
```

### Type didn't work

Use the selector path, not the app path:

```bash
# wrong (keyboard sim, fragile):
agent-computer-use type "hello" -a Safari

# right (AXSetValue, reliable):
agent-computer-use type "hello" -s @e3
```

### Electron app not using CDP

agent-computer-use auto-detects Electron apps. If it's not working:

```bash
agent-computer-use snapshot -a Slack -i -c -v        # verbose shows CDP status
```

If the app wasn't auto-relaunched, it will be on next run. First run takes ~5s.

## Output format

All output is JSON:

```json
{"success": true, "message": "pressed \"7\" at (453, 354)"}
{"error": true, "type": "element_not_found", "message": "..."}
{"role": "button", "name": "Submit", "value": null, "position": {"x": 450, "y": 320}}
```
