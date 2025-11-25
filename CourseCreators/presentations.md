You are building a professional educational presentation generator. Here's the complete specification:

## PART 1: SLIDE CONTENT GENERATION (Claude 4.5 Sonnet)

When generating presentation slides, use this system prompt:
"You are a world-class educational presentation designer who creates visually dynamic, cognitively engaging slide decks. You balance aesthetic appeal with educational rigor. You understand visual storytelling, Bloom's Taxonomy, and how to structure content for maximum learner engagement. You design presentations that tell a visual story with minimal text, vary slide types, include inquiry-based prompts, build from simple to complex, and create 'wow' moments."

For the user prompt, include:
- Lesson title, topic, duration, learning objectives, and lesson structure
- Generate 8-15 slides with these guidelines:
  1. Max 4 bullet points per slide
  2. Every slide needs a visualAidSuggestion for image generation
  3. Include speaker notes for all slides
  4. Follow Bloom's Taxonomy progression (remember → understand → apply → analyze → create)
  5. Vary slide types (concept, visual metaphor, chart, question, activity, quiz, reflection)
  6. Include at least one "wow" slide with surprising fact or bold visual

Output JSON format:
{
  "title": "Lesson Title",
  "slides": [
    {
      "slideNumber": 1,
      "title": "Slide title",
      "bulletPoints": ["Point 1", "Point 2"],
      "visualAidSuggestions": ["Detailed description of image needed"],
      "speakerNotes": "Teaching guidance"
    }
  ]
}

## PART 2: IMAGE GENERATION (Gemini Nano Banana)

Model: gemini-2.5-flash-image

For each slide's visualAidSuggestions, build an image prompt following this template:
"A [style] educational [type] showing [subject]. [Composition details]. [Color scheme]. [Quality modifiers]. Suitable for [age group]."

Example: "A vibrant, modern educational diagram showing the water cycle. Clean vector style with bright blues and greens. Show evaporation, cloud formation, precipitation, and collection with clear arrows. Professional textbook quality, suitable for KS3 students aged 11-14."

Configuration:
- responseModalities: [TEXT, IMAGE]
- Upload images to Google Drive
- Store URLs for Google Slides insertion

## PART 3: GOOGLE SLIDES FORMATTING

Brand colors (RGB format for API):
- COGNITO_ORANGE = { red: 1.0, green: 0.46, blue: 0.0 }
- COGNITO_DARK = { red: 0.2, green: 0.2, blue: 0.2 }
- WHITE = { red: 1.0, green: 1.0, blue: 1.0 }

Title Slide (Slide 1):
- Layout: TITLE (predefined)
- Background: Cognito Orange
- Title text: White, 44pt, Bold, Centered
- No body/subtitle

Content Slides (Slide 2+):
- Layout: TITLE_AND_BODY
- Background: White
- Title: Cognito Dark, 32pt, Bold
- Body: Black, 18pt, 1.15 line spacing
- Insert images from Google Drive URLs (280x280pt typical)

API workflow:
1. Create empty presentation
2. For each slide: createSlide → updatePageProperties (background) → insertText → updateTextStyle → insertInlineImage
3. Delete default blank slide
4. Move to folder, set permissions

## COMPLETE WORKFLOW

1. User provides lesson details
2. Generate slides with Claude (8-15 slides, max 4 bullets each)
3. For each visualAidSuggestion: generate image with Nano Banana → upload to Drive
4. Create Google Slides: title slide (orange) + content slides (white with images)
5. Apply branding, export to Drive folder
6. Return shareable Google Slides URL

Total time: ~30-60 seconds
