# TenjinQABench

This repository contains **TenjinQABench** evaluation script and data, which provides a holistic evaluation platform to test LLMs' abilities of assisting researchers to conduct scientific literature synthesis. This work is from the [OpenTenjin]((https://github.com/esstringbean/Tenjin)) project. 

**Change logs** 

- December 6 2024: Initial release. 

**Table of Contents**

1. [Overview and Installation](#overview-and-installation)
2. [Evaluations](#evaluations)
3. [License](#license)
4. [Citations](#citations)


## Overview and Installation
### Setup

```python
conda create -n sb_env python=3.10.0
conda activate sb_env
pip install -r requirements.txt
python -m nltk.downloader punkt_tab
``` 

### Repository Organizations

- [data](data/): provides relevant data files. 
    - [Tenjin_cs](Tenjin_cs): includes **TenjinQA-CS** (Computer Science) data files
        - ``output_snippets.jsonl`` : Contains the questions with system responses for eval (and some other metadata used to generate the test cases but not required for subsequent runs). Should not require further modification.
        - `test_configs_snippets.json` : A collection of test cases in json format with associated rubrics for each question. Each question has its own test case and rubrics. Should not require further modification.
        - `qa_metadata_all.jsonl` : Metadata file that was used to bootstrap this utility. Should not require further modification.
        - `src_answers`: Directory containing sample system responses from 4 systems.

    - [Tenjin_multi](Tenjin_multi): includes **TenjinQA-Multi** (Multi-domain; CS, Bio and Physics) data files
    - [Tenjin_bio](Tenjin_bio): includes **TenjinQA-Bio** (Biomedicine) data files
    - [Tenjin_neuro](Tenjin_neuro): includes **TenjinQA-Neuro** (Neuroscience) data files
- [scripts](scripts/): provides evaluation scripts for each evaluation aspects
    - [rubric_eval.py](scripts/rubric_eval.py): a script to run rubric based evaluations for **TenjinQA-CS**. 
    - [citation_correctness_eval.py](scripts/citation_correctness_eval.py): a script to run citations as well as string-matching-based correctness evaluations for single-paper tasks. 
    - [prometheus_eval.py](scripts/prometheus_eval.py): a script to evaluate *organization*, *relevance* and *coverage*, based on five-scale rubrics in [rubrics](rubrics/prometheous_rubrics_v8.json)
- [rubrics](rubrics): rubrics for `prometheus_eval`. 

The availability of annotations, as well as overview of the annotations are summarized below. Note that TenjinQA-Bench does not provide training data. 

| Dataset    | Input | Output | Label Available | Evaluation Metrics |
| :-------- | :-------: |:-------: | :-------: | :-------: |
| `TenjinQA-SciFact`  |  claim   | `true` or `false` |  ✅ | `accuracy`, `citations_short` |
| `TenjinQA-PubmedQA`  |  question   | `yes` or `no` | ✅  | `accuracy`, `citations_short` |
| `TenjinQA-QASA`  |  question   | long-form |  ✅ | `rouge-l`, `citations` |
| `TenjinQA-CS`  |  question   | long-form |  ✅ (rubrics) | `rubrics`, `citations` |
| `TenjinQA-Multi`  |  question   | long-form |  ✅ | `prometheus`, `citations` |
| `TenjinQA-Bio`  |  question   | long-form |  | `citations` |
| `TenjinQA-Neuro`  |  question   | long-form | |  `citations` |



## Evaluations
After you run model inferences, run evaluations for each task and aspects using the following scripts. 

Your answer files for all tasks are expected to format in the following way (`json``) 
```
[
{"input": query (str), "output": final_model_output (str), "ctxs: citations (list of    dict, where each dict has text )}, ...
]
```

### Citation Accuracy (All tasks) 

#### Short-form generation (SciFact, PubMedQA)

```
python citation_correctness_eval.py --f PATH_TO_YOUR_PREDICTION_FILE --citations_short
```


#### Long-form generation (QASA, TenjinQA-*)

```
python citation_correctness_eval.py --f PATH_TO_YOUR_PREDICTION_FILE --citations_long
```


### String-based Correctness (SciFact, PubmedQA, QASA)
To run string based evaluations, run the following commands:

#### SciFact and PubMedQA (accuracy)

```
python citation_correctness_eval.py --f PATH_TO_YOUR_PREDICTION_FILE --match
```

#### QASA (ROUGE-L)

```
python citation_correctness_eval.py --f PATH_TO_YOUR_PREDICTION_FILE
```


### Rubric-based Correctness (TenjinQA-CS)

#### Convert the output file
To run eval for your system, first setup the prediction file with the system answers to be evaluated as per following requirement:

A jsonl file with fields `case_id` and `answer_text` (See [example file](https://github.com/allenai/multidoc_qa_eval/blob/main/data/src_answers/gpt.jsonl)) -

- `case_id` corresponds to the identifier of the question for which the response is to be evaluated, map the question text with the case_id in `test_configs_snippets.json`
- `answer_text` is the system answer (along with citations and excerpts, if applicable) in plain text

We provide an answer conversion script, [`convert_answer_nora.py`](scripts/convert_answer_nora.py), which convert the original answer file to the expected format. 

```
python scripts/convert_answer_nora.py \
    --pred_file YOUR_PRED_FILE_NAME \
    --data_file data/Tenjin_cs/test_configs_snippets.json \
    --output_file Tenjin_cs/src_answers/CONVERTED_OUTPUT_FILE_NAME
```

##### Run evaluation 
Once the prediction json file is ready, save it a new directory run the eval script as follows (You can save as many system response files under a directory, they will be picked together for eval):

```python
export OPENAI_API_KEY=<openai key>
python scripts/rubric_eval.py \
    --qa-dir data/Tenjin_cs/src_answers \
    --test-config data/Tenjin_cs/test_configs_snippets.json \
    --rubrics --snippets \
    --src-names <optional comma separated src names prefixes of prediction files with .jsonl, if not given all the files will be picked>
```
**Note:** To evaluate only using rubrics, remove `--snippets` parameter and vice-versa to use only snippets. 

**Acknowledgements:** The original code of the TenjinQA-CS are available at [allenai/multidoc_qa_eval](https://github.com/allenai/multidoc_qa_eval).

### Prometheus for Coverage, Relevance and Organization (TenjinQA-CS)

#### Coverage and Organization

```
python scripts/prometheus_eval.py \
    --batch_process_dir YOUR_PREDICTION_FILE_PATH \
    --output_path OUTPUT_DIR_NAME \
    --rubric_path rubrics/prometheus_rubrics_v8.json \
    --instruction "Answer the question related to the most recent scientific literature." \
    --model prometheus-eval/prometheus-7b-v2.0 \
    --load_vllm \
    --top_n 10 \
    -f data/Tenjin_multi/human_answers.json \
    --aspects organization coverage
```

#### Relevance 
We use `prometheus-eval/prometheus-8x7b-v2.0` for relevance, due to unstable performance of `prometheus-eval/prometheus-bgb-8x7b-v2.0` for relevance. 

```
python scripts/prometheus_eval.py \
    --batch_process_dir YOUR_PREDICTION_FILE_PATH \
    --output_path OUTPUT_DIR_NAME \
    --rubric_path rubrics/prometheus_rubrics_v8.json \
    --instruction "Answer the question related to the most recent scientific literature." \
    --model prometheus-eval/prometheus-bgb-8x7b-v2.0 \
    --load_vllm \
    --top_n 10 \
    -f data/Tenjin_multi/human_answers.json \
    --aspects relevance
```

## License
The aggregate test cases, sample system answers under `data/src_answers` and other files under data directory are released under [ODC-BY](https://opendatacommons.org/licenses/by/1.0/) license. 
By downloading this data you acknowledge that you have read and agreed to all the terms in this license.
For constituent datasets, also go through the individual licensing requirements, as applicable. 


