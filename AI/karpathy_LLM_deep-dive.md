# Pretraining

1. **download and preprocess the internet**

> URL filtering => text extraction => language filtering => gopher filtering => minhash deduplication => C4 filters => custom filters => PII removal
- neural networks expect a finite 1D sequence of symbols
- the length of this sequence is significant to any neural network
- we don't want a very long seq of say 2 symbols rather it is better to have a shorter seq of more symbols
- how do we decr the length of these sequences => say we treat each 8 bit as a byte
- **byte-pair-encoding algorithm** is used to decrease the length of sequences in state of the art models (SOTA - means highest level of perf currently acheived on a specific benchmark)
- so we group commonly occurring symbol pair to something one

2. **Tokenization**

> **Tokenization** - the process of converting raw text to tokens

**How GPTs/LLMs perform tokenization**

3. **Neural Network training**
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