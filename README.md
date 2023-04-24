# Outlines

Build _reliable_ workflow based on interactions with large language models.

## Prompt management

Outlines makes it easier to write and manage prompts by encapsulating templates
inside "template functions". These functions make it possible to neatly separate
the prompt logic from the general program logic; they can be imported from other
modules and libraries.

Template functions use the Jinja2 templating engine to help build complex
prompts (like few-shot examples) in a concise manner:

``` python
import outlines.text as text


@text.prompt
def few_shot_examples(question, examples):
    """Something something

    {% for example in examples %}
    EXAMPLE: {{ example }}
    {% endfor %}

    QUESTION: {{question}}
    Let's think step by step.

    """
```

Functions can also be _partially evaluated_ just like any function, which can be useful when building agents:

``` python
import functools as ft
import outlines.text as text

@text.prompt
def my_agent(name, goals):
    """Your name is {{name}}.

    GOALS:
    {% for goal in goals %}
    {{loop.counter}}. {{goal}}
    {% endfor %}
    """

jarvis = ft.partial(my_agent, "JARVIS")
```

### Tools

Prior work has shown that we can teach language models to call external functions to get additional informations or perform tasks, by encoding the functions' description in the prompt. To avoid duplicating information between the function definition and the description passed to the prompt we introduce a `@outlines.tool` decorator which automatically extracts the needed information from the function's definition:


``` python
from typing import Callable, List
import outlines
import outlines.text as text


@outlines.tool
def google_search(query: str):
    """Google Search"""
    pass


@outlines.tool
def wikipedia_search(query: str):
    """Wikipedia Search"""
    pass


@text.prompt
def my_commands(tools: List[Callable]):
    """AVAILABLE COMMANDS:

    {% for tool in tools %}
    {{loop.counter}}. {{tool.name}}, {{tool.description}}, args: {{tool.signature}}
    {% endfor %}
    """

prompt = my_commands([google_search, wikipedia_search])
```

## Text completion

Prompts are often attached to a given model and specific settings, but this can
be hard to find in codebases. Following this logic, we abstract calls to any
model that takes prompts as an input by decorating template functions:

``` python
import outlines.text as text


@text.completion("openai/text-davinci-003", stop_at=["\n"])
def few_shot_examples(question, examples):
    """You are a question answering AI.

    {% for example in examples %}
    QUESTION: {{example.question}}
    ANSWER: {{example.answer}}
    {% endfor %}

    QUESTION: {{question}}
    Let's think step by step.

    """

result, completed = few_shot_examples(question, examples)
```

## Image generation

A similar syntax can be used with image generation models:

``` python
import outlines.image as image

@image.generation("hf/stabilityai/stable-diffusion-2.1")
def generate(subject, location):
   "A photo of a {{subject}} riding a horse in {{location}}."
```

## Natural language functions

Large language models can be prompted so their output can be parsed into a data structure that can be manipulated by programming languages. The combination prompt + model call + output parser can thus be thought as a "natural language" function.

``` python
import json
import outlines.text as text
import outlines.models as models


@text.prompt
def prime_numbers(n: int):
    """Return a list that contains all prime numbers between 1 and {{n}}.

    The output must be parsable as a Python list.
    """

def parse(result):
    return json.loads(result)

get_prime_numbers = text.function(
   models.text_completion.openai("gpt-3.5-turbo"),
   prime_numbers,
   parse
)

get_prime_numbers(10)
# [2, 3, 5, 7]

```

For more complex outputs one can pass a Pydantic model to `text.function`, which will be used to parse the output:

``` python
from pydantic import BaseModel
import outlines.text as text

class Joke(BaseModel):
    joke: str
    explanation: str


@text.prompt
def joke_ppt(n: int):
    """Tell a joke and explain why the joke is funny.

    RESPONSE FORMAT:
    {
       "joke": "The joke"
       "explanation": "The explanation of why the joke is funny"
    }
    """

tell_a_joke = text.function(
   models.text_completion.openai("gpt-3.5-turbo"),
   joke_ppt,
   Joke
)

get_prime_numbers(10)
# [2, 3, 5, 7]
```
