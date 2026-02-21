---
description: Compare a trip's packing list against past trips to find missing items and logistics gaps
---

<task>
Analyze a trip in Notion by comparing its packing list and logistics against similar past trips, then offer to update the trip page with missing items.
Argument: $ARGUMENTS (trip name to search for, e.g. "Selva" or "Japan 2025")
</task>

<context>
- Notion database name: "My Trips"
- Search via `mcp__notion__notion-search`, fetch via `mcp__notion__notion-fetch`
- Update via `mcp__notion__notion-update-page` with `insert_content_after` command
- Trip pages use tags like "Skiing", "Smart Casual", "Photography", "Snow", "Multi-Weather" for matching similar trips
- Packing lists are structured in columns (bags) with checkbox items
- Orange-colored items indicate items stored at a different location
</context>

<workflow>
Follow these stages strictly in order:

**Stage 1: Find the Trip**
1. If `$ARGUMENTS` is empty, ask the user which trip to analyze
2. Use `mcp__notion__notion-search` to search the "My Trips" database for the trip name from `$ARGUMENTS`
3. If multiple results, ask the user to pick one
4. Display: "Found trip: <trip name>"

**Stage 2: Fetch the Target Trip**
1. Use `mcp__notion__notion-fetch` to get the full content of the target trip page
2. Extract:
   - Tags/properties (Skiing, International, Photography, etc.)
   - All packing list items (including which are checked/unchecked)
   - Logistics sections (transportation, hotel, budget, weather, itinerary, restaurants)
   - Any retrospective/review notes

**Stage 3: Find and Fetch Comparable Trips**
1. Use `mcp__notion__notion-search` to find other trips in "My Trips"
2. Select 4-5 trips that share the most tags with the target trip. Prioritize:
   - Same activity type (e.g. other ski trips for a ski trip)
   - Same travel style (international vs domestic)
   - Similar multi-weather or seasonal conditions
3. Use `mcp__notion__notion-fetch` to get the full content of each comparison trip
4. Display which trips you're comparing against and why

**Stage 4: Compare Packing Lists**
1. Extract all packing list items from each comparison trip
2. Count how many comparison trips include each item
3. Identify items that appear in comparison trips but are MISSING from the target trip
4. Categorize missing items by frequency:
   - **Critical** (in every comparable trip): These are almost certainly needed
   - **Recommended** (in most comparable trips): Very likely needed
   - **Consider** (in some trips, or particularly relevant to destination): Worth thinking about
5. Ignore items that are clearly destination-specific and irrelevant (e.g. beach gear for a ski trip)

**Stage 5: Check Logistics Gaps**
1. Compare what planning sections exist in the target trip vs comparison trips
2. Flag any commonly-present sections that are missing, such as:
   - Transportation details
   - Hotel/accommodation bookings
   - Budget breakdown
   - Weather research
   - Day-by-day itinerary
   - Restaurant reservations
   - Emergency contacts / insurance

**Stage 6: Check Retrospectives**
1. Look at retrospective/review sections of comparison trips for:
   - "What could be improved" notes
   - "Should've brought" or "forgot to pack" mentions
   - Any lessons learned relevant to the target trip type
2. Surface relevant suggestions from past trip learnings

**Stage 7: Present Findings**
1. Display a categorized summary:

   ```
   === Missing Packing Items ===

   CRITICAL (in all comparable trips):
   - [ ] Item 1
   - [ ] Item 2

   RECOMMENDED (in most comparable trips):
   - [ ] Item 3
   - [ ] Item 4

   CONSIDER:
   - [ ] Item 5 (from trip X retrospective: "wish I had brought this")
   - [ ] Item 6

   === Logistics Gaps ===
   - Missing: budget breakdown (present in 4/5 comparable trips)
   - Missing: restaurant reservations (present in 3/5 comparable trips)

   === Lessons from Past Trips ===
   - [Trip Name]: "relevant retrospective note"
   ```

**Stage 8: Offer to Update**
1. Ask the user which items they want to add to the Notion page
2. Ask which bag/column the items should be added to (or suggest based on where similar items appear in comparison trips)
3. Before updating, fetch the `notion://docs/enhanced-markdown-spec` resource to ensure correct formatting
4. Use `mcp__notion__notion-update-page` with `insert_content_after` to add the selected items to the packing list
5. Confirm what was added

</workflow>

<error_handling>
- If no trip found: Ask user to refine the search term or provide the Notion page URL
- If trip has no packing list yet: Note this and still show what comparable trips pack
- If fewer than 3 comparable trips found: Proceed with what's available but note the limited comparison set
- If Notion tools are unavailable: Inform the user that Notion MCP tools are required
</error_handling>
