---
name: flex-markdown-examples
description: Add example sentences from markdown documents to FieldWorks Language Explorer (FLEx) lexical entries using flextools-mcp. Use this skill when users want to import linguistic examples from markdown tables into their FLEx database, especially when the markdown contains phonetic transcriptions, audio file references, and translations. This skill handles the complete workflow of finding the correct lexical entry, extracting example data from markdown tables, and adding properly formatted examples with audio and translations to FLEx.
compatibility:
  required_tools:
    - read_file
    - use_mcp_tool
  required_mcps:
    - flextools-mcp
---

# FLEx Markdown Examples Importer

This skill helps you add example sentences from markdown documents to FieldWorks Language Explorer (FLEx) lexical entries using the flextools-mcp server.

## When to use this skill

Use this skill when:
- The user wants to add examples from a markdown document to FLEx entries
- The markdown contains linguistic data in table format with phonetic transcriptions, audio files, and translations
- The user needs to batch-import multiple examples into FLEx
- The examples are organized in markdown tables with columns for phonetic form, audio reference, and gloss/translation

## Workflow Overview

The process follows these steps:

1. **Initialize flextools-mcp session** - Set up the API mode and project
2. **Read and parse the markdown** - Extract example data from tables
3. **Find the target lexical entry** - Locate the correct entry in FLEx
4. **Add examples with proper formatting** - Create examples with phonetic text, audio, and translations
5. **Verify the additions** - Confirm examples were added successfully

## Step 1: Initialize the Session

### Step 1a: API Discovery (REQUIRED)

**IMPORTANT**: Before running any operations with flextools-mcp, you MUST first discover APIs. The server requires this even though the skill provides complete working code.

Perform API discovery using one of these methods:

**Option 1: Initialize with task description (recommended)**
```xml
<use_mcp_tool>
<server_name>flextools-mcp</server_name>
<tool_name>start</tool_name>
<arguments>
{
  "api_mode": "flexlibs_stable",
  "task": "Add example sentences to lexical entries by searching for entries by gloss and adding examples with phonetic text, audio files, and translations",
  "project_name": "project-name",
  "write_enabled": false
}
</arguments>
</use_mcp_tool>
```

**Option 2: Quick API search**
```xml
<use_mcp_tool>
<server_name>flextools-mcp</server_name>
<tool_name>search_by_capability</tool_name>
<arguments>
{
  "query": "get all entries from lexicon",
  "max_results": 5,
  "api_mode": "flexlibs_stable"
}
</arguments>
</use_mcp_tool>
```

After API discovery, you can proceed with operations.

### Step 1b: Session Initialization

Start by initializing flextools-mcp with the appropriate settings:

```xml
<use_mcp_tool>
<server_name>flextools-mcp</server_name>
<tool_name>start</tool_name>
<arguments>
{
  "api_mode": "flexlibs_stable",
  "project_name": "project-name",
  "write_enabled": false
}
</arguments>
</use_mcp_tool>
```

**Important**: Always start with `write_enabled: false` for safety. Only enable writes after confirming the operation will work correctly.

## Step 2: Parse the Markdown

Read the markdown file and identify the table structure. Common patterns include:

**Example table format:**
```markdown
| Zhire        | Audio                     | Gloss            |
| ------------ | ------------------------- | ---------------- |
| [mī pə̄ɾ àlì] | ![](I beat Ali.webm)      | I beat Ali       |
| [ŋū pə̄ɾ àlì] | ![](you beat Ali.webm)    | you beat Ali     |
```

Extract:
- **Phonetic form**: Text in square brackets in the markdown (e.g., `[mī pə̄ɾ àlì]`), but store WITHOUT brackets in FLEx since the writing system already indicates it's phonetic
- **Audio filename**: From image markdown syntax (e.g., `I beat Ali.webm`)
- **Translation/gloss**: Plain text in the gloss column

## Step 3: Find the Lexical Entry

**CRITICAL**: Finding the correct entry is essential. A wrong entry will corrupt your data.

### Strategy 1: Search by Gloss (Recommended)

Search for entries by their meaning/gloss rather than headword. This is more reliable because:
- Glosses are unique and descriptive
- Headwords may have similar spellings (e.g., "pər" vs "kpər")
- You can search in the user's language

```python
# Search for entries with 'beat' or 'flog' in the gloss
entries = project.LexiconAllEntries()
candidates = []
for entry in entries:
    headword = safe_str(entry.HeadWord)
    for sense in entry.SensesOS:
        gloss = safe_str(sense.Gloss.BestAnalysisAlternative).lower()
        if 'beat' in gloss or 'flog' in gloss:
            candidates.append((entry, sense, headword, gloss))
            report.Info(f'Found: {headword} - {gloss}')
```

