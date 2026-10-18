# Pretraining

**1. download and preprocess the internet**

> URL filtering => text extraction => language filtering => gopher filtering => minhash deduplication => C4 filters => custom filters => PII removal
- neural networks expect a finite 1D sequence of symbols
- the length of this sequence is significant to any neural network
- we don't want a very long seq of say 2 symbols rather it is better to have a shorter seq of more symbols
- how do we decr the length of these sequences => say we treat each 8 bit as a byte
- **byte-pair-encoding algorithm** is used to decrease the length of sequences in state of the art models (SOTA - means highest level of perf currently acheived on a specific benchmark)
- so we group commonly occurring symbol pair to something one

**2. Tokenization**

> **Tokenization** - the process of converting raw text to tokens

**How GPTs/LLMs perform tokenization**

**3. Neural Network training**
- take a variable length window of tokens randomly in an arbitrary range (not too long)
- this window of tokens that we feed to a neural nets is the context and neural nets does some work to output/predict the next token.
- say the NN gave probabilities of next probable tokens, but we see from the training data that the actual next token isn't having the highest probability from what the NN produced.
- so we need to iteratively update (adjust external configs) the NN somehow to ensure that the NN produces higher probability for the actual next token **this is called tuning a neural network**

**neural network internals**
- input seq tokens + parameters (weights) --> [a giant mathematical expression] --> output

- **inference** - generating new data from the trained model.
- **sampling** - a strategy used by any model to select its next token.

> **CRUX till here** - preprocessing roughly comprises of downloading data and tokenization now we have a token sequence. Now we start training our networks and once we're happy with a set of parameter (i.e, once the results we're getting are kinda desirable) then we start inference (i.e, we ask the network to give output for our new data, say ask it to make predictiions or decisions etc.)

> **Base Model** - a pretrained ML system that has learned general patterns by analyzing massive amount of unlabelled data

> **regurgitation** - it occurs when an LLM outputs training content verbatim(as it is) rather than generating a unique response. The model doesn't kinda creates something new rather it acts as a copy-paste machine outputing exact memorized blocks of text from its data

> **hallucination** - when a model confidently produces incorrect or fabricated outputs

> **Few-shot prompting** - a technique in AI where you provide a model with a handful of input-output examples inside the prompt so it can infer the task pattern and apply it to new queries. the few-shot prompting helps the model in in-context learning.


> **CRUX of a Base Model** => Base Model is kinda internet document simulator => it is stochastic/probabilistic => it can recite some training data verbatim(regurgitation)

# SFT Training (supervised learning)
- we train based on examples of conversation b/w user and assistant

- **human labeller** - a person who manually reviews and tags data so that it can be used to train or evaluate the models.

**Mitigating hallucinations**
1. approach 1
- use model integration to discover the model's knowledge and programmatically augment its training dataset with knowledge-based rufusals in cases where the model doesn't know.
2. approach 2
- using tools
- knowledge in parameters of a neural nets is kinda vague recollection (from past say 1 month) whereas the knowledge in the tokens of the context window is the working memory

- **knowledge of self** - if we don't train the model on questions like 'who are you' etc., all it does is it tries to make a statistically best guess
- the LLMs have no knowledge of self "out of the box"
- we can program a "sense of self" in 2 ways:
- 1. hardcoded convos around these topics
- 2. "system msg" that reminds the model at the begining of every convo about its identity

**models need tokens to think**
- the computation must be spread across the tokens so that the whole compute isn't relied on a single or a small portion of tokens
- doing all the mental arithematic in-memory is not good for models as they try to do it in a single forward pass, so they can't do the intermediate calculations and often turn up giving wrong answers.
- we spreading the compute across the tokens is better. we could also ask the model to use tools (say code => turn this problem into a code and then solve it) and turns up models are good at these tasks, so we get right answer

**no of `r`s in `strawberry`**
- models don't see characters. what they see are tokens so maybe the case it used to tell there are 2 `r`s in `strawberry`. similarly the models are not very good at spelling related tasks for ex: u ask it to give every 3rd character in a string and it might end up giving wrong answer as it doesn't see characters rather tokens instead.

# Reinforcement Learning
- we get an SFT (supervised Fine-Tuning) model as a result of supervised training
- now we perform post training (RL)
- here the model learns itself from many solutions it has generated and keeps correcting itself

**Deepseek-R1 - the RL paper**
- models use chains of thought and it increases the response length and also the accuracy of the problem solving
- model tries to find ways to solve a problem. it uses cognitive strategies to tackle a problem
--

## RLHF (RL from human feedback)
- models give different responses for unverfiable problems and these problems don't have one specific or correct answer. their answers can be opinionated
- RLHF is used in unverifiable these domains
- what we do is ask humans to evaluate those responses and we try to build a separate model (**reward model**) that would imitate the human's evaluation of those evaluation (for unverifiable problems)
- for example: tell me a joke about CPU, write a poem on RAM etc.
- the reward model takes an input prompt and a candidate response and output a number (score)
- the scores of the reward model might not be correct as per the actual human feedback so we try to tune the scores in different passes.

**upsides of RLHF**
- running RLHF in unverifiable domains empirically improves the model perf. possibly due to the `discriminator-generator` gap. in many cases it is easy to discriminate than to generate. eg: write a poem vs which of these 5 poems is the best

**donwnsides of RLHF**
- we're doing RL with respect to a lossy simulation of humans. it might be misleading.
- the model interprets the scoring as a game and tries to improve scores and might end up giving non-sense responses (adverserial inputs).

> **CRUX of all three stages** - exposition(background knowledge) is pretraining, problem+demonstrated soln is supervised finetuning, practice problems is reinforcement leraning
