# references_functionHelper

Paste this before the function or concept you want explained.

## Ground rules (read before explaining anything)

1. **Load the scope first.** Before explaining any function, check that the full scope is available: all in-scope contracts, plus the README/docs if they were provided. If the contracts or full scope are missing, say so and ask for them. Do not explain from a partial view.
2. **Assume nothing.** Only state things that are actually present in the provided contracts, files, or README. Do not fill gaps from memory, from how "similar protocols usually work", or from what a name suggests.
3. **Read before you speak.** If a function calls another function, library, modifier, interface, or external contract, open and read it in the scope first. If it is not in the scope, say "not in scope, can't confirm" rather than guessing what it does.
4. **Mark every claim's source.** For each claim, be able to point to where it comes from (file, contract, function, or README line). If you can't, don't claim it.
5. **Separate fact from unknown.** End with a short "Can't confirm" list: anything the explanation depends on that is not visible in the scope (external calls, off-chain behavior, config/admin values, deployed state).

## Paste-in prompt

```
Before explaining anything:
- Confirm which files/contracts/README are in scope. If the contracts or full scope are not provided, stop and tell me what is missing.
- Read every contract, library, and interface the code below touches. Do not assume anything about code you have not read.
- Only talk about what is actually present in the scope. If something is not in the scope, say "not in scope, can't confirm" instead of guessing.

Then explain the code/concept below in simple, plain language, the way you'd explain it to a smart person who doesn't know this protocol.

Rules:
1. Start with one line: what it does.
2. Say why it exists, from the point of view of the person or contract using it
   (who calls it, what they hand over, what they get back).
3. Give a short step-by-step of what happens, in everyday words,
   using real function and variable names only where they help.
4. Add one tiny concrete example with simple numbers.
5. End with the net effect: who gains, who loses, and what changes in the system.
6. No jargon without a one-phrase definition. Short paragraphs. No walls of text.
7. Check every claim against the code I gave you. If you can't tell something from the code, say so instead of guessing.
8. Finish with a "Can't confirm" list: anything that depends on code, config, or state that is not in the scope.

Explain this:
[paste function / contract / concept here]
```

## Even shorter version

```
First confirm the full scope (contracts + README) is provided; if not, tell me what's missing. Read everything the code touches and assume nothing. Then explain this in plain words: what it does, why it exists, one simple example, and the net effect. Only state what is present in the scope, and list what you can't confirm.

[paste here]
```

## Why this works

- **Ground rules 1-3** stop the explanation from starting before the real code has been read, so nothing is built on a guess.
- **Rule 2** gives the "who hands over what, who gets what" framing, like the reUSD redemption explanation.
- **Rule 4** gives the concrete example, like the "if reUSD trades below $1" case.
- **Rule 5** gives the net-effect line at the end.
- **Rule 7 and 8** stop guessing and surface what is unverified, which matters for audit work.

## Depth control

If you want a particular depth, add one of these to the prompt:

- "keep it under 120 words"
- "go deeper on the accounting"
