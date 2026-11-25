# Level 12: Multimodal

**Phase 6: Frontier & proof** · Previous: [Level 11](../5-production/11-managed-ai-platforms.md) · Next: [Level 13](13-fine-tuning.md)

**Goal:** understand how models handle images, audio and video, why they fail differently
from text-only models, and how to choose and use one when a task is not just text.

**Skip or defer this level if** your work is text-only. Sections 4 (documents) and 8
(voice) are the most widely useful.

A **modality** is a type of data: text, image, audio, video. A **multimodal** model
handles more than one.

---

## 1. What a multimodal model is

From level 1: a language model turns text into tokens, turns tokens into vectors, and
processes the vectors. A multimodal model adds extra front ends that turn *other* kinds of
data into the same kind of vectors.

```
Text   → text tokenizer  ┐
Image  → image encoder   ├─► one shared stream of vectors ─► the language model ─► output
Audio  → audio encoder   ┘
```

Once a picture has become vectors, the model reasons about it with the same machinery it
uses for words. That is why you can ask a question about a photo in plain language.

**Input and output are separate questions.** Check both for any model:

| Model type | Reads | Writes |
|---|---|---|
| Vision-language model | Text + images | Text |
| Image generator | Text | Images |
| Speech-to-text | Audio | Text |
| Text-to-speech | Text | Audio |
| "Omni" models | Text + images + audio | Text + audio, sometimes images |

Many people assume a model that can *see* images can also *draw* them. Often it cannot.

---

## 2. Vision: how images become tokens

1. The image is cut into a grid of small squares called **patches**.
2. Each patch is turned into a vector, much as a word would be.
3. Those vectors go into the model alongside the text.

So an image is "read" as a sequence of little squares.

**Resolution drives cost.** More pixels means more patches, which means more tokens.

*Illustration:* a small thumbnail might cost a few hundred tokens; a full-page
high-resolution scan can cost a couple of thousand or more. Exact numbers vary by model.

The trade-off:

- **Shrink the image** → cheaper and faster, but small print becomes unreadable.
- **Keep it large** → the model can read fine detail, at higher cost.

*Example:* a photo of a receipt. At full size the model reads every line. Shrunk to a
quarter, the totals blur and it starts guessing numbers. Whenever a vision model misreads
small text, check the resolution you actually sent before blaming the model.

**Sending an image:**

```python
import base64

with open("receipt.jpg", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode()

response = client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=500,
    messages=[{"role": "user", "content": [
        {"type": "image",
         "source": {"type": "base64", "media_type": "image/jpeg", "data": image_data}},
        {"type": "text", "text": "List each item and its price as a table, then the total."},
    ]}],
)
```

Put the image before the question, just as you put long documents before the question (level 3).

---

## 3. Vision: strengths and weaknesses

**Usually good at:**

- Describing what is in a picture.
- Reading printed and much handwritten text.
- Understanding charts, diagrams and screenshots.
- Comparing two images.
- Turning a sketch or screenshot of a web page into code.

**Usually weak at:**

| Weakness | Example |
|---|---|
| Exact counting | "How many people are in this crowd?" gives 40 when there are 53 |
| Spatial relations | Confusing left and right; which object is behind which |
| Precise measurement | Reading an exact value off a dial or an unlabelled chart |
| Tiny details | A serial number in the corner of a large photo |
| Many similar items | Which of 30 identical-looking boxes on a shelf is out of place |

**They hallucinate too.** A vision model may describe an object that fits the scene but is
not there, or "read" blurred text as something plausible.

**Ways to get better results:**

- Crop to the part that matters, so those pixels get the full resolution.
- Ask it to describe what it sees first, then answer the question.
- For numbers that matter, ask twice in different ways, or check against a second source.
- Give it a tool: let it zoom, or run code to measure.

---

## 4. Vision: documents, charts and visual RAG

Level 4 warned that turning PDFs into text breaks tables, columns and charts. Vision offers
another route: **give the model a picture of the page.**

*Example.* A financial report page has a table of quarterly revenue and a bar chart.

- *Text extraction* gives: `Q1 Q2 Q3 Q4 4.2 4.8 5.1 6.0 Revenue Costs 3.1 3.3...` The
  layout is gone and the chart is lost entirely.
- *Page image:* the model sees the table as a table and the chart as a chart, and can
  answer "which quarter had the highest margin?"

**When to send page images:** complex layouts, tables, charts, forms, scans, slides.
**When plain text is fine:** simple running text, where it is cheaper.

**Visual RAG.** The level 4 pipeline, but with pages treated as images.

```
Usual RAG:    PDF → extract text → chunk → embed text → search → answer
Visual RAG:   PDF → picture of each page → embed the picture → search → send the
              matching page pictures to a vision model → answer
```

This skips the fragile text-extraction step. The cost is larger storage and more tokens
per answer. A well-known research method of this kind is called ColPali.

