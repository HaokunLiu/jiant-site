# jiant: The Manual

## Getting Started

To find the setup instructions for using jiant and to run a simple example demo experiment using data from GLUE, follow this [getting started tutorial](https://github.com/nyu-mll/jiant/blob/master/tutorials/setup_tutorial.md)! 


## Supported Tasks

We currently support the below data sources, plus several more documented only in the code:
- All [GLUE](https://gluebenchmark.com) tasks (downloadable [here](https://github.com/nyu-mll/jiant/blob/master/scripts/download_glue_data.py))
- All [SuperGLUE](https://gluebenchmark.com) tasks (downloadable [here](https://github.com/nyu-mll/jiant/blob/master/scripts/download_superglue_data.py))
- Translation: WMT'14 EN-DE, WMT'17 EN-RU. Scripts to prepare the WMT data are in [`scripts/wmt/`](scripts/wmt/).
- Language modeling: [Billion Word Benchmark](http://www.statmt.org/lm-benchmark/), [WikiText103](https://einstein.ai/research/the-wikitext-long-term-dependency-language-modeling-dataset). We use the English sentence tokenizer from [NLTK toolkit](https://www.nltk.org/) [Punkt Tokenizer Models](http://www.nltk.org/nltk_data/) to preprocess WikiText103 corpus. Note that it's only used in breaking paragraphs into sentences. It will use default tokenizer on word level as all other tasks unless otherwise specified. We don't do any preprocessing on BWB corpus.  
- DisSent: Details for preparing the data are in [`scripts/dissent/README`](scripts/dissent/README).
- DNC (**D**iverse **N**atural Language Inference **C**ollection) recast data: The DNC is available [online](https://github.com/decompositional-semantics-initiative/DNC). Follow the instructions described there to download the DNC.
- CCG: Details for preparing the data are in [`scripts/ccg/README`](scripts/ccg/README).
- Edge probing analysis tasks: see [`probing/data`](probing/data/README.md) for more information.

Data files should be in the directory specified by `data_dir` in a subdirectory corresponding to the task, as specified in the task definition (see [`src/tasks`](https://github.com/nyu-mll/jiant/tree/master/src/tasks)). The GLUE and SuperGLUE download scripts should create acceptable directories automatically.

To add a new task, refer to this [tutorial](https://github.com/nyu-mll/jiant/blob/master/tutorials/adding_tasks.md)!


## Command-Line Options

All model configuration is handled through the config file system and the `--overrides` flag, but there are also a few command-line arguments that control the behavior of `main.py`. In particular:

`--tensorboard` (or `-t`): use this to run a [Tensorboard](https://www.tensorflow.org/guide/summaries_and_tensorboard) server while the trainer is running, serving on the port specified by `--tensorboard_port` (default is `6006`).

The trainer will write event data even if this flag is not used, and you can run Tensorboard separately as:
```
tensorboard --logdir <exp_dir>/<run_name>/tensorboard
```

`--notify <email_address>`: use this to enable notification emails via [SendGrid](https://sendgrid.com/). You'll need to make an account and set the `SENDGRID_API_KEY` environment variable to contain the (text of) the client secret key.

`--remote_log` (or `-r`): use this to enable remote logging via Google Stackdriver. You can set up credentials and set the `GOOGLE_APPLICATION_CREDENTIALS` environment variable; see [Stackdriver Logging Client Libraries](https://cloud.google.com/logging/docs/reference/libraries#client-libraries-usage-python).


## Models

Models in `jiant` generally have three components: A shared input component (typically a word embedding layer, or a pretrained ELMo, GPT, or BERT model), a shared general-purpose encoder component which sits on top of the input component (optional, typically a BiLSTM trained from scratch within `jiant`, specified by `sent_enc`), and task-specific components for each tasks.


### Input Components

#### ELMo

We use the ELMo implementation provided by [AllenNLP](https://github.com/allenai/allennlp/blob/master/tutorials/how_to/elmo.md).
To use ELMo, set ``elmo`` to 1.
By default, AllenNLP will download and cache the pretrained ELMo weights. If you want to use a particular file containing ELMo weights, set ``elmo_weight_file_path = path/to/file``.

To use only the _character-level CNN word encoder_ from ELMo, set `elmo_chars_only = 1`.

#### CoVe

We use the CoVe implementation provided [here](https://github.com/salesforce/cove).
To use CoVe, clone the repo and set the option ``path_to_cove = "/path/to/cove/repo"`` and set ``cove = 1``.

#### FastText

Download the pretrained vectors located [here](https://fasttext.cc/docs/en/english-vectors.html), preferably the 300-dimensional Common Crawl vectors. Set the ``word_emb_file`` to point to the .vec file.

#### GloVe

To use [GloVe pretrained word embeddings](https://nlp.stanford.edu/projects/glove/), download and extract the relevant files and set ``word_embs_file`` to the GloVe file.

### Specialized Shared Encoders

#### OpenAI GPT

To use the OpenAI transformer model, set `openai_transformer = 1`, download the [model](https://github.com/openai/finetune-transformer-lm) folder that contains pre-trained models, and place it under `src/openai_transformer_lm/pytorch_huggingface/`.

#### BERT

To use [BERT](https://arxiv.org/abs/1810.04805) architecture, set ``bert_model_name`` to one of the models listed [here](https://github.com/huggingface/pytorch-pretrained-BERT#loading-google-ai-or-openai-pre-trained-weigths-or-pytorch-dump), e.g. ``bert-base-cased``. You should also set ``tokenizer`` to the BERT model name used in order to ensure you are using the same tokenization and vocabulary. When using BERT, we generally follow the procedures set out in the original work as closely as possible: For pair sentence tasks, we concatenate the sentences with a special `[SEP]` token. Rather than max-pooling, we take the first representation of the sequence (corresponding to the special `[CLS]` token) as the representation of the entire sequence. We also have support for the version of Adam that was used in training BERT (``optimizer = bert_adam``).

[`copa_bert.conf`](https://github.com/nyu-mll/jiant/blob/master/config/copa_bert.conf) shows an example setup using BERT on a single task, and can serve as a reference.

#### Plain Transformer

We also include an experimental option to use a shared [Transformer](https://arxiv.org/abs/1706.03762) in place of the shared BiLSTM by setting ``sent_enc = transformer``. When using a Transformer, we use the [Noam learning rate scheduler](https://github.com/allenai/allennlp/blob/master/allennlp/training/learning_rate_schedulers.py#L84), as that seems important to training the Transformer thoroughly. 

#### Ordered Neurons (ON-LSTM) Grammar Induction Model

To use the ON-LSTM sentence encoder from [Ordered Neurons: Integrating Tree Structures into Recurrent Neural Networks](https://arxiv.org/abs/1810.09536), set ``sent_enc = onlstm``. To re-run experiments from the paper on WSJ Language Modeling, use the configuration file [config/onlstm.conf](config/onlstm.conf). Specific ON-LSTM modules use code from the [Github](https://github.com/yikangshen/Ordered-Neurons) implementation of the paper.

#### PRPN Grammar Induction Model

To use the PRPN sentence encoder from [***Neural language modeling by jointly learning syntax and lexicon***](https://arxiv.org/abs/1711.02013), set ``sent_enc=prpn``. To re-run experiments from the paper on WSJ Language Modeling, use the configuration file [config/prpn.conf](config/prpn.conf). Specific PRPN modules use code from the [Github](https://github.com/yikangshen/PRPN) implementation of the paper. 

Task-specific components include logistic regression and multi-layer perceptron for classification and regression tasks, and an RNN decoder with attention for sequence transduction tasks.
To see the full set of available params, see [config/defaults.conf](config/defaults.conf). For a list of options affecting the execution pipeline (which configuration file to use, whether to enable remote logging or Tensorboard, etc.), see the arguments section in [main.py](main.py).

## The Trainer

The standard trainer is designed around sampling-based multi-task training. At each step, a task is sampled one step of training is run on that task.
The trainer evaluates the model on the validation data after a fixed number of gradient steps, set by ``val_interval``.
The learning rate is scheduled to decay by ``lr_decay_factor`` (default: .5) whenever the validation score doesn't improve after ``lr_patience`` (default: `1`) validation checks.

If you're training only on one task, you don't need to worry about this. You'll still see macro-average and micro-average performance statistics, but these will simply be repetitions of your the results for your single task.

If you are training on multiple tasks, you can vary the sampling weights with ``weighting_method``, e.g. ``weighting_method = uniform`` or ``weighting_method = proportional`` (proportional to amount of training data). You can also scale the losses of each minibatch via ``scaling_method`` if you want to weight tasks with different amounts of training data equally throughout training. 

We use a shared global optimizer and LR scheduler for all tasks. In the global case, we use the macro average of each task's validation metrics to do LR scheduling and early stopping. When doing multi-task training and at least one task's validation metric should decrease (e.g. perplexity), we invert tasks whose metric should decrease by averaging ``1 - (val_metric / dec_val_scale)``, so that the macro-average will be well-behaved.

### Pretraining vs. Target-Task Training

Within a run, tasks are distinguished between training tasks (`pretrain_tasks`) and evaluation tasks (`target_tasks`). The logic of ``main.py`` is that the entire model is pretrained on all the `pretrain_tasks`, then the best model is then loaded, and task-specific components are trained for each of the evaluation tasks with a shared sentence encoder that is either frozen or reset between `target_tasks` (controlled by `transfer_paradigm`).
You can control which steps are performed or skipped by setting the flags ``do_pretrain, do_target_task_training, do_full_eval``.

Specify training tasks with ``pretrain_tasks = $pretrain_tasks`` where ``$pretrain_tasks`` is a comma-separated list of task names; similarly use ``target_tasks`` to specify the eval-only tasks.

For example, ``python main.py --overrides "pretrain_tasks = mnli, target_tasks = \"sst,mprc\""`` (note the escaped quotes in command line arguments for lists).
Note: if you want to train and evaluate on a task, that task must be in both ``pretrain_tasks`` and ``target_tasks``.

We support two modes of adapting pretrained models to target tasks. 
Setting `transfer_paradigm = finetune` will fine-tune the entire model while training for a target task.
The mode will create a copy of the full model for each target task after pretraining.
Setting `transfer_paradigm = frozen` will only train the target-task specific components while training for a target task.
If using ELMo and `sep_embs_for_skip = 1`, we will also learn a task-specific set of ELMo's layer-mixing weights.

[`copa_bert.conf`](https://github.com/nyu-mll/jiant/blob/master/config/copa_bert.conf) shows an example setup using a single task without pretraining, and can serve as a reference.

## Saving Preprocessed Data

Because preprocessing is expensive (e.g. building vocab and indexing for very large tasks like WMT or BWB), we often want to run multiple experiments using the same preprocessing. So, we group runs using the same preprocessing in a single experiment directory (set using the ``exp_dir`` flag) in which we store all shared preprocessing objects. Later runs will load the stored preprocessing. We write run-specific information (logs, saved models, etc.) to a run-specific directory (set using flag ``run_dir``), usually nested in the experiment directory. Experiment directories are written in ``project_dir``. Overall the directory structure looks like:

```
project_dir  # directory for all experiments using jiant
|-- exp1/  # directory for a set of runs training and evaluating on FooTask and BarTask
|   |-- preproc/  # shared indexed data of FooTask and BarTask
|   |-- vocab/  # shared vocabulary built from examples from FooTask and BarTask
|   |-- FooTask/  # shared FooTask class object
|   |-- BarTask/  # shared BarTask class object
|   |-- run1/  # run directory with some hyperparameter settings
|   |-- run2/  # run directory with some different hyperparameter settings
|   |
|   [...]
|
|-- exp2/  # directory for a runs with a different set of experiments, potentially using a different branch of the code
|   |-- preproc/
|   |-- vocab/
|   |-- FooTask/
|   |-- BazTask/
|   |-- run1/
|   |
|   [...]
|
[...]
```

You should also set ``data_dir`` and  ``word_embs_file`` options to point to the directories containing the data (e.g. the output of the ``scripts/download_glue_data`` script) and word embeddings (optional, not needed when using ELMo, see later sections) respectively.

To force rereading and reloading of the tasks, perhaps because you changed the format or preprocessing of a task, delete the objects in the directories named for the tasks (e.g., `QQP/`) or use the option ``reload_tasks = 1``.

To force rebuilding of the vocabulary, perhaps because you want to include vocabulary for more tasks, delete the objects in `vocab/` or use the option ``reload_vocab = 1``.

To force reindexing of a task's data, delete some or all of the objects in `preproc/` or use the option ``reload_index = 1`` and set ``reindex_tasks`` to the names of the tasks to be reindexed, e.g. ``reindex_tasks=\"sst,mnli\"``. You should do this whenever you rebuild the task objects or vocabularies.


## Updating `conf` Files

As some config arguments are renamed, you may encounter an error when loading past config files (e.g. params.conf) created before Oct 24, 2018. To update a config file, run

```sh
python scripts/update_config.py <path_to_file>
```


## Suggested Citation

If you use `jiant` in academic work, please cite it directly:

```
@misc{wang2019jiant,
    author = {Alex Wang and Ian F. Tenney and Yada Pruksachatkun and Katherin Yu and Jan Hula and Patrick Xia and Raghu Pappagari and Shuning Jin and R. Thomas McCoy and Roma Patel and Yinghui Huang and Jason Phang and Edouard Grave and Najoung Kim and Phu Mon Htut and Thibault F'{e}vry and Berlin Chen and Nikita Nangia and Haokun Liu and and Anhad Mohananey and Shikha Bordia and Ellie Pavlick and Samuel R. Bowman},
    title = {{jiant} 0.9: A software toolkit for research on general-purpose text understanding models},
    howpublished = {\url{http://jiant.info/}},
    year = {2019}
}
```

## Papers

`jiant` has been used in these three papers so far:

- [Looking for ELMo's Friends: Sentence-Level Pretraining Beyond Language Modeling](https://arxiv.org/abs/1812.10860)
- [What do you learn from context? Probing for sentence structure in contextualized word representations](https://openreview.net/forum?id=SJzSgnRcKX) ("Edge Probing")
- [Probing What Different NLP Tasks Teach Machines about Function Word Comprehension](https://arxiv.org/abs/1904.11544)

To exactly reproduce experiments from [the ELMo's Friends paper](https://arxiv.org/abs/1812.10860) use the [`jsalt-experiments`](https://github.com/jsalt18-sentence-repl/jiant/tree/jsalt-experiments) branch. That will contain a snapshot of the code as of early August, potentially with updated documentation.

For the [edge probing paper](https://openreview.net/forum?id=SJzSgnRcKX), see the [probing/](probing/) directory.


## License

This package is released under the [MIT License](LICENSE.md). The material in the allennlp_mods directory is based on [AllenNLP](https://github.com/allenai/allennlp), which was originally released under the Apache 2.0 license.

## FAQs

***I changed/updated the code, and my experiment broke with errors that don't seem related to the change. What should I do?***

Our preprocessing pipeline relies on Python pickles to store some intermediate data, and the format of these pickles is closely tied to the internal structure of some of our code. Because of this, you may see a variety of strange errrors if you try to use preprocessed data from an old experiment that was created with a previous version of the code.

To work around this, try your experiment again without the old preprocessed data. If you don't need your old log or checkpoints, simply delete your experiment directory (`$JIANT_PROJECT_PREFIX/your_experiment_name_here`) or move to a new one (by changing or overriding the `exp_name` config option). If you'd like to save as much old data as possible, try deleting the `tasks` subdirectory of your experiment directory, and if that doesn't work, try deleting `preproc` and `vocab` as well.

***It seems like my preproc/{task}\_\_{split}.data has nothing in it!***

This probably means that you probably ran the script before downloading the data for that task. Thus, delete the file from preproc and then run main.py again to build the data splits from scratch.

***How can I pass BERT embeddings straight to the classifier without a sentence encoder?***

Right now, you need to set `skip_embs=1` and `sep_embs_for_skip=1` just because of the current way 
our logic works. We're currently streamlining the logic around `sep_embs_for_skip` for the 1.0 release!

***How can I do STILTS-style training?***

For a typical STILTs experiment on top of BERT, GPT, ELMo, or some other supported pre-trained encoder, you can simply start from an effective configuration like [`config/superglue_bert.conf`](https://github.com/nyu-mll/jiant/blob/master/config/superglue-bert.conf) set `pretrain_tasks` to your intermediate task and `target_tasks` to your target task.

Right now, we only support training in two stages, so if you'd like to do the initial pretrianing stage from scratch, things get more complicated. Training in more than two stages is possible, but will require you to divide your training up into multiple runs. For instance, assume you want to run multitask training on task set A, and then train on task set B, and finally fine-tune on task set C. You would perform the following:
- First run: pretrain on task set A
   - pretrain_tasks=“task_a1,task_a2”, target_tasks=“”
- Second run: load checkpoints, and train on task set B and then C:
   - load_model = 1
   - load_target_train_checkpoint_arg=/path/to/saved/run
   - pretrain_tasks=“task_b1,task_b2, target_tasks=task_c1,task_c2”
   
***How can I train only one task?***
If you set do_pretrain = 1, do_target_train = 0, put your task in pretrain_tasks, and set target_tasks = "", then jiant will treat it as a normal non-multitask training experiment. 

***Can I evaluate on tasks that weren't part of the training process (not in pretraining or target task training)?***

Not at the moment. We're working on it!


## Getting Help

Post an issue here on GitHub if you have any problems, and create a pull request if you make any improvements (substantial or cosmetic) to the code that you're willing to share.

