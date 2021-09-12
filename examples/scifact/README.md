# SciFact

We use SciFact example to show how to train dense retrieval in the "research" way by using `run.py`

> Note: Different from original [SciFact](https://github.com/allenai/scifact) task that focus on Fact Verification, we focus on the retrieval stage, 
and consider a document as relevant to a claim if it appears in `cited_doc_ids`. 

## Dataset Preparation 
The SciFact dataset is self contain in our toolkit based on huggingface datasets.
The dataset name is `Tevatron/scifact`, see below for details.

## Train
```bash
CUDA_VISIBLE_DEVICES=0 python run.py \
  --output_dir scifact_model_e80_64x2 \
  --model_name_or_path bert-base-uncased \
  --do_train \
  --save_steps 20000 \
  --dataset_name Tevatron/scifact \
  --fp16 \
  --per_device_train_batch_size 64 \
  --train_n_passages 2 \
  --learning_rate 1e-5 \
  --q_max_len 64 \
  --p_max_len 512 \
  --num_train_epochs 80 \
  --grad_cache \
  --gc_p_chunk_size 8 \
  --logging_steps 10 \
  --overwrite_output_dir
```

## Encode Corpus
```bash
CUDA_VISIBLE_DEVICES=0 python run.py \
  --do_encode \
  --output_dir=temp_out \
  --model_name_or_path scifact_model_e80_64x2 \
  --fp16 \
  --per_device_eval_batch_size 156 \
  --dataset_name Tevatron/scifact/corpus \
  --p_max_len 512 \
  --encoded_save_path docs_emb/docs.pt 
```

## Encode Query
```bash
CUDA_VISIBLE_DEVICES=0 python run.py \
  --do_encode \
  --output_dir=temp_out \
  --model_name_or_path scifact_model_e20_16x2 \
  --fp16 \
  --per_device_eval_batch_size 156 \
  --dataset_name Tevatron/scifact/dev \
  --encode_is_qry
  --q_max_len 64 \
  --encoded_save_path queries_emb/queries.pt 
```

## Search
```bash
python -m dense.faiss_retriever \
--query_reps queries_emb/queries.pt \
--passage_reps docs_emb/docs.pt \
--depth 20 \
--batch_size -1 \
--save_text \
--save_ranking_to run.scifact.dev.txt
```

## Evaluate
Download qrels
```bash
wget https://www.dropbox.com/s/lpq8mfynqzsuyy5/dev_qrels.txt
```

Evaluate
```bash
python -m dense.utils.format.convert_result_to_trec --input run.scifact.dev.txt --output run.scifact.dev.trec
python -m pyserini.eval.trec_eval -c -mrecip_rank -mndcg_cut.10 dev_qrels.txt run.scifact.dev.trec
```

Following results can be reproduced by following the instructions above:
```
recip_rank              all     0.7322
ndcg_cut_10             all     0.7473
```
Comparing with BM25 baseline: `NDCG@10=0.665`