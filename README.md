# Semantic Change Detection

Instructions for installation assume the usage of PyPI package manager.<br/>


## Installation, documentation ##

Install dependencies if needed: pip install -r requirements.txt <br/>
You also need to download 'tokenizers/punkt/english.pickle' using nltk library.

### Prepare the data :<br/> 

The script preprocess.py accepts corpus in the tsv format as an input (see 'data/example_data.csv' for example). To run 
the script on the example data, run:<br/>

```
python preprocess.py  --data_path data/example_data.tsv --chunks_column date --text_column text --lang slo --output_dir output  --min_freq 10
```
**Arguments:**<br/>
**--data_path** Path to the tsv file containing the data <br/>
**--chunks_column** Name of the column in the data tsv file that should be used for splitting the corpus into chunk, 
between which semantic shift will be calculated <br/>
**--text_column** Name of the column in the data tsv file containing text <br/>
**--lang Language** of the corpus, currently only Slovenian ('slo') and English ('en') are supported <br/>
**--output_dir** Path to the folder that will contain generated output vocab and language_model training files <br/>
**--min_freq** Minimum frequency of the word in a specific chunk to be included in the vocabulary <br/>


**Outputs:**<br/>
1.) **preprocessed tsv corpus** saved to the folder containing input data <br/>
2.) **vocab.pickle** A pickled vocab class used as input for script get_embeddings_scalable.py <br/> 
3.) **train_lm.txt** An input train corpus for language model fine-tuning <br/>
4.) **test_lm.txt** An input test corpus for language model fine-tuning <br/>
5.) **vocab_list_of_words.csv** All words in the corpus vocab for which semantic shift will be calculated <br/>


### Fine-tune RoBERTa language model:<br/>

Fine-tune RoBERTa language model:<br/>

```
python finetune_mlm.py --train_file output/train_lm.txt --validation_file output/test_lm.txt --output_dir models --data_path data/example_data.tsv --chunks_column date --model_name_or_path EMBEDDIA/sloberta --do_train true --do_eval true --per_device_train_batch_size 16 --per_device_eval_batch_size 4 --save_steps 20000 --evaluation_strategy steps --eval_steps 20000 --overwrite_cache --num_train_epochs 10 --max_seq_length 512
```

**Arguments:<br/>**
**--train_file** An input train corpus for language model fine-tuning generated by the preprocessing.py script <br/>
**--validation_file** An input test corpus for language model fine-tuning generated by the preprocessing.py script <br/>
**--output_dir** Directory where the fine-tuned model is saved <br/>
**--data_path** Path to the tsv file containing the data <br/>
**--chunks_column** Name of the column in the data tsv file that should be used for splitting the corpus into chunk, 
between which semantic shift will be calculated) <br/> 
**--model_name_or_path** Which transformer model to use, currently only English 'roberta-base' and Slovenian 'EMBEDDIA/sloberta' models are supported <br/>  


**Outputs:**<br/>
1.) A **fine-tuned RoBERTa model** that can be used for embedding generation in script get_embeddings_scalable.py 

### Extract embeddings:<br/>

Generate corpus chunk specific embeddings:<br/>

```
python get_embeddings_scalable.py --vocab_path output/vocab.pickle --embeddings_path embeddings/embeddings.pickle --lang slo --path_to_fine_tuned_model models --batch_size 16 --max_sequence_length 256 --device cuda
```

**Arguments:**<br/>
**--vocab_path** Paths to vocab pickle file generated by the preprocessing.py script <br/>
**--embeddings_path** Path to output pickle file containing embeddings. <br/>
**--lang** Language of the corpus, currently only Slovenian ('slo') and English ('en') are supported <br/>
**--path_to_fine_tuned_model** Path to fine-tuned model. If empty, pretrained model is used <br/>

**Outputs:**<br/>
**1.) output pickle file containing embeddings**


### Conduct clustering and measure semantic shift:<br/>

```
python measure_semantic_shift.py --output_dir output --embeddings_path embeddings/embeddings.pickle -random_state 123 --cluster_size_threshold 10 -metric JSD
```

**Arguments:**<br/>
**--output_dir** Paths to output results folder <br/>
**--embeddings_path** Path to input pickle file containing embeddings <br/>
**--random_state** Random seed <br/>
**--cluster_size_threshold** Remove cluster or merge it with other if it contains less than threshold word usages <br/>
**--metric** Which metric to use for measuring semantic shift, should be JSD or WD <br/>

**Outputs:**<br/>
1.) **word_list_results.csv** a list of all words in the vocab with their semantic change scores <br/>
2.) **corpus_slices.pkl** pickled list of corpus slices used as input for script 'interpretation.py' <br/>
3.) **id2sents.pkl** pickled sentence dictionary used as input for script 'interpretation.py' <br/>
4.) **kmeans_5_labels.pkl** pickled dictionary of kmeans cluster labels for each word usage used as input for script 'interpretation.py' <br/>
5.) **sents.pkl** pickled list of all sentences  used as input for script 'interpretation.py' <br/>


### Extract keywords for each cluster and plot clusters distributions for interpretation:<br/>

```
python interpretation.py  --target_words "valovanje,letnik" --lang slo --input_dir output --results_dir results --cluster_size_threshold 10 --max_df 0.8 --num_keywords 10
```

**Arguments:**<br/>
**--target_words** Target words to analyse, separated by comma <br/>
**--lang** Language of the corpus, currently only Slovenian ('slo') and English ('en') are supported <br/>
**--input_dir** Folder containing data generated by the script 'measure_semantic_shift.py <br/>
**--results_dir** Path to final results <br/>
**--max_df** Words that appear in more than that percentage of clusters will not be used as keywords <br/>
**--cluster_size_threshold** Clusters smaller than a threshold will be deleted <br/>
**--num_keywords** Number of keywords per cluster <br/>


**Outputs:**<br/>
**1.) An image showing a distribution of word usages for each target word** <br/>
**2.) A tsv document per each target word containing information about sentences in which it appeared and into which cluster was each usage clustered** <br/>





