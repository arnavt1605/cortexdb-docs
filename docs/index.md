# CortexDB

## Introduction

Cortex is a privacy first, locally hosted LLM that remembers your conversations over time. Built as a command line interface(CLI), it solves the "AI amnesia" problem by automatically extracting and deduplicating key facts from your chats.

Under the hood, it uses an AES-GCM encrypted SQLite vault and a two stage Retrieval Augmented Generation (RAG) pipeline to silently inject relevant memories into your future prompts. Everything runs 100% offline using your local Ollama models, ensuring your personal data never leaves your machine.

## Getting Started

### Prerequisites

- Python 3.8+ is required to run the local package

- Ollama: Download and install [Ollama](https://ollama.com/download) to use LLMs locally on your machine. Ensure that the Ollama app is running in the background.

- A Local Model: Pull the default model by running ```ollama pull llama3.1:8b``` in your terminal

### Installation

You can install the package globally directly from the repository. Open your terminal and run:

```py title="Terminal"
pip install --upgrade git+https://github.com/arnavt1605/Cortex.git
```

## Quickstart Guide

1. Launch the package. Open your terminal and type the global command:

    ```py title="Terminal"
    cortexdb -m <model-name>
    # Or just use cortexdb to use the default llama3.1:8b model
    # Make sure this model is installed in your system first
    ```

2. Teach the model something new. A simple example using LLaMA 3.1 is given below:

    ```py title="Example"
    # Users types in a command to view all the previously stored memories by the LLM
    You: show memories


    # Currently the model has no stored memory about the user
    Your Stored Memories
    Your menory is currently empty


    # User enters a prompt
    You: i like python programming


    # The LLM's response
    AI: Python is a fantastic language to learn and work with, especially for beginners and experienced programmers alike.
    What would you like to do with Python? Here are some ideas:

    1.Learn the basics  If you're new to Python, I can help you understand the fundamental concepts, such as data types, lo
    ops, functions, and object-oriented programming.
    2. Work on projects : Do you have a project in mind that you'd like to implement using Python? Perhaps something with data analysis, machine learning, web development, or automation?
    3. Get help with a specific problem : Are you stuck on a particular issue or error message while working on a Python project? Feel free to share the details, and I'll do my best to assist you.
    4. Explore popular libraries and frameworks : We could discuss popular libraries like NumPy, pandas, Flask, Django, or Scikit-learn, which can help with data science, web development, and nore.
    What's your current Python-related interest or goal?


    # The user terminates the session
    You: quit

    Closing session. Analyzing chat for permanent facts ...
    Learned 2 new things about you!


    # In a new session the user asks the following question
    You: Which programming Language do I like?
    [System: silently injected 2 memories into context]


    # The LLM's reply after looking through the saved memories
    AI: Based on what I know, it's Python! You've expressed interest in Python programming and have some experience with the language, which suggests that Python is your preferred choice. Is there anything specifi
    c you'd like to work on or learn in Python?

    ```
