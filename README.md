# claude-cboms
A public record of using Claude Code to evaluate cryptographic packages and PQC in open source repositories, edited by humans.  Read with caution.

## Prompt
The prompt used for most of these files was:
```
Please analyze the contents of this folder and (1) list all locations where cryptography is unsed in this code or dependencies and (2) identify whether these functions or packages are quantum-safe.  Please keep your answer as concise as possible without missing packages.
```
with the exception of keylime.md, for which I also requested an action plan.  I found this to be too verbose and providing bad recommendations, but I have left it for posterity.

### ! Warning !
Most of this output is unverified.  As I find errors, I will manually correct them.