### Strategy 2: Exact Headword Match

If you must search by headword, use exact matching:

```python
# Search for exact headword match
target_headword = 'pər'  # NOT 'kpər'
entries = project.LexiconAllEntries()
for entry in entries:
    headword = safe_str(entry.HeadWord)
    if headword == target_headword:  # Exact match, not substring
        report.Info(f'Found: {headword}')
        break
```

### Step 3b: User Confirmation (REQUIRED)

**ALWAYS confirm with the user before proceeding:**

1. Display all matching entries with their glosses
2. Ask the user to confirm which entry and sense to use
3. Show the number of existing examples
4. Only proceed after explicit confirmation

Example confirmation message:
```
I found these matching entries:
1. pər - to flog; to beat (2 existing examples)
2. kpər - to make a hole (0 existing examples)

Which entry should I add the examples to? Please confirm the entry number.
```

## Step 4: Preview and Confirm Examples

**PREREQUISITE**: User has confirmed the correct entry and sense.

Before adding examples, display them to the user for final confirmation:

```python
# Display examples to be added
report.Info('Examples to be added:')
for i, (phonetic, audio, translation) in enumerate(examples_data, 1):
    report.Info(f'{i}. {phonetic}')
    report.Info(f'   Audio: {audio}')
    report.Info(f'   Translation: {translation}')
    report.Info('')
```

**Then ask the user**: "These are the examples I will add. Should I proceed?"

Only after user confirms "yes", proceed to Step 5.

## Step 5: Add Examples to the Entry

Once you've confirmed both the entry AND the examples to add, add each example with:
- Phonetic transcription in the appropriate writing system
- Audio file reference
- Translation

**Key writing systems to use:**
- `zhi-fonipa-x-etic` - Phonetic transcription (not phonemic)
- `zhi-Zxxx-x-audio` - Audio file references
- `en` - English translations

**Complete code pattern with confirmation:**

```python
from SIL.LCModel.Core.Text import TsStringUtils
from SIL.LCModel import *
import System

# Step 1: Find candidate entries by gloss
entries = project.LexiconAllEntries()
candidates = []
for entry in entries:
    headword = safe_str(entry.HeadWord)
    for sense in entry.SensesOS:
        gloss = safe_str(sense.Gloss.BestAnalysisAlternative)
        if 'beat' in gloss.lower() or 'flog' in gloss.lower():
            candidates.append({
                'entry': entry,
                'sense': sense,
                'headword': headword,
                'gloss': gloss,
                'example_count': sense.ExamplesOS.Count
            })

# Step 2: Display candidates and get user confirmation
if not candidates:
    report.Error('No entries found with "beat" or "flog" in gloss')
elif len(candidates) == 1:
    report.Info(f'Found 1 entry: {candidates[0]["headword"]} - {candidates[0]["gloss"]}')
    report.Info(f'Existing examples: {candidates[0]["example_count"]}')
    # Proceed with this entry after user confirms
    target_entry = candidates[0]['entry']
    sense = candidates[0]['sense']
else:
    report.Info(f'Found {len(candidates)} matching entries:')
    for i, c in enumerate(candidates):
        report.Info(f'{i+1}. {c["headword"]} - {c["gloss"]} ({c["example_count"]} examples)')
    # STOP HERE and ask user which entry to use
    # After user confirms, use: target_entry = candidates[user_choice]['entry']

# Step 3: Add examples (only after user confirmation)
if target_entry:
    cache = sense.Cache
    
    # Create example
    example_factory = cache.ServiceLocator.GetService(ILexExampleSentenceFactory)
    new_example = example_factory.Create()
    sense.ExamplesOS.Add(new_example)
    
    # Set phonetic text (without square brackets - FLEx knows it's phonetic)
    ws_phonetic = project.WSHandle('zhi-fonipa-x-etic')
    new_example.Example.set_String(ws_phonetic,
        TsStringUtils.MakeString('phonetic_text', ws_phonetic))
    
    # Set audio file
    ws_audio = project.WSHandle('zhi-Zxxx-x-audio')
    new_example.Example.set_String(ws_audio, 
        TsStringUtils.MakeString('audio_filename.webm', ws_audio))
    
    # Create translation
    free_trans_guid = System.Guid('d7f7164a-e8cf-11d3-9764-00c04f186933')
    trans_type = cache.ServiceLocator.ObjectRepository.GetObject(free_trans_guid)
    trans_factory = cache.ServiceLocator.GetService(ICmTranslationFactory)
    translation = trans_factory.Create(new_example, trans_type)
    new_example.TranslationsOC.Add(translation)
    
    # Set translation text
    ws_en = project.WSHandle('en')
    translation.Translation.set_String(ws_en, 
        TsStringUtils.MakeString('English translation', ws_en))
    
    report.Info(f'Added example: {phonetic_text}')
```

