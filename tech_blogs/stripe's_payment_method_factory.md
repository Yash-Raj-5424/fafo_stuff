```
https://stripe.dev/blog/stripes-payment-method-factory-orchestrating-agents-for-repeated-custom-integrations
```

- they built their payment method factory from over 100 reusable prompts, hardened by impl runs and kept them current by feeding back each run's learnings
- each payment method does: make a payment, issue a refund, handl a dispute
- these diff payment method have diff behaviors and diff API shapes but to build these integration took time as teams had to start from scratch everytime even though the patterns were familiar
- why not use LLMs to build these integrations an use the same prompt for every future integration ?

**Hardened reusable `one-shot` prompts**
- even in a sucessful `one-shot` prompts, the agents perform unnecessary investigations or repeatedly attempt to fix failing tests -- evaluating and refining these prompts could give faster results plus low cost, more confidence

**writing good `one-shot` prompt**
- they used earlier implementation as base for their prompt, gave a template for structuring the prompt then refined the output by hand. they kept the template payment-method agnostic for reusability
- an agent follows the prompt to build a payment integration while the other acts as observer in parallel and kinda monitors all the work of impl agent like its transcript, tool failure, CI results etc. and an engineer takes the observer's report and refines the prompt further

**keeping prompts up to date**
- a payment method factory collects these artifacts while it's working on an integration: learning logs from impl agents, decision logs, transcript of sessions, review comments by engineers on a PR -- taking these improvements are suggested that could be implemented by a single click using `minion`

**Orchestrating the collection of prompts**
- two phases - 1. create a plan (seq of discrete steps each linked to one of the prompts -- each step should map to one PR), ask the engineer for context and export it to a shared knowledge base and impl agent can refer this.   2. plan execution with impl subagents handling each step. here an orchestrator is needed to assign right task to a subagent.

**Confidence**
- gave agents the tools to test their integrations autonomously.
- a sample scenario looks like: factory creates test payments and runs diff scenarios, snapshots responses and creates immutable contracts that are used throughout the stack -- all of this is presented to the engineer
- agents should hv tools to interact with every part of the system [improvement] 
