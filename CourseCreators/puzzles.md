You are building an educational puzzle generator for word searches and crosswords. Here's the complete specification:

## WORD SEARCH GENERATION

### PART 1: AI Word Selection (Claude)

System prompt:
"You are an expert educational content creator who selects key vocabulary terms for educational puzzles. You choose words that are directly tied to the lesson topic and learning objectives."

User prompt:
Select 12-15 key vocabulary words for a word search puzzle for this lesson:

PROJECT CONTEXT:
- Subject: ${project.subject}
- Framework: ${project.curriculumFramework}
- Target Audience: ${project.targetAudience}
- Skill Level: ${project.skillLevel}

LESSON CONTEXT:
- Title: ${lesson.title}
- Topic: ${lesson.topic}
- Learning Objectives: ${JSON.stringify(lesson.learningObjectives)}

WORD SELECTION REQUIREMENTS:
- Select 12-15 key vocabulary terms DIRECTLY related to the topic
- All words must be curriculum-relevant terms from the lesson
- Include a mix of word lengths (4-12 letters)
- Words should be single terms (no phrases with spaces)
- Remove any hyphens or special characters

RETURN FORMAT (must be valid JSON):
{
  "title": "Word Search: [Topic Name]",
  "instructions": "Find all the words related to [topic]",
  "words": ["word1", "word2", "word3", ...],
  "difficulty": "beginner" | "intermediate" | "advanced",
  "topic": "Topic Name"
}

### PART 2: Grid Generation (Library)

Use @sbj42/word-search-generator library to create perfect grid:

const puzzle = wordSearchLib.generate({
  height: 15,
  width: 15,
  words: words.map(w => w.toUpperCase().replace(/[^A-Z]/g, '')),
  allowBackwards: skillLevel === 'advanced',
  allowDiagonal: skillLevel === 'intermediate' || skillLevel === 'advanced'
});

Difficulty settings:
- Beginner: No backwards, no diagonals
- Intermediate: No backwards, allow diagonals
- Advanced: Allow backwards and diagonals

### PART 3: PDF Formatting (jsPDF)

Document settings:
- Orientation: Portrait
- Format: A4
- Unit: mm

Layout:
1. TITLE
   - Font: Helvetica Bold, 20pt
   - Color: Orange RGB(255, 107, 53)
   - Position: Centered, Y=20

2. METADATA
   - Text: "Topic: [topic] | Difficulty: [difficulty]"
   - Font: Helvetica Normal, 10pt
   - Color: Gray RGB(100, 100, 100)
   - Position: Centered

3. INSTRUCTIONS
   - Font: Helvetica Italic, 11pt
   - Color: Dark Gray RGB(60, 60, 60)
   - Position: Centered

4. GRID
   - Cell size: 8mm
   - Grid: Centered on page
   - Lines: 0.3pt black
   - Letters: Helvetica Bold, 10pt, centered in cells

5. WORD LIST
   - Header: "Words to Find (N):" in Bold 12pt
   - Words: 3 columns, 9pt
   - Position: Below grid

---

## CROSSWORD GENERATION

### PART 1: AI Word Selection (Claude)

System prompt:
"You are an expert educational content creator who selects key vocabulary terms for crossword puzzles. You choose words that are directly tied to the lesson topic and create crossword grid layouts."

User prompt:
Select crossword vocabulary words for this lesson:

PROJECT CONTEXT:
- Subject: ${project.subject}
- Framework: ${project.curriculumFramework}
- Target Audience: ${project.targetAudience}
- Skill Level: ${project.skillLevel}

LESSON CONTEXT:
- Title: ${lesson.title}
- Topic: ${lesson.topic}
- Learning Objectives: ${JSON.stringify(lesson.learningObjectives)}

TASK: Select 10-12 curriculum-relevant single words from this lesson topic.

