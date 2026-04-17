---
context: none
name: pdf-sg-review
description: Parallel PDF style review workflow using PDF style guide chunks. Reviews PRs, commits, or files against PDF style guide rules. Use with arguments like PR URLs (#123), commit refs (HEAD~1), or file paths. Spawns parallel agents to analyze content against style guide chunks for faster review.
---

<!--
You can use this skill with the following arguments:

- pdf-sg-review https://github.com/org/repo/pull/123 - review a PR by URL
- pdf-sg-review #123 or /pdf-sg-review 123 - review a PR by number
- pdf-sg-review HEAD~1 - review a commit
- pdf-sg-review path/to/file.adoc - review a file
- pdf-sg-review - review the latest commit (default)
-->

# Parallel PDF Style Review Workflow

This skill performs a parallelized PDF style review where multiple agents process chunk files concurrently.

## Content to Review

$ARGUMENTS

Interpret the argument as follows:
- If it's a GitHub PR URL (e.g., `https://github.com/org/repo/pull/123`): use `gh pr diff <number>` to get the diff
- If it's a PR number (e.g., `#123` or `123`): use `gh pr diff <number>` to get the diff
- If it's a commit reference (e.g., `HEAD`, `HEAD~1`, `abc123`): review that commit's diff
- If it's a commit range (e.g., `HEAD~3..HEAD`): review the diff for that range
- If it's a file path (e.g., `docs/guide.adoc`): review that file's content
- If it's a glob pattern (e.g., `modules/**/*.adoc`): review all matching files
- If empty or not provided: review the latest commit (HEAD)

## Overview

Instead of sequentially reading all style guide chunk files and analyzing content against each, this workflow:
1. Checks if chunk files exist, and if not, guides the user through PDF setup
2. Lists all chunk files in the plugins/docs-tools/skills/pdf-sg-review/chunks/ directory
3. Spawns multiple parallel agents (one per chunk)
4. Each agent analyzes the commit content against its assigned chunk's rules
5. Each agent writes findings to a separate temporary file
6. A final merge step combines all findings into the review report

## Instructions

### Phase 0: Verify Chunks Exist

Before starting the review, check if the chunks directory exists and contains files:

1. Run the following command to check for chunk files:
   ```bash
   ls ~/redhat-docs-agent-tools/plugins/docs-tools/skills/pdf-sg-review/chunks/ 2>/dev/null | head -1
   ```

2. **If chunks exist** (command returns output): Proceed to Phase 1.

3. **If chunks are missing** (command returns nothing or errors):

   a. Inform the user:
      ```
      No PDF style guide chunks found. I need to set up your style guides first.
      ```

   b. Check if `pdftotext` is installed:
      ```bash
      which pdftotext
      ```
      If not installed, tell the user:
      ```
      The pdftotext command is required but not installed.
      Please install it with: sudo dnf install poppler-utils (Fedora)
      ```
      Then STOP and wait for the user to install it.

   c. Create the style-guides directory and get the full absolute path:
      ```bash
      mkdir -p ~/redhat-docs-agent-tools/plugins/docs-tools/skills/pdf-sg-review/style-guides && realpath ~/redhat-docs-agent-tools/plugins/docs-tools/skills/pdf-sg-review/style-guides/
      ```

      CRITICAL: You MUST display the path to the user in this EXACT format (copy the path from the realpath command output):
      ```
         Open your file manager and copy your PDF style guides to the following directory:
         <PASTE_THE_FULL_PATH_FROM_REALPATH_COMMAND_HERE>

         ...
      ```

      IMPORTANT: Replace <PASTE_THE_FULL_PATH_FROM_REALPATH_COMMAND_HERE> with the actual path returned by realpath. Do NOT use ~ or any abbreviation. The path must start with /home/ or similar absolute path.

      Then use AskUserQuestion tool with options: ["Done - I uploaded the PDF(s)", "Skip - proceed without PDF review"]

   d. If user chose to skip, inform them that PDF review cannot proceed without chunks and end the review.

   e. If user uploaded PDFs, process them:
      ```bash
      mkdir -p ~/redhat-docs-agent-tools/plugins/docs-tools/skills/pdf-sg-review/chunks
      ```

      For each PDF file found in the style-guides directory:
      ```bash
      cd ~/redhat-docs-agent-tools/plugins/docs-tools/skills/pdf-sg-review/style-guides && for pdf in *.pdf; do
        if [[ -f "$pdf" ]]; then
          basename="${pdf%.pdf}"
          echo "Processing $pdf..."
          pdftotext -layout "./$pdf" "./${basename}.txt"
          split -l 1000 -d -a 3 -e "./${basename}.txt" "../chunks/${basename}_"
          echo "Created chunks for $pdf"
        fi
      done
      ```

   f. Verify chunks were created:
      ```bash
      ls ~/redhat-docs-agent-tools/plugins/docs-tools/skills/pdf-sg-review/chunks/ | wc -l
      ```
      If count is 0, inform the user that no PDFs were found or processing failed, and STOP.

   g. Inform the user of success and proceed to Phase 1.

### Phase 1: Setup

1. Create the reports directory if it doesn't exist, then create the main review report file:
   ```bash
   mkdir -p ~/redhat-docs-agent-tools/plugins/docs-tools/skills/pdf-sg-review/reports
   ```

   Save the report as `~/redhat-docs-agent-tools/plugins/docs-tools/skills/pdf-sg-review/reports/review-<YYYY-MM-DD-hh:mm:ss>.md` with the header:
   ```
   AI review report
   (Do not use preview to read this report unless your previews are set to a monospace font.)

   **User request:** <user's request>

   **Commit:** <commit hash>
   **Subject:** <commit subject>
   ```

2. List all chunk files in `plugins/docs-tools/skills/pdf-sg-review/chunks/` directory and its subdirectories:
   ```
   ls ~/redhat-docs-agent-tools/plugins/docs-tools/skills/pdf-sg-review/chunks/
   ```
   Store the list of chunk filenames for spawning agents.

3. Create the temporary directory if it doesn't exist:
   ```bash
   mkdir -p ~/redhat-docs-agent-tools/plugins/docs-tools/skills/pdf-sg-review/temp
   ```

4. Retrieve the commit diff content and save it to a temporary file:
   ```
   ~/redhat-docs-agent-tools/plugins/docs-tools/skills/pdf-sg-review/temp/commit-diff.txt
   ```

### Phase 2: Parallel Analysis

Launch parallel agents using the Task tool - one agent per chunk file found in Phase 1. Each agent receives:

**Agent Prompt Template:**
```
You are analyzing documentation content for style guide violations.

1. Read the chunk file: ~/redhat-docs-agent-tools/plugins/docs-tools/skills/pdf-sg-review/chunks/<CHUNK_FILENAME>
2. Read the commit diff: ~/redhat-docs-agent-tools/plugins/docs-tools/skills/pdf-sg-review/temp/commit-diff.txt

3. Analyze every sentence in the commit diff against ALL rules in your assigned chunk.

4. Return your findings in your response (do NOT write to any files). Use this format:

   CHUNK: <CHUNK_FILENAME>
   VIOLATIONS_FOUND: <number>

   If violations found, list each one:
   ---VIOLATION---
   FILE: <filename>
   CURRENT: <sentence where violation appears>
   SUGGESTED: <corrected sentence>
   RULE: <first sentence of the style rule>
   TOCPATH: <Style Guide Name> > <Section> > <Subsection>
   ---END---

   If no violations found, return:
   CHUNK: <CHUNK_FILENAME>
   VIOLATIONS_FOUND: 0
   NO_VIOLATIONS_REASON: <brief explanation of what rules were checked>
```

**Parallel Execution:**
- Use a single message with multiple Task tool invocations
- Set subagent_type to "general-purpose"
- Each agent handles exactly one chunk file
- Agents return findings in their response (no file writes needed)
- No permission issues since agents don't write files

### Phase 3: Merge Results

After all parallel agents complete:

1. Parse each agent's response to extract violations:
   - Look for the `---VIOLATION---` markers in agent results
   - Extract FILE, CURRENT, SUGGESTED, RULE, TOCPATH fields
   - Skip agents with `VIOLATIONS_FOUND: 0`

2. Combine findings into the main review report file:
   - Renumber all issues sequentially (Issue 1, Issue 2, etc.)
   - **CRITICAL:** Convert each violation to the exact format specified in the "Review Report Format" section below - do NOT use any other format
   - Deduplicate issues that flag the same sentence

3. Append "End of report" to the review report file

4. Clean up temporary files:
   ```bash
   rm -rf ~/redhat-docs-agent-tools/plugins/docs-tools/skills/pdf-sg-review/temp
   ```

## Review Report Format

Start every response with a line "AI review report".
On the next line after "AI review report", add a line "(Do not use preview to read this report unless your previews are set to a monospace font.)

End every response with a line "End of report".

Each of the other attached sources contains a plurality of rules.

You must review every sentence of the entered text separately, sentence by sentence for violations of all rules (issues) in the sentence.

Number the issues in the order in which you add them.

If you detect only one violation in a sentence, then use the following format to document the violation:

*   **Issue 1**
    *   **File:** <filename from the last line of the entered text that contains `--- a`> (skip this line if there are no instances of `--- a` in the text)
    *   **Current sentence:** `<sentence where the violation appears>` (enclose this sentence with opening and closing `)
    *   **Suggested change:** `<sentence of violation updated to resolve the violation>` (enclose this sentence with opening and closing `)(do not emphasize the changes)
    *   **Style rule:** <copy the first sentence of the style rule>
    *   **TOC path:** *<Source Title>* > *<CHAPTER>* > *<Section>* > *<Subsection>* (include all TOC levels)

(start the next list item, which is for the next sentence, after a blank line)

If you detect multiple violations in a sentence, then use the following format to document the violations for that particular sentence:

*   **Issue 2**
    *   **File:** <filename from the last line of the entered text that contains `--- a`> (skip this line if there are no instances of `--- a` in the text)
    *   **Current sentence:** `<sentence where the violation appears>` (enclose this sentence with opening and closing `)
    *   **Suggested change:** `<sentence of violation updated to resolve the violation>` (enclose this sentence with opening and closing `)(do not emphasize the changes)
    *   **⚠ WARNING!** Sentence with multiple issues! Evaluate suggestions one by one!
    *   **Style rule:** <copy the first sentence of the style rule>
    *   **TOC path:** *<Source Title>* > *<CHAPTER>* > *<Section>* > *<Subsection>* (include all TOC levels)
*   **Issue 3**
    *   **Current sentence:** `<sentence where the violation appears>` (enclose this sentence with opening and closing `)
    *   **Suggested change:** `<sentence of violation updated to resolve the violation>` (enclose this sentence with opening and closing `)(do not emphasize the changes)
    *   **⚠ WARNING!** Sentence with multiple issues! Evaluate suggestions one by one!
    *   **Style rule:** <copy_the_first_sentence_of_the_style_rule>
    *   **TOC path:** *<Source Title>* > *<CHAPTER>* > *<Section>* > *<Subsection>* (include all TOC levels)

(start the next list item, which is for the next sentence, after a blank line)

## Phase 4: Interactive Issue Resolution

After completing all review tasks and generating the report:

1. **Check if issues were found**: If at least one issue was detected during the review, proceed to step 2. If no issues were found, skip this phase.

2. **Prompt the user**: Ask the user whether they want to review and fix the issues one by one using the AskUserQuestion tool with options:
   - "Yes - go through issues one by one"
   - "No - keep the report as-is"

3. **If the user chooses to go through issues**: For each issue in the report, present the issue details in the Review Report Format and offer three choices using the AskUserQuestion tool:
   - **Apply**: Apply the suggested change to the source file
   - **Skip**: Leave the original text unchanged and move to the next issue
   - **Modify**: Allow the user to provide a custom fix (different from the suggested change)

4. **Process user choices**:
   - **Apply**: Use the Edit tool to replace the current sentence with the suggested change in the source file
   - **Skip**: Take no action and proceed to the next issue
   - **Modify**: Wait for the user to provide their custom text, then use the Edit tool to apply their modification

5. **Continue until all issues are processed** or the user requests to stop.
