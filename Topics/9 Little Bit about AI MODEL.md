---
Title:
Type:
Topic:
tags:
ee: "[[/IMAGES/BUS.png]]"
---
#  **"I have a model. Now what? How do I actually use it?"**

There are **three common ways** to use a model.
## Method 1 (Easiest): Ollama

Suppose you want to use **Llama 3**.
### Step 1: Install Ollama

`Download Ollama` Install it.

### Step 2: Download the model

`ollama pull llama3` What happens?

```
Internet
    │
    ▼
Downloads model
    │
    ▼
Stores on your PC
```

The model is now on your computer.

### Step 3: Run it

```
ollama run llama3
```

Output

```
>> Hello
Hi! How can I help?
```

Done. No Python. No coding.

## But what actually happened?

```
You
↓
Ollama
↓
Loads model into RAM
↓
Uses CPU/GPU
↓
Model generates answer
↓
Shows answer
```

## Method 2 (Most Common): Python + Ollama

Suppose you want your own application. Instead of typing in the terminal, Python talks to Ollama.

```
Python
↓
Ollama
↓
Model
↓
Answer
↓
Python prints answer
```

Install library `pip install ollama`
Python code

```
from ollama import chat

response = chat(
    model='llama3',
    messages=[
        {
            'role': 'user',
            'content': 'Explain Kubernetes like I am 5.'
        }
    ]
)

print(response.message.content)
```

Output `Imagine Kubernetes is a school principal...` That's literally it.
Your Python program never "understands AI." It simply sends a request to Ollama, which runs the model and returns the response.
## Method 3: Directly from Hugging Face

Suppose you found `meta-llama/Llama-3-8B` on Hugging Face. You don't use it directly. You need a library. Usually `transformers`
Install `pip install transformers torch`
Then

```
from transformers import pipeline

chat = pipeline(
    "text-generation",
    model="gpt2"
)

result = chat("Hello, my name is", max_new_tokens=20)

print(result[0]["generated_text"])
```

Notice

```
pipeline()
↓
Downloads model
↓
Loads model
↓
Runs inference
```

The `pipeline` function hides a lot of complexity.
## Real AI Project Structure

Suppose GitHub contains

```
AI-Chatbot/
app.py
requirements.txt
README.md
config.py
```

README says

```
Install Python
Install requirements
Install Ollama
Download llama3
```

You do `git clone https://github.com/example/chatbot`
Then `cd chatbot`
Install libraries `pip install -r requirements.txt`
Run `python app.py` The app connects to Ollama.

## What happens inside app.py?

```
User types
↓
Python receives text
↓
Python sends text to Ollama
↓
Ollama loads model
↓
Model thinks
↓
Answer returned
↓
Python displays answer
```

## Example Architecture

```
             You
              │
              ▼
        Python Application
              │
      HTTP/API Request
              │
              ▼
          Ollama Server
              │
              ▼
      Llama 3 Model (.gguf)
              │
              ▼
        CPU / GPU computes
              │
              ▼
         Generated Answer
              │
              ▼
          Python prints
```
## If you use Hugging Face instead

```
Python
↓
Transformers Library
↓
Downloads model
↓
Loads weights
↓
Uses PyTorch
↓
CPU/GPU
↓
Answer
```

## Why do GitHub projects have requirements.txt?

Example

```
transformers
torch
fastapi
gradio
numpy
```

These are Python libraries your application depends on. Install them `pip install -r requirements.txt`

## What if the project says

```
Download model from Hugging Face
```

Example

```
README
↓
Download
mistralai/Mistral-7B
```

You either:

- let the `transformers` library download it automatically the first time your code runs, or
- manually download the model files if the project requires that.

## Complete Life Cycle