REQUIREMENTS:
1. All words must be KEY TERMS from the topic
2. Single words only (no phrases)
3. Words should be 4-10 letters long
4. Mix of different lengths for variety
5. Choose words that share common letters (e.g., TEST, QUALITY both have "T")

RETURN FORMAT (must be valid JSON):
{
  "title": "Crossword: [Topic Name]",
  "instructions": "Complete the crossword using your knowledge of [topic]",
  "words": ["TESTING", "QUALITY", "DEFECT", "VALIDATION", "BUG", "CODE"],
  "difficulty": "beginner" | "intermediate" | "advanced",
  "topic": "Topic Name"
}

IMPORTANT: Return ONLY words (no positions, no clues yet).

### PART 2: Real Definitions (Datamuse API)

Fetch real dictionary definitions for each word:

const response = await fetch(`https://api.datamuse.com/words?sp=${word}&md=d&max=1`);
const data = await response.json();
const definition = data[0]?.defs?.[0];

Skip words without definitions.

### PART 3: Grid Layout (crossword-layout-generator library)

Use crossword-layout-generator to arrange words:

const layout = clg.generateLayout(wordsWithClues);

Returns:
- grid: 2D array with letters and nulls
- across: Array of { number, clue, row, col }
- down: Array of { number, clue, row, col }

### PART 4: PDF Formatting (jsPDF)

Document settings:
- Orientation: Portrait
- Format: A4
- Unit: mm

Layout:
1. TITLE
   - Font: Helvetica Bold, 20pt
   - Color: Pink RGB(236, 72, 153)
   - Position: Centered, Y=20

2. METADATA
   - Text: "Topic: [topic] | Difficulty: [difficulty]"
   - Font: Helvetica Normal, 10pt
   - Color: Gray RGB(100, 100, 100)

3. INSTRUCTIONS
   - Font: Helvetica Italic, 11pt
   - Color: Dark Gray RGB(60, 60, 60)

4. CROSSWORD GRID
   - Cell size: 8mm
   - White cells: Empty squares with black borders
   - Black cells: Filled black (null in grid)
   - Numbers: Small numbers in top-left of cells
   - Letters: Hidden (for puzzle) or shown (for answer key)

5. CLUES
   - Two columns: "ACROSS" and "DOWN"
   - Format: "1. Clue text here"
   - Font: 9pt
   - Clues sorted by number

---

## OUTPUT FORMATS

### Word Search JSON:
{
  "title": "Word Search: Topic Name",
  "instructions": "Find all the hidden words",
  "grid": [["A","B","C",...], ["D","E","F",...], ...],
  "words": ["WORD1", "WORD2", ...],
  "topic": "Topic Name",
  "difficulty": "intermediate"
}

### Crossword JSON:
{
  "title": "Crossword: Topic Name",
  "instructions": "Complete the crossword",
  "grid": [["A",null,"B",...], ...],
  "across": [
    { "number": 1, "clue": "Definition here", "row": 0, "col": 0 }
  ],
  "down": [
    { "number": 2, "clue": "Definition here", "row": 0, "col": 2 }
  ],
  "topic": "Topic Name",
  "difficulty": "intermediate"
}

---

## COMPLETE WORKFLOW

### Word Search:
1. AI selects 12-15 curriculum-relevant vocabulary words
2. Library generates perfect 15x15 grid with words hidden
3. Configure difficulty (backwards/diagonals based on skill level)
4. Generate PDF with orange branding
5. Export to Google Drive

### Crossword:
1. AI selects 10-12 curriculum-relevant vocabulary words
2. Datamuse API fetches real dictionary definitions
3. Library arranges words into interlocking grid
4. Generate PDF with pink branding + clues
5. Export to Google Drive

### Libraries Used:
- @sbj42/word-search-generator (word search grids)
- crossword-layout-generator (crossword layouts)
- Datamuse API (real definitions)
- jsPDF (PDF generation)

Total time: ~5-10 seconds per puzzle
