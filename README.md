# Minimal text diffusion



_A minimal implementation of diffusion models of text: given a corpus of text, sample new sentences from the same corpus._



----


| ![diffusion](./minimal-text-diffusion.gif) |
|:--:|
| <b> Diffusion in action:</b> a DDPM model gradually denoising random text _`hotnutggy pi greentedsty rawyaented`_ to _`the white eggplant is dried`_ and _`mac clement star fe honey spin theapple purpleip`_ to _`the brown radicchio is sour`_|


----

This repo has been refactored by taking a large amount of code from https://github.com/XiangLi1999/Diffusion-LM (which includes some code from: https://github.com/openai/glide-text2im), thanks to the authors for their work!

The main idea was to retain _just enough code_ to allow training a simple diffusion model and generating samples, remove image related terms, and make it easier to use.

 I've included an extremely simple corpus (`data/simple-{train,test}.txt`) I used for quick iterations and testing.

---

## Getting started

### Setup

- Install the requirements: `pip install -r requirements.txt`

- Some of the dependencies might be easier to install via conda:
```sh
conda install mpi4py
conda install pytorch torchvision torchaudio cudatoolkit=11.3 -c pytorch
```

### Training

- To train a model, run `scripts/train.sh`. By default, this will train a model on the simple corpus. However, as long as you have a corpus of text, you can train a model on it by specifying the path to the corpus in the `--train_data` argument. Note that you may have to increase the sequence length (`--seq_len`) if your corpus is longer than the simple corpus. The other default arguments are set to match the best setting I found for the simple corpus (see discussion below).

- Once training finishes, the model will be saved in `ckpts/simple`. You can then use this model to generate samples.

- The checkpoint can also be downloaded from [here](https://drive.google.com/drive/folders/1UXx1HJVeWdAjlTNTiCydnCCHD431Q4yh?usp=sharing).



### Inference

- To generate samples, run:

```sh
bash scripts/text_sample.sh ckpts/simple/ema_0.9999_025000.pt 2000 10
```
- Here:
    * `ckpts/simple/ema_0.9999_025000.pt` is the path to the checkpoint
    * `2000` is the number of diffusion steps.
    * `10` is the number of samples to generate.

- By default, this will generate 10 samples from the model trained on the simple corpus. Changing `SEED` in `scripts/text_sample.sh` will generate different samples. You can also change the number of samples generated by changing the `NUM_SAMPLES` argument.

- During inference (denoising), the intermediate sentences will be printed to the console.

- The generated samples will be saved in `ckpt/simple/`.

- Complete set of outputs are available [here](https://drive.google.com/drive/folders/1UXx1HJVeWdAjlTNTiCydnCCHD431Q4yh?usp=sharing).

---


## Experiments and Results

* I've tried experiments with the following hyperparameters:

1. Embeddings/vocab: pre-trained `bert-base-uncased` vs. initialized randomly.

2. Model backbone: pre-trained `bert-base-uncased` vs. initialized from scratch.

3. Embeddings fine-tuning: fine-tuned vs. frozen.

Out of the 8 possible combinations, the best results were obtained with the following hyperparameters:

| File                                               | Sample Sentences                                                                                                                                                           | Perplexity         | % Unique Lines | % Unique Tokens |
|----------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------|----------------|-----------------|
| MODEL_PT-False_EMBEDS_PT-False-FREEZE_EMBEDS-False | The yellow lentil is stir-fried., The green lemon is stir-fried., The white kiwi is deep-fried., The orange turnip is stir-fried., The blue blackberry is deep-fried.      | 212.70 | 80.0           | 3.76            |
| MODEL_PT-False_EMBEDS_PT-False-FREEZE_EMBEDS-True  | The green spinach is stir-fried., The pink pomelo is stir-fried., The brown onion is stir-fried., The yellow artichoke is stir-fried., The blue pomegranate is deep-fried. | 218.77  | 74.2           | 3.76            |
| MODEL_PT-True_EMBEDS_PT-True-FREEZE_EMBEDS-False   | the yellow poccoli isggy., the red pe is sauteed., the green spinach is candied., the green danmelli isy., the brown kale is candied.                                      | 1424.21  | 78.0           | 6.1             |


---

- The best setting in terms of diversity is using pre-trained bert, bert embeddings, and fine-tuning the embeddings. However, this setting has the lowest perplexity because it generates weird sentences (which could be a good thing depending on the application!).

- Some random samples from this  setting:

```
the purple chard is braised. 
the pink eggplant is soft. 
the blue perychmmon is fluffy. 
the pink eggplant is fluffy. 
the orange macadamia is spicy. 
the blue almond is poached. 
the black avychnd is steamed. 
the brown radicchio is delicious. 
the blue yam is microwaved. 
the black pistachio is dried. 
```

- The model was trained on a single RTX 2080 Ti GPU for 25k steps. The training time was ~1 hours.

---

## Gory details


* Below are my rough notes on how the code works. [TODO] Clean this up and add more details.

### Training

* Input text is embedded. This is the mean of `x_start_mean`. Some noise is added to `x_start_mean` to get `x_start`.

* Using random `t`, a noisy version of the input is created from q(x_t | x_0). This is simply x_t = x_0 * sqrt(1 - \beta_t) + \epsilon_t * sqrt(\beta_t). The function used for this is `q_sample`. Any operation that involves going ahead in the diffusion process is carried out by functions that start with `q_`.

* `x_t` is fed to the transformer model. The transformer model is trained to generate an approximation of `x_start` given `x_t` and `t` (the timestep). Specifically, the embedded text is passed through a BERT encoder, and then downsampled. The size of the output embeddings and input embeddings is obviously the same for this reason. Maybe this is the trick mentioned in the paper where they want to tie each weight with the `x_start` term, but I'm not sure how it's different from DDIM.

* The loss has several terms:
1) Difference between the actual `x_start` and the output of the transformer model. This is the MSE loss.
2) Mean of the `xT` should be close to zero. This is the `tT_loss` term. It is obtained by calling `q_mean_variance` for the t=T. `q_mean_variance` is like `q_sample` but it returns the mean and variance of the distribution `x_t | x0`  instead of a sample.