A practical middle path: have a vision model write a text description of each chart and
table when you first load the documents, and index that text alongside the rest.

---

## 5. Image and video generation

Most image generators work by **diffusion**. The idea:

1. Start with pure visual noise, like television static.
2. Remove a little noise, guided by your text description.
3. Repeat many times until a clear picture appears.

A sculptor's version: begin with a rough block and keep removing what does not look like
the description.

**Writing prompts for images** differs from chat. Describe the picture, not a task:

```
Weak:    "a dog"
Better:  "A golden retriever puppy sitting on a wooden porch at sunset, warm side
          lighting, shallow depth of field, photograph"
```

Cover: the subject, the setting, the lighting, the style, the framing.

**Known difficulties:**

- **Text inside images** often comes out misspelt, though this is improving.
- **Consistency:** keeping one character looking the same across several pictures is hard.
- **Exact counts and positions:** "three red apples to the left of two green pears" may
  not come out right.

**Video generation** extends the same idea across time. It costs far more, clips are
short, and things can drift or change between frames.

**Responsibilities:** read the usage rules of whatever tool you use. Do not make deceptive
images of real people. Many tools add an invisible mark or label saying the content is
AI-made; keep it.

---

## 6. Audio: how sound becomes tokens

Sound is a wave: air pressure changing over time. A recording stores that as thousands of
numbers per second, which is too many to feed in directly.

**The usual steps:**

1. Turn the wave into a **spectrogram**: a picture of which pitches are present at each
   moment. Time runs left to right, pitch runs bottom to top, brightness shows loudness.
2. An audio encoder turns slices of the spectrogram into vectors.
3. Those vectors go into the model, like image patches.

In effect, the model "looks at" sound as a picture.

Some systems instead turn audio into **audio tokens**, a kind of sound alphabet, so the
model can both read and write audio the way it reads and writes text. This is what lets
newer models speak directly instead of going through separate steps.

**Why you care:** audio is long. A minute of speech becomes many tokens, so long
recordings are costly and may need to be cut into pieces.

---

## 7. Speech to text, and text to speech

### Speech to text (transcription)

Turns a recording into written words. Whisper is a well-known open model you can run
yourself for free.

```python
# pip install openai-whisper
import whisper

model = whisper.load_model("base")
result = model.transcribe("meeting.mp3")
print(result["text"])
```

**How accuracy is measured:** word error rate, the share of words that came out wrong.
5% means 1 word in 20.

**What makes it worse:**

- Background noise and people talking over each other.
- Strong accents the model heard little of in training.
- Special terms: drug names, product codes, people's names.
- Poor microphones and phone-line audio.

