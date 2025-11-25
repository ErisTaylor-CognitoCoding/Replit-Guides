You are building an educational interactive resource generator. Here's the complete specification:

## INTERACTIVE RESOURCE GENERATION (Claude)

### System Prompt:
"You are an expert educational technology specialist who creates detailed guides for using educational software and coding platforms. You write clear, step-by-step instructions that help teachers and students complete meaningful tasks aligned with learning objectives."

### User Prompt:

Create a comprehensive guide for using ${targetPlatform} to complete a task that teaches the following lesson:

PROJECT CONTEXT:
- Subject: ${project.subject}
- Framework: ${project.curriculumFramework}
- Target Audience: ${project.targetAudience}
- Skill Level: ${project.skillLevel}

LESSON CONTEXT:
- Title: ${lesson.title}
- Topic: ${lesson.topic}
- Duration: ${lesson.duration} minutes
- Learning Objectives: ${JSON.stringify(lesson.learningObjectives)}

LESSON TEACHING ACTIVITIES (align your guide with these):
1. Activity Title (duration min): Description
2. Activity Title (duration min): Description
...

---

### PLATFORM-SPECIFIC GUIDANCE

#### For CodingHub (codinghub.app):
CODINGHUB WORKFLOW (IMPORTANT):
CodingHub (codinghub.app) is a coding canvas platform where:
1. TEACHERS create a new canvas on codinghub.app
2. TEACHERS prepare the canvas with starter code and setup
3. TEACHERS share the canvas link with students or the class
4. STUDENTS open the shared canvas link and work on the code

YOUR GUIDE MUST INCLUDE:
- "Teacher Setup" section: How to create and prepare the canvas
- "Student Instructions" section: What students do once they receive the canvas link
- Starter code recommendations for the canvas

#### For Other Platforms (Scratch, Code.org, etc.):
Create a standard student guide for this platform.

---

### ALIGNMENT REQUIREMENTS:
- This interactive activity should support the Main Teaching phase of the lesson
- Reference the teaching activities when designing the task
- The activity should be hands-on practice of concepts taught in the lesson
- Ensure it aligns with the learning objectives

---

### OUTPUT JSON FORMAT:

{
  "title": "Interactive Activity: [Task Name] using ${targetPlatform}",
  "platform": "${targetPlatform}",
  "platformUrl": "https://platform-url.com",
  "description": "Clear description of what students will create/do",
  "instructions": "Introduction and overview (include Teacher Setup section and Student Instructions section for CodingHub)",
  "activity": {
    "goal": "What students will accomplish",
    "steps": [
      {
        "stepNumber": 1,
        "instruction": "Clear instruction for this step",
        "uiElement": "Which button/menu to use (if applicable)",
        "expectedInput": "What to enter/select (if applicable)",
        "expectedOutcome": "What should happen after this step"
      },
      {
        "stepNumber": 2,
        "instruction": "Next step instruction",
        "uiElement": "UI element to interact with",
        "expectedInput": "What to enter",
        "expectedOutcome": "Expected result"
      }
    ],
    "expectedFinalOutcome": "What the completed project/task should look like or do",
    "troubleshooting": [
      {
        "issue": "Common problem students might encounter",
        "solution": "How to fix or avoid this problem"
      }
    ]
  },
  "extensions": [
    "Challenge task 1 for advanced students",
    "Challenge task 2 for faster finishers"
  ],
  "skillsTargeted": ["skill1", "skill2", "skill3"],
  "estimatedDuration": 30,
  "difficulty": "beginner" | "intermediate" | "advanced"
}

---

## GOOGLE DOCS EXPORT FORMAT

### Document Structure:

1. TITLE (HEADING_1, Centered)
   - Text: Activity title

2. METADATA
   - Platform: [Platform Name]
   - Duration: [X] minutes
   - Difficulty: [easy/medium/hard]

3. DESCRIPTION
   - Brief overview paragraph

4. OVERVIEW (HEADING_2)
   - Instructions/introduction text
   - For CodingHub: Include Teacher Setup and Student Instructions sections

5. ACTIVITY STEPS (HEADING_2)
   - Goal: [Activity goal]
   
   For each step:
   - Step [N]: [Instruction]
     → Use: [UI Element]
     → Enter: [Expected Input]
     ✅ Expected: [Expected Outcome]

6. FINAL OUTCOME (HEADING_3)
   - What the completed project should look like

7. TROUBLESHOOTING (HEADING_2)
   - Common Issue: [Problem]
   - Solution: [How to fix]

8. EXTENSIONS (HEADING_2)
   - Challenge tasks for advanced students

9. SKILLS TARGETED (HEADING_3)
   - Bulleted list of skills

---

## SUPPORTED PLATFORMS

Common platforms to support:
- Scratch (scratch.mit.edu) - Visual block coding
- CodingHub (codinghub.app) - Code canvas with teacher sharing
- Code.org - CS Fundamentals courses
- Tynker - Game-based coding
- Python (Replit/Trinket) - Text-based coding
- micro:bit (makecode.microbit.org) - Hardware programming
- Roblox Studio - Game development
- Minecraft Education - Block-based game coding

---

## COMPLETE WORKFLOW

1. User provides lesson details + target platform
2. Check if CodingHub → add Teacher Setup + Student Instructions sections
3. Build lesson workflow context from teaching activities
4. Generate step-by-step guide with Claude
5. Create Google Doc with proper formatting:
   - Title (Heading 1, centered)
   - Metadata (platform, duration, difficulty)
   - Overview section
   - Numbered activity steps with UI elements
   - Troubleshooting section
   - Extensions for advanced students
6. Export to Google Drive lesson folder
7. Return Google Docs URL

Total time: ~10-15 seconds

---

## STEP STRUCTURE BEST PRACTICES

Each step should include:
1. **stepNumber**: Sequential number (1, 2, 3...)
2. **instruction**: Clear action ("Click the green flag", "Add a new sprite")
3. **uiElement**: Specific UI reference ("Events block palette", "File menu")
4. **expectedInput**: Exact input if needed ("Type 'Hello World'", "Select 'Cat' sprite")
5. **expectedOutcome**: What happens next ("The sprite moves 10 steps", "A new sprite appears")

---

## TROUBLESHOOTING EXAMPLES

Include common issues:
- "Sprite not moving" → "Check that the 'when green flag clicked' block is connected"
- "Code not running" → "Make sure you saved your project first"
- "Canvas not loading" → "Clear browser cache or try a different browser"
- "Can't find the block" → "Check you're in the correct category (Motion, Looks, Sound)"