3) Decoder NLL loss. This is the `decoder_nll` term. It is obtained by calling `token_discrete_loss`. `token_discrete_loss` calls `get_logits`, which in turns uses the embeddings to convert to logits. The logits are then used to calculate the NLL loss. Essentially this is how the embeddings are trained.

```py

    def get_logits(self, hidden_repr):
        return self.lm_head(hidden_repr)
```


- One thing to note is that:

```py
    print(model.lm_head.weight == model.word_embedding.weight)
    print(model.lm_head.weight.shape, model.word_embedding.weight.shape)
```

They are identical! Intuitively, the model is trained to predict the embedded input. Thus, having a linear layer with the weights from `word_embedding` is like doing a nearest neighbor search. While initializing, the weights are assigned to `lm_head` from `word_embedding` under `torch.no_grad()`, so that the gradients are not computed for `lm_head`. 


### Evolving input

- Note that the embeddings are *trained*. In training losses, although initial embeddings are passed, they are not used. Instead, the `get_embeds` method is used to get the embeddings. This is because the embeddings are trained to predict the input text. Thus, the embeddings are not the same as the input embeddings.


### Sampling

* `p_mean_variance`: returns the distribution `p(x_{t-1} | x_t)` (the mean and variance). In addition, returns prediction for the initial `x_0`.

* `q_posterior_mean_variance`: returns the distribution `q(x_{t-1} | x_t, x_0)`. 

* Additionally, recall that our model is trained to predict `x_start` given `x_t` and `t`.

- Putting these together, we can sample from the model. The sampling is done in the following way:

1. Starting with noise `xT`, a noisy `x_start` is first generated using the model. 

2. The `xT` and `x_start` are used to generate `x_{T-1}` using `q_posterior_mean_variance` (`x_{T-1} ~ q(x_{T-1} | x_T, x_start)`).

The process is repeated until `x_0` is generated.

---


## TODO

- [ ] Add more details to the inner workings section.
- [ ] Add classifier-guided sampling.
- [ ] Add more experiments.


### Opportunities for further minimization

- [ ] `logger.py` can be completely deleted.
- [ ] `args.py` and `factory_methods.py` can be combined.