```
                 AI Company
                     │
      Trains model using huge datasets
                     │
                     ▼
              Uploads to Hugging Face
                     │
     You download model (or Ollama does)
                     │
                     ▼
            Stored on your computer
                     │
                     ▼
       Runtime loads model into memory
      (Ollama / Transformers / vLLM)
                     │
                     ▼
         CPU/GPU performs inference
                     │
                     ▼
         Your application gets answer
                     │
                     ▼
              User sees response
```
## A real example

Let's build the simplest chatbot.
### 1. Install Ollama

```
# Install from the Ollama website
```
### 2. Download a model

```
ollama pull llama3
```
### 3. Install Python library

```
pip install ollama
```
### 4. Create `chat.py`

```
from ollama import chat

while True:
    prompt = input("You: ")

    if prompt.lower() == "exit":
        break

    response = chat(
        model="llama3",
        messages=[
            {
                "role": "user",
                "content": prompt
            }
        ]
    )

    print("AI:", response.message.content)
```

### 5. Run it

```
python chat.py
```

Example:

```
You: What is Docker?
AI: Docker is a platform that packages applications...
```

## The one thing most beginners misunderstand

Many people think:

```
Python
↓
AI happens
```

That's **not** what's happening. The real flow is:

```
Python Code
     │
     ▼
AI Runtime (Ollama / Transformers / vLLM)
     │
     ▼
Model Files (.gguf, .safetensors, etc.)
     │
     ▼
CPU / GPU performs the maths
     │
     ▼
Generated Output
     │
     ▼
Back to Python
```

Python is usually just the **orchestrator**. The model contains the learned knowledge, and the runtime knows how to execute it efficiently.

# **Language of AI**.

Think of AI like cooking.

```
Chef = Model
Kitchen = Runtime (Ollama)
Recipe = Prompt
Ingredients = Tokens
Experience = Parameters
Cooking Time = Steps
```

## 1. Parameters (The Brain Size)

This is the **most important one**. People say

```
7B
8B
13B
70B
405B
```

What is "B"?  **B = Billion parameters**
A parameter is simply a number the model learned during training.
Imagine teaching a child. At first `Dog? "I don't know."`
After learning `Dog? "I know."`
How did the child remember? By changing billions of tiny values inside the brain. Those values are parameters.
Example

```
Model
↓
7 Billion learned numbers
```

Another `70 Billion learned numbers`
More parameters usually means

- Better reasoning
- Better writing
- Better coding
- Better memory of patterns

But also

- More RAM
- More VRAM
- Slower
- Bigger download

Example

```
Tiny model 1B ≈ 1 GB
Medium 7B ≈ 4-8 GB
Large 70B 40+ GB
```

Think of parameters like this:

```
1B model = Primary school student 
7B = College student
70B = Experienced engineer
```

Not always true, but good enough for intuition.

## 2. Tokens

People think AI reads words. It doesn't. It reads **tokens**.
Example
Sentence `I love pizza.`
Might become

```
"I"
" love"
" pizza"
"."
```

Four tokens.
Another `Kubernetes`
Might become

```
"Kuber"
"netes"
```

Two tokens.
Numbers

```
123456789
```

may become several tokens too.
Imagine LEGO.
Sentence `I love programming.`
AI sees

```
🧱
🧱
🧱
🧱
🧱
```

Those LEGO pieces are tokens.

### Why are tokens important?

Suppose model supports `8,000 tokens`
That means

```
Question
+
Previous Chat
+
Answer
must fit within 8000 tokens
```

If too long Old,  conversation gets forgotten.
## 3. Context Window

Closely related to tokens.
Example

```
Model
↓
Context Window
128K tokens
```

Means AI can remember

```
Entire book
Large PDF
Huge conversation
```

during one chat.
Small context `4K tokens` Means `Few pages only`

## 4. Temperature

One of the easiest. Temperature controls creativity.
Imagine asking `Write a story.`
Temperature = 0

```
Very predictable.
Very factual.
Always similar answers.
```

Temperature = 1

```
Creative.
Funny.
Different every time.
```

Temperature = 2

```
Crazy.
Random.
Sometimes nonsense.
```