**Important API differences in flexlibs_stable:**
- Use `project.LexiconAllEntries()` not `project.lexicon.entries`
- Use `entry.HeadWord` not `entry.headword`
- Use `entry.SensesOS` not `entry.senses`
- Use `report.Info()` and `report.Error()` (capital letters) not lowercase
- Use `safe_str()` helper function to convert multistrings to strings

## Step 6: Verify and Enable Writes

**Two-stage verification process:**

### Stage 1: Dry-run with write_enabled=false
This will fail with "Not in the right state to register a change" because LibLCM tries to register changes even in read-only mode. This is expected behavior.

### Stage 2: Actual execution
After user confirms:
1. The correct entry and sense have been identified
2. The examples to add are correct
3. They understand this will modify the database

Then run with `write_enabled: true` and `confirmed: true`.

### Stage 3: Verification
After execution:
1. Query the database to confirm examples were added
2. Check the example count matches expectations
3. Verify no duplicate examples were created
4. Ask user to check in FLEx that audio files play correctly

## Important Notes

### Audio File Placement
Audio files must be in the correct location for FLEx to play them:
- Check the LinkedFiles directory in your FLEx project
- Ensure audio filenames match exactly (including extension)
- The play button in FLEx will only appear if the file exists

### Writing System Codes
Always verify the correct writing system codes for your project:
- Use `zhi-fonipa-x-etic` for phonetic (not `zhi-fonipa-x-emic` for phonemic)
- Audio writing system is typically `{lang}-Zxxx-x-audio`
- Translation writing system depends on the language (e.g., `en`, `fr`)

### Safety First
- Always start with `write_enabled: false` to test
- Review the operation output before enabling writes
- Backup the FLEx project before making bulk changes
- Use `confirmed: true` for Create/Update/Delete operations

### Batch Processing
When adding multiple examples:
- Process them one at a time initially to verify the pattern works
- Once confirmed, you can batch multiple examples in a single operation
- Keep operations focused on a single lexical entry at a time for clarity

## Common Issues and Solutions

**Issue**: Audio play button doesn't appear in FLEx
- **Solution**: Verify the audio file exists in the LinkedFiles directory with the exact filename

**Issue**: Wrong writing system used
- **Solution**: Use `zhi-fonipa-x-etic` for phonetic transcriptions, not the phonemic variant

**Issue**: Translation not showing
- **Solution**: Ensure you're creating the translation object with the correct translation type GUID

**Issue**: Duplicate examples created
- **Solution**: Check if examples already exist before adding; delete duplicates manually in FLEx if needed

## Example Workflow

Here's a complete example of adding "we beat Ali", "you(pl) beat Ali", and "they beat Ali" to the entry for "pər" (beat):

1. **Initialize**: `start(api_mode="flexlibs_stable", project_name="zhi-flex", write_enabled=false)`
2. **Parse markdown**: Read lines 148-155, extract the three remaining examples
3. **Find candidates**: Search for entries with "beat" or "flog" in gloss
4. **Display options**: Show user all matching entries with their glosses and example counts
5. **Get entry confirmation**: User confirms which entry (e.g., "pər - to flog; to beat")
6. **Preview examples**: Display the examples to be added with phonetic, audio, and translation
7. **Get examples confirmation**: User confirms the examples look correct
8. **Execute**: Run with `write_enabled=true, confirmed=true`
9. **Verify**: Query database to confirm examples were added (not duplicates)
10. **User check**: Ask user to verify in FLEx that examples appear with audio playback

## Tips for Success

- **Always search by gloss first**: More reliable than headword substring matching
- **Always get user confirmation**: Display all candidates and let user choose
- **Check for duplicates**: Query existing examples before adding new ones
- **Verify after execution**: Don't trust success messages - query the database to confirm
- **Start with small batches**: Add 1-3 examples at a time, not dozens
- **Check existing examples**: Look at examples already in FLEx to understand the expected format
- **Verify writing systems**: Confirm the correct writing system codes for the project
- **Test audio placement**: Ensure audio files are in LinkedFiles directory before adding references
- **Communicate clearly**: Explain each step to the user so they understand what's happening

## Common Mistakes to Avoid

1. **Substring matching on headwords**: "pər" matches both "pər" and "kpər" - use exact match or search by gloss
2. **Skipping user confirmation**: Always confirm the entry before adding examples
3. **Trusting success messages**: The operation may report success but add to wrong entry - always verify
4. **Not checking for duplicates**: Query existing examples first to avoid creating duplicates
5. **Using lowercase report methods**: Use `report.Info()` not `report.info()`
6. **Wrong API for mode**: In flexlibs_stable, use `project.LexiconAllEntries()` not `project.lexicon.entries`