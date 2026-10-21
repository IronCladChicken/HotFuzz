# HotFuzz

Automated black-box red-teaming of LLM safety guardrails.

Codebase for my MSc Cyber Security dissertation, *LLM Guardrail Subversion through Automatic Adversarial Prompt Generation and Obfuscation* (University of the West of England, 2024).

---

## What this project does

Language models are trained to refuse harmful requests. Jailbreak attacks are prompts written to get past that training. Testing for them by hand is slow, and it does not scale.

HotFuzz automates the testing. It needs only black-box access to the target model, and it runs on consumer hardware. For each test question it works through the same loop:

1. An **attack model** writes a prompt designed to get past the target's guardrails.
2. The same model **mutates** that prompt, using one of six rewriting methods.
3. The prompt is sent to the **target model**.
4. A **judge model** scores the reply from 1 to 10, according to how far the guardrails were bypassed.

Every response was also reviewed and categorised by hand, so the automatic judge could be checked against a human.

The approach combines three earlier pieces of work:
- prompt generation from PAIR (Chao et al.)
- a combined attacker and judge (Ramesh et al.)
- template mutation from GPTFuzzer (Yu et al.)

As far as I could find, the study is also the first to compare abliterated and uncensored models with standard models as attack-prompt generators.

---

## Findings

- **Current open-source models can still be jailbroken.** Where results could be compared with published work, HotFuzz matched, but did not exceed, the success rates of related tools.
- **Standard models made better attackers than abliterated ones.** Mistral 7B had the highest attack success rate (21.4%), almost double that of the abliterated Llama 3.
- **Mutation can recover failed prompts.** Mistral turned a failed prompt into a successful one 21.3% of the time. Of the mutation methods, crossover was the most effective (17.5%).
- **Automatic judging is unreliable.** The LLM judges disagreed with the human review on 60–65% of responses. Automated success scores need human checking.

Llama 2 was the hardest target to jailbreak against every attack model.

---

## Models tested

All models were run locally through [Ollama](https://ollama.com).

| Model | Attacker | Target |
|---|---|---|
| Mistral 7B | ✓ | ✓ |
| Llama 2 7B | ✓ | ✓ |
| Llama 3 8B (abliterated) | ✓ | |
| Llama 3 8B (uncensored) | ✓ | |
| Gemma 2 9B | | ✓ |
| Phi-3 3.8B | | ✓ |
| Vicuna 7B | ✓ | ✓ |

The Vicuna 7B results were left out of the analysis, because the model returned short, incomplete answers. The uncensored Llama 3 results were left out for lack of time, but its raw results are in `test_results/`.

---

## Repository layout

| Files | Purpose |
|---|---|
| `hotfuzz.py` | Entry point. Runs the full test loop |
| `global_config.py` | Chooses the generation method and which attack stages run |
| `llm.py`, `llm_manager.py` | Model selection and connection (Ollama, or an OpenAI-compatible API) |
| `llm_prompt_generator.py`, `llm_gptfuzz.py`, `fuzzer_gtpfuzz.py` | Prompt generation and mutation |
| `llm_attack_judge.py`, `llm_offtopic_judge.py` | Automatic scoring of target replies |
| `obfuscater.py` | Text-level obfuscation of prompts |
| `system_prompt_*.py`, `template_prompt_*.py` | System prompts and the template library |
| `logger_*.py` | CSV logs of every prompt, reply and score |
| `test_results/` | Raw results for each attack model (zipped) |

---

## Setup

1. Install [Ollama](https://ollama.com), then pull the models you want to test, for example `ollama pull mistral`.
2. Create a Python environment, for example with Miniconda: `conda create -n hot_fuzz python=3.11`.
3. Install the dependencies:
   ```
   pip install openai nltk eng_to_ipa emojis
   ```
4. Download the NLTK data:
   ```python
   import nltk
   for pkg in ["punkt", "averaged_perceptron_tagger", "gutenberg", "wordnet", "omw-1.4"]:
       nltk.download(pkg)
   ```

## Configuration

**Models.** In `llm.py`, set `attack_llm` and `defense_llm`. The prompt generator, the mutator, the two judges and the target can each be set individually. External models (with an `OPENAI_API_KEY`) can be configured here too.

**Test approach.** In `global_config.py`:

```python
# Prompt generator: PAIR, TAP, template, GPTFuzz or FuzzLLM
system_prompt_manager.type = system_prompt_manager.prompt_type.PAIR

# PAIR/TAP: number of templates to loop through (max 3; 0 = in-code template only)
system_prompt_manager.system_prompts.prompt_generator.loop_max = 0

# PAIR/TAP: number of times a prompt is returned for revision
system_prompt_manager.system_prompts.prompt_generator.attempt_max = 4

# GPTFuzz: use all 70 templates (True) or the refined 25 (False)
template_prompt_gptfuzz.use_full = False

# Attack an external model instead of a local one
llm_manager.attack_external_target = False

# Which attack stages run, and in what order
attacking_state.attack_list = [...]
```

## Running

```
python hotfuzz.py
```

The results are written as timestamped CSV files in the working directory.

---

## Responsible use

HotFuzz is a research tool for testing the safety of models you are authorised to test. All the experiments here were run against locally hosted open-source models.

---

## Citation

```
Lewis, S. (2024). LLM Guardrail Subversion through Automatic Adversarial Prompt
Generation and Obfuscation. MSc dissertation, University of the West of England.
```

Licensed under Apache-2.0.
