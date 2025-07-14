---
title: Resume Copilot with GPT4 + Streamlit
author: ch4174nya
date: 2024-12-22 10:10:00 +0800
tags: ["GenerativeAI", "streamlit", "python"]
---

Resumes can be tricky.
Whether you're a new grad or a senior professional, crafting the _right_ bullet points and targeting the _right_ roles can be tedious and frustrating -- particularly when applying across different job titles.

As someone exploring real-world applications of AI and in the market for roles, I wondered: "_Can GPT-4 assist people revise and improve their resumes?_"

That's how I started prototyping **Resume Copilot** -- a GPT powered tool that: 
- Extracts content from your resume
- Suggests relevant job roles
- Improves resume bullet points
- Gives feedback on the overall resume, in terms of missing skills and improving content
- Compares your resume to actual job descriptions using RAG (Retrieval-Augmented Generation)

## What the App Does

<img src="{{site.url}}{{site.baseurl}}/assets/img/resume-copilot-architecture.png" alt="Resume Copilot Architecture" width="800"/>


I ideated this prototype with the following features in mind: 
- Upload your resume as a PDF
- See GPT-4’s suggestions on what roles you might fit
- Paste a bullet point and get an improved, quantifiable version
- Get instant, personalized resume improvement tips
- Paste any job description and receive pointers on how your resume compares, using vector search + GPT-4 feedback

i.e., with a clear focus: no login, no fluff just helpful feedback in a few clicks.

## The Stack

### Streamlit for UI
I used Streamlit to rapidly create a clean UI with a resume PDF uploader, some LLM output text areas (with some of them hidden away to take advantage of streamlit components) and buttons to trigger various functionalities.

### PyMuPDF to Extract Resume Text
I used `PyMuPDF` to extract text from the uploaded PDF resume, as it is fast, clean and handles multi-page files.
```python
import fitz

def extract_text_from_pdf(file):
    document = fitz.open(stream=file.read(), filetype="pdf")
    text = ""
    for page in document:
        text += page.get_text()
    return text
```

### GPT-4o-mini for intelligence
I used OpenAI's `GPT-4o-mini` model for:
- role matching (_Suggest 3 job titles matching this resume..._)
- resume bullet point revisions
- feedback generation for the overall resume (e.g., missing skills, turning vague points to impactful ones, etc.)
- comparison against a given job description

### RAG-powered JD Comparator
I added a second page to the app for comparing any given job description with the resume:
1. **Resume Parsing + Chunking**: the resume PDF gets chunked into smaller sections
2. **Embeddings + Vector DB**: Each chunk is embedded using SentenceTransformers and stored in a local FAISS index
3. **User provided JD**: Embeddings for the job description are generated and used as a query to retrieve the top-5 most similar resume chunks
4. **GPT-4 feedback**: The retrieved chunks + JD are passed to GPT-4o-mini to generate feedback along the lines of alignment, gaps and suggestions.

This is the RAG pattern and made the feedback far more relevant and targeted to the give role.


## My Learnings
- Prompt Design matters -- even small tweaks can change output quality dramatically. For instance, the overall feedback was being generated on the text version of the resume PDF. Therefore, I had to modify the prompt such that the LLM skips feedback on headers and subheadings as plaintext is devoid of such richness anyway.
- The LLM did a fine job at suggesting roles but might need fine-tuning for some industry-specific resumes.
- RAG-powered feedback is far more useful as it is targeted to the given role. Using this I was able to ground the LLM's output in the context of the provided role.
- Streamlit's simplicity makes it ideal for demos, and fast feedback loops.
- While the core functionality relies heavily on the LLM being used and how good it is (plus prompt engineering!), a real-world tool needs more things-- UX, clarity and speed.

Adding RAG into the flow was a game-changer, it transitions the app from AI-feature to real AI product.

## Try It Out
- [Live Demo](https://genai-resume-copilot-aaz7xhssqfnn3auazspr4u.streamlit.app/)
- [GitHub Repo](https://github.com/ch4174nya/genAI-resume-copilot)


## Screengrab
<img src="{{site.url}}{{site.baseurl}}/assets/img/demo-resume-copilot.gif" alt="Demo gif" width="800"/>

---
---

## What’s Next?
While this was a fun hands-on project to try out GenAI product development, real-world user value and speed prototyping, I can already think of upgrades like:
- Matching resumes against real job descriptions (via scraping or APIs)
- Saving feedback history per user
- Adding industry-specific suggestions (e.g., PM, DevOps, Research roles)
- Exporting feedback for users to save and work on offline