Think

```
0
↓
Robot

1
↓
Normal Human

2
↓
Crazy Poet
```

### Example
 
Ask `Name a blue animal.`
Temperature 0 `Blue whale`
Every time.
Temperature 1

```
Blue whale
Blue poison dart frog
Bluebird
```

Temperature 2 `Cosmic blue dragon riding a rainbow.`

## 5. Top-p

This is another randomness setting. Imagine the AI has these possible next words:

```
eat    50%
drink  30%
sleep  15%
dance   5%
```

If `top_p = 1.0`, it can consider all of them.
If `top_p = 0.8`, it only considers the words whose probabilities add up to 80%:

```
eat
drink
```

It ignores the less likely options. So lower `top_p` usually means more focused answers.
## 6. Top-k

Even simpler.
Imagine
100 possible next words. Top-k = 10 Means Only consider `Top 10` Ignore `Other 90` 
Again, it's about controlling randomness.

## 7. Max Tokens

Suppose `Max Tokens = 100`
Model stops after roughly 100 generated tokens.
Example
Ask `Explain Linux.`
Instead of `5 pages` It stops around 100 tokens.
## 8. Steps

This one confuses everyone because it means **different things** depending on the type of AI.

### For Chat Models (Llama, GPT, Qwen)

Normally you **don't see "steps."** The model generates

```
Token 1
↓
Token 2
↓
Token 3
↓
Token 4
```

One token at a time until it finishes.

### For Image Models (Stable Diffusion, Flux, SDXL)

Steps are very important.
Imagine drawing.
Step 1 `Random noise`
Step 5 `Looks like a blob`
Step 20 `Looks like a cat`
Step 40 `Beautiful cat`
Each step refines the image.
More steps

```
Better quality
Longer generation time
```

## 9. Turbo Models

Turbo usually means

```
Optimized
↓
Faster
↓
Cheaper
↓
Sometimes slightly lower quality
```

Think

```
Sports Car
↓
Fast
↓
Slightly less comfortable
```

Example

```
Normal model
↓
10 seconds
```

```
Turbo
↓
2 seconds
```


Different companies use the word "Turbo" differently. It's a product name, not a universal AI setting.

## 10. Quantization (Q4, Q5, Q8)

You'll often see:

```
Q2
Q4
Q5
Q8
```

These are **compressed versions** of the same model.
Think of image quality.

```
Original Photo
↓
100 MB

Compressed
↓
10 MB
```

Still looks good.
Same with AI.
Original

```
16-bit
Very accurate
```

Q4

```
Smaller
Uses less RAM
Slightly lower quality
```

Q8

```
Bigger
Better quality
More RAM
```

## Putting it all together

Imagine you run:

```
response = model.generate(
    prompt="Tell me a pirate story",
    temperature=0.8,
    max_tokens=200,
)
```

What happens?

```
Your Prompt
      │
      ▼
Model (7B parameters)
      │
Reads your prompt as tokens
      │
Uses its learned parameters
      │
Chooses the next token
(temperature/top-p/top-k influence the choice)
      │
Repeats token by token
      │
Stops after 200 tokens
      │
Returns the story
```

## Quick Cheat Sheet

|Term|Simple meaning|
|---|---|
|**Parameters**|The model's learned knowledge (brain size).|
|**Tokens**|The small pieces of text the model reads and writes.|
|**Context Window**|How many tokens the model can keep in mind at once.|
|**Temperature**|How creative or random the output is.|
|**Top-p**|Limits choices to the most likely cumulative probability.|
|**Top-k**|Limits choices to the top _k_ candidate tokens.|
|**Max Tokens**|Maximum length of the generated response.|
|**Steps**|Iterations used mainly by image generation models to refine an image.|
|**Turbo**|A vendor-specific, optimized model variant that's usually faster.|
|**Quantization (Q4/Q5/Q8)**|Compression level that trades memory usage for quality.|

