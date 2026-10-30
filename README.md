# When does ML trust evidence expire? A small watermark pilot

A model owner releases a model and keeps two pieces of evidence about it:

* **Hash evidence:** a SHA-256 hash of all model parameters (weights and biases). The owner keeps the hash of the original model. If even one parameter changes slightly, the hash of the changed model no longer matches the original one. It answers "is this exactly the model I released?"
* **Watermark evidence:** a 100-image trigger-set watermark [1], where the model must return the owner's secret label for at least 50 of the 100 trigger images for the ownership claim to hold. It answers "is this still my model?"

Someone then fine-tunes or prunes the model. **Which piece of evidence still says something true about the changed model?**

This is a first, small-scale step towards studying how long ML property evidence (hashes, watermarks, attestations) stays valid after a model is transformed.

## Setup

* MNIST. The owner trains on 50,000 images; 10,000 other images play the role of someone else's data.
* Small CNN (2 conv + 2 linear layers). The watermarked model M0 is trained with the trigger set mixed in.
* Fine-tuning on the other data with lr = 1e-4 and lr = 1e-3, re-checked after each of 5 epochs.
* Global L1 magnitude pruning at 20, 40, 60, 80, 90 and 95% of weights.
* False-claim check, motivated by [2]: 3 models trained independently without the triggers.

## Results

![Watermark and test accuracy under fine-tuning and pruning](results.png)

| Group | Models | Hash matches M0 | Ownership claim holds |
|---|---|---|---|
| Original M0 | 1 | yes | yes (watermark 1.00) |
| Fine-tune, lr = 1e-4, epochs 1–5 | 5 | no | yes in 5/5 |
| Fine-tune, lr = 1e-3, epochs 1–5 | 5 | no | yes in 4/5; fails at epoch 5 (watermark 0.27, test acc 98.85%) |
| Prune 20–90% | 5 | no | yes in 5/5 |
| Prune 95% | 1 | no | yes, exactly at the threshold (50/100) |
| Independent models | 3 | no | no (watermark 0.11–0.14) |

Main observations:

1. The hash stopped matching after all 16 transformations, while the watermark still supported the claim in 15 of them.
2. Fine-tuning with lr = 1e-3 removed the watermark while test accuracy stayed at about 98.9%: the model stays useful, but the ownership evidence silently stops holding.
3. At 95% pruning the claim sat exactly on the threshold, so the threshold choice alone decided the verdict.
4. No independent model passed. The binomial probability that a uniformly guessing model reaches 50/100 is 5.83e-24, but real models are not uniform guessers, so the measured values are the more meaningful check.

Per-model values are in [`results.csv`](results.csv) (when re-run, the notebook also records each model's hash, shortened to the first 16 hex characters of the SHA-256 digest).

## Limitations

One small dataset, one architecture, one watermarking scheme, three independent models, no adaptive attacker and no model extraction. It is a pilot, not a benchmark.

## Run it

Open `watermark_pilot.ipynb` in Google Colab and run all cells. The shown outputs come from one run on a CPU runtime; a GPU runtime is faster. Seeds are fixed.

## How this was made

I am a beginner in PyTorch and highly interested in exploring and deepening my skills in AI and ML, especially in ML security. I wrote this pilot with the help of an AI assistant (Claude). I ran the fine-tuning and pruning experiments myself, observed how the results changed, and checked that the numbers in this README match the notebook outputs.

## References

1. Y. Adi, C. Baum, M. Cisse, B. Pinkas, J. Keshet. [*Turning Your Weakness Into a Strength: Watermarking Deep Neural Networks by Backdooring.*](https://www.usenix.org/conference/usenixsecurity18/presentation/adi) 27th USENIX Security Symposium, 2018, pp. 1615–1631.
2. J. Liu, R. Zhang, S. Szyller, K. Ren, N. Asokan. [*False Claims against Model Ownership Resolution.*](https://www.usenix.org/conference/usenixsecurity24/presentation/liu-jian) 33rd USENIX Security Symposium, 2024.