**Fixes:** give the model a list of expected names and terms if it supports that; clean up
the output afterwards with a language model ("fix obvious transcription errors; our product
is called Lumen, not Lumin").

**Who said what** is a separate task called speaker diarisation: labelling each stretch of
speech with a speaker.

### Text to speech

Turns writing into a spoken voice. What to compare:

- **Naturalness:** does it sound like a person?
- **Speed to first sound:** matters a great deal for conversation.
- **Streaming:** can it start speaking before the whole text is ready?
- **Control:** pace, emotion, choice of voice.

Only copy a real person's voice with their clear permission.

---

## 8. Real-time voice agents

A voice agent is something you talk with, like a phone assistant. There are two ways to
build one.

**Design 1: a chain of three**

```
Your speech → [speech to text] → text → [language model] → text → [text to speech] → voice
```

- For: you can pick the best part for each step, use any text model and all your existing
  tools, and read exactly what was said.
- Against: three delays add up, and tone of voice is lost when speech becomes text.

**Design 2: speech to speech**

```
Your speech → [one model that hears and speaks] → voice
```

- For: quicker and more natural; it can pick up tone and respond with feeling.
- Against: fewer models to choose from, usually dearer, harder to inspect.

**The hard parts, in either design:**

**Speed.** In human conversation the gap between turns is a fraction of a second. Past
about a second, an assistant feels slow. A budget for the chain design:

| Step | Time |
|---|---|
| Notice the person stopped talking | 0.2s |
| Finish the transcript | 0.2s |
| Model starts answering | 0.4s |
| Voice starts playing | 0.2s |
| **Total** | **1.0s** |

Every piece has to stream; nothing can wait for the previous step to fully finish.

**Turn-taking.** How does it know you have finished and are not just pausing to think? Cut
in too early and it interrupts you; wait too long and it feels sluggish.

**Interruptions.** If you start talking while it is speaking, it must stop at once and listen.

**Speaking style.** Spoken answers must be short and plain. No lists, no headings.
"There are three options. First…" works. Reading out a table does not.

Open-source toolkits such as LiveKit Agents and Pipecat handle much of this plumbing.

---

## 9. Searching across modalities

Level 4 placed text on a "map of meaning" using embeddings. Some embedding models put
**images and text on the same map**. A photo of a dog on a beach lands near the sentence
"a dog playing on the beach".

That allows:

- **Text → image:** type "red running shoes" and find matching product photos, even if no
  photo was ever labelled with those words.
- **Image → image:** upload a photo of a chair and find similar chairs.
- **Image → text:** find the manual page that matches a photo of a part.

The best-known model of this kind is CLIP.

```python
from sentence_transformers import SentenceTransformer, util
from PIL import Image

model = SentenceTransformer("clip-ViT-B-32")
image_vectors = model.encode([Image.open("shoe1.jpg"), Image.open("chair.jpg")])
text_vector = model.encode("red running shoes")
print(util.cos_sim(text_vector, image_vectors))   # the shoe photo scores higher
```

Everything else from level 4 carries over: vector databases, filters, reranking, and
measuring recall with labelled test queries.

---

## 10. Choosing a multimodal model

The method from level 1 applies, with extra questions.

1. **What goes in and what must come out?** (Section 1.)
2. **Test on your own files.** Twenty of *your* invoices, *your* call recordings, *your*
   product photos. Public scores say little about them.
3. **Check the limits:** largest image size, number of images per request, longest audio.
4. **Work out the cost with real sizes.** Images and audio use many tokens.
5. **Check the speed** if a person is waiting.

*Example.* Pulling totals from supplier invoices. You test three models on 30 real
invoices, including blurry phone photos:

| Model | Totals correct | Cost per invoice | Time |
|---|---|---|---|
| Large | 29 / 30 | 1.8¢ | 4s |
| Medium | 28 / 30 | 0.5¢ | 2s |
| Small | 22 / 30 | 0.1¢ | 1s |

The medium model is the sensible pick, with low-confidence cases sent to a person.

---

## 11. Deep research and long tasks

"Deep research" is the name for agents that spend many minutes answering a hard question:
searching, reading dozens of sources, and writing a report with references.

It is levels 5 and 6 applied to mixed material:

- The sources are not just text: PDFs with charts, web pages with images, tables, videos.
- The agent must plan, keep notes, follow leads and know when it has enough.
- Context management matters a lot: 50 sources will not fit in one window, so helpers read
  and summarise (level 6, section 5).

*Example.* "Compare the battery life of the five best-selling laptops this year." The
agent searches, opens review pages, reads figures from comparison charts (vision), notes
where sources disagree, and writes a table with links.

What to watch for: sources that are unreliable, figures misread from charts, and claims
with no source behind them. Ask for a link beside every figure and spot-check some.

---

## 12. On-device and edge

"On-device" or "edge" means the model runs on the phone, laptop or gadget itself, not on a
distant server.

**Why:**

- **Privacy:** the photo or recording never leaves the device.
- **No connection needed:** works on a plane or in a basement.
- **Instant:** no network round trip.
- **No per-use fee.**

**Limits:**

- Devices have little memory and power, so the models are much smaller and less able.
- Running them drains the battery and warms the device.
- Updating means shipping a new model to every device.

*Examples:* live captions on a phone, dictation that works offline, a camera app that
recognises objects, translating a sign through the camera.

**A common hybrid:** do the simple, private, instant things on the device, and send harder
requests to a larger model in the cloud, with the user's agreement.

---

## Free resources

- CLIP paper: <https://arxiv.org/abs/2103.00020>
- Vision Transformer paper: <https://arxiv.org/abs/2010.11929>
- ColPali paper (visual document retrieval): <https://arxiv.org/abs/2407.01449>
- Whisper: <https://github.com/openai/whisper>
- Anthropic vision docs: <https://docs.anthropic.com/en/docs/build-with-claude/vision>
- Hugging Face computer vision course: <https://huggingface.co/learn/computer-vision-course>
- Hugging Face audio course: <https://huggingface.co/learn/audio-course>
- LiveKit Agents: <https://docs.livekit.io/agents/>
- Pipecat: <https://github.com/pipecat-ai/pipecat>

## Project: ship one multimodal build

Pick one.

**A. Question-answering over page images.** Index 200 PDF pages containing charts and
tables. Compare a text-extraction pipeline with sending page images, on 40 questions.

**B. A voice agent.** Speech in, speech out, with one tool. Measure the delay from the end
of your sentence to the start of its reply, and handle being interrupted.

**C. Cross-modal search.** Text-to-image search over 2,000 images, with labelled test
queries and recall@10.

For any of them: report accuracy on your test set, cost per request, and the three most
common failures.

## Self-check

1. Why does image size affect both cost and accuracy?
2. Name two things vision models do poorly.
3. When does sending a page image beat extracting its text?
4. What are the two ways to build a voice agent, and what does each trade away?
5. Where does the time go in a voice pipeline?
