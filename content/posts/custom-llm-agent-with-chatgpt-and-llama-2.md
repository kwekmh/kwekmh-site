---
title: "Custom LLM Agent With ChatGPT and LLaMA 2"
date: 2023-12-18T11:03:48+08:00
draft: false
---

[Langchain](https://github.com/langchain-ai/langchain) is one more of the popular agents or frameworks for Large Language Models (LLMs). However, it can be extremely complicated to use, and as it is relatively mature, it can abstract away significant efforts that go into developing an LLM agent.

For that purpose, [Infuser](https://github.com/kwekmh/infuser) was born. It was conceived for my learning; I wanted to build a (ReAct)[https://arxiv.org/abs/2210.03629] agent without using Langchain. I started with ChatGPT as it was easier to implement; all I really needed was an API key. The API even provides abstractions for system and user prompts, which was really useful!

**ReAct** attempts to combine reasoning and acting with LLMs. To summarise, a ReAct agent receives a question, thinks about what is necessary to answer that question, attempts to solicit observations from a list of possible actions, and repeats this process until an answer can be found. An example of an action can be searching Wikipedia for certain keywords, or running semantic search on a vector database that stores relevant information. The observation is the outcome of the action that is passed to the ReAct agent to process. The agent will then continue to think based on the observation, and decide if it needs further observations or can provide an answer. The details can be found in the paper referenced above.

With ChatGPT, all I needed was to communicate with the API by setting up the system prompts, asking the questions using user prompts and providing observations through user prompts as well. It was painless to setup, although the current approach can probably be improved significantly. It was a good first-cut attempt, though, and I was quickly able to get the entire ReAct flow working with ChatGPT.

When it came to LLaMA 2, it was significantly more challenging. Due to limitations of my hardware, I opted for the models with 7B parameters for experimentation and developing the ReAct framework. What worked for me was the model that was finetuned for chat, namely `llama-2-7b-chat`. I tried to use `llama-2-7b`, even at the end when the ReAct framework was working with the chat-finetuned model, but it did not return a response. I have not yet dived deeper into why that is so.

There are various options for loading the LLaMA2 models, including [llama.cpp](https://github.com/ggerganov/llama.cpp) and HuggingFace's [transformers](https://huggingface.co/docs/transformers/index) library. I opted for the latter as it implied more versatility in the ability to load other HuggingFace's models more easily, so I could reuse the same code for other models that were not LLaMA. I had previously quantized using `llama.cpp` the 13B model with 8-bit quantization, but as it was in `GGUF` format I was not able to load it into `transformers` casually. Hence, I opted for the 7B model so I could focus on implementing the framework and optimise the model selection later. I was also not able to use the quantization library that HuggingFace implemented for `transformers`, as I was on Windows and `bitsandbytes`, which was used for quantization, did not play well. Installing `bitsandbytes-windows` did not help. I will probably have to save the quantized model using HuggingFace's libraries on a Linux-based system and load it on my Windows machine at some point in the future.

The generated output also kept including my original prompt, and I was not sure if that was due to how my prompt was written, the use of the special tokens, or something else. I tried with a basic prompt with no system prompts, and the same happened. Eventually, I realised that in order to prevent that behaviour, I needed to pass in `return_full_text=False` when I used the pipeline from `transformers.pipeline()`. An example is as follows:

```
sequences = self._pipeline(
            final_prompt,
            do_sample=True,
            top_k=10,
            eos_token_id=self._tokenizer.eos_token_id,
            return_full_text=False,
            max_length=4096,)
```

Once the model setup was done, I had to start building the prompts for LLaMA 2. My initial attempts were not very successful, as I did not explicitly label the system prompts, and the model asked me to choose an action instead of choosing an action as it should. I did not realise that I had to use [special tokens to identify system prompts and for multi-turn conversations](https://huggingface.co/blog/llama2#how-to-prompt-llama-2). I eventually came upon this [discussion](https://discuss.huggingface.co/t/trying-to-understand-system-prompts-with-llama-2-and-transformers-interface/59016), which helped to pave the way ahead for me. I also referred to example implementations at ReAct, and the system prompt that I had was heavily inspired by https://til.simonwillison.net/llms/python-react-pattern. The final prompt that I had, including the actions and observations, was something along the lines of:

```
<s>[INST]<<SYS>>
You run in a loop of Thought, Action, PAUSE and Observation.
...
<</SYS>>Question: ...
</INST>
Action: wikipedia: ...
</s><s> <INST> Observation: ... [/INST]
```

When the model finds an answer, it will respond with `Answer: ...`. Interestingly enough, the model seems to return me an answer immediately, even when it does not have it, although it also provided an action and was asked to `PAUSE`. It provided its own observation that did not rely on external sources. Perhaps I need to refine the system prompt for LLaMA 2. The test cases are not extensive right now, and there are no promises that the project will work as designed, but I intend to improve on it incrementally, including adding unit and integration tests at some point.

All in all, this was a fun exercise in implementing a basic ReAct agent, and in developing a framework for building more LLM agents.