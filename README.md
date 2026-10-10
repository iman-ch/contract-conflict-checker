# Contract Conflict Checker
**Model:** [Hugging Face](https://huggingface.co/imanxch/distilbert-contractnli) · **Demo:** coming soon

Compare an incoming contract against your standard terms, and flag conflicts on payment terms, liability caps, termination notice, and IP ownership. For small businesses and freelancers who sign vendor and client contracts without a lawyer.

**Status: in progress** (*see [roadmap](#roadmap)*)

## Problem

Small businesses and freelancers often sign contracts without legal review. An incoming contract might change important terms or include fine print. These conflicts can be easy to miss in a long document.

## How it works
This tool is built in layers:
```
Pretrained language model (DistilBERT)
  -> learns general contradiction (MultiNLI)      <- Notebooks 1–2 (DONE)
  -> adapts to contract language (ContractNLI)    <- Notebook 3
  -> analyzed for weak spots                      <- Notebook 4
  -> wrapped in a pipeline*                       <- Notebook 5
  -> deployed as a web app                        <- Planned
```
The final pipeline will*
1. split both contracts into clauses
2. label each clause type (payment, liability, termination, IP)
3. match clauses of the same type across two contracts
4. check each pair: fine-tuned NLI model for wording conflicts, plus rules for numbers (day, dollar amount)
5. output a plain English conflict report

## Results so far
### Notebook 1: Baseline
DistilBERT fine-tuned with a PyTorch training loop on 20k MultiNLI sentence pairs. Validation accuracy after 2 epochs was 71.5%.

### Notebook 2: Experiments
Training was optimized (dynamic padding, mixed precision), and controlled experiments were tracked in MLflow.
| Run | Learning rate | Max length | Training pairs | Val accuracy | Contradiction F1 | Minutes |
| --- | --- | --- | --- | --- | --- | --- |
| **Best** | **3e-5** | **128** | **100K** | **77.5%** | **0.787** | **10.3** |
| | 3e-5 | 128 | 50K | 75.5% | 0.763 | 5.1 |
| | 2e-5 | 128 | 50K | 75.2% | 0.756 | 5.3 |
| | 3e-5 | 256 | 50K | 74.7% | 0.756 | 5.9 |
| | 5e-5 | 128 | 50K | 74.7% | 0.755 | 5.1 |
| | 2e-5 | 128 | 20K | 70.8% | 0.710 | 2.2 |

*Findings:*
- Training data mattered most: 20K -> 50K -> 100K pairs raised accuracy from 70.8% -> 75.5% -> 77.5%.
- Learning rate mattered a little: 3e-5 was best; 5e-5 was too large for fine-tuning.
- Longer sequences added nothing: few MultiNLI pairs exceed 128 tokens.
- The optimized pipeline is about 4–5× faster: the 20K run dropped from ~10 to 2.2 minutes with matching accuracy.

### Notebook 3: Contract NLI Adaptation
The best general model was tested on ContractNLI (real NDAs) before and after fine-tuning on contract language.

| Model | Test accuracy | F1 entailment | F1 neutral | F1 contradiction |
| --- | --- | --- | --- | --- |
| General model (zero-shot) | 40.5% | 0.454 | 0.428 | 0.271 |
| **Fine-tuned on ContractNLI** | **82.4%** | **0.837** | **0.826** | **0.760** |

*Findings:*
- Fine-tuning doubled accuracy and almost tripled contradiction F1.
- The general model over-flagged contradictions: contract hypotheses often contain negations ("shall not…"), a shortcut it had learned from MultiNLI.
- Class weights were tested for the imbalanced data (contradiction is 12% of pairs) but didn't improve contradiction F1, so the simpler model was kept.

**Model:** [imanxch/distilbert-contractnli on Hugging Face](https://huggingface.co/imanxch/distilbert-contractnli)

## Roadmap
- [x] **Notebook 1:** baseline model with a hand-written training loop
- [x] **Notebook 2:** optimized training and tracked experiments
- [x] **Notebook 3:** adapt the model to contract language (ContractNLI), before/after comparison
- [ ] **Notebook 4:** error analysis: where and why the model fails on contracts
- [ ] **Notebook 5:** clause-by-clause contract comparison pipeline, evaluated on contract pairs with planted conflicts
- [ ] **Web app:** Gradio demo on Hugging Face Spaces

## Tech Stack
- Modelling: PyTorch, Hugging Face Transformers (DistilBERT)
- Data: Hugging Face Datasets (MultiNLI, ContractNLI, CUAD)
- Experiment tracking: MLflow
- Evaluation: scikit-learn (accuracy, per-class F1, confusion matrices)
- Training environment: Google Colab (T4 GPU)
- Planned deployment: Gradio, Hugging Face Spaces

## How to Run
Notebooks run top-to-bottom in Colab with a GPU runtime (Runtime → Change runtime type → T4 GPU). Intermediate models and MLflow logs are saved to Google Drive; update `PROJ_DIR` in each setup cell to your own folder. The final model is published on [Hugging Face](https://huggingface.co/imanxch/distilbert-contractnli).

## LIMITATIONS
- This will not output legal advice. It will flag potential conflicts for further review, **not replace a lawyer**.
