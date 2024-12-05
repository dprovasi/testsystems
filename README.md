# BERTloweff

A simple model that incorporates a protein language model 
(proteinBERT) and a dense classifier based on ligand properties to classify partial agonists
at Class A GPCRs.


## 1. prepare fingerprints 

```
python calculate_fingerprints.py --dataset_folder ./datasets \
      --input_file combined_training_properties_seqs_v4_loweff.csv \
      --output_file combined_training_properties_seqs_v4_loweff_withfp.csv
```

to calculate the fingerprints on the biased ligands (activity) dataset
```
python calculate_fingerprints.py --input_file ligands_bias_v0.csv \
     --output ligands_bias_v0_withfp.csv --ligand_col ligand
```

## 2. Train classifiers

### 2.1 train with regular pretrain (on activity) + finetuning (on partiality)
#### 2.1.1 Train using random split

```
 # create the random splits
 tag=202411211058_random0
 python prepare_dataset.py --split random --dataset_tag $tag --labels partiality --random_state=1
 python prepare_dataset.py --split random --dataset_tag $tag --labels activity --random_state=1

 # pretrain then finetune
 python pretrain.py --dataset_tag $tag --labels activity 
 python predict.py --dataset_tag $tag --labels activity 

 python finetune.py --dataset_tag $tag --labels partiality
 python predict.py --dataset_tag $tag --labels partiality

```

On nodes with multiple cards specify one GPU 
```
CUDA_VISIBLE_DEVICES=0 python pretrain.py --dataset_tag $tag --labels activity
```

#### 2.1.2 Train using target split 

Prepare train/validation datasets splitting on targets:
```
tag=202411201713_targets0
python prepare_dataset.py --split targets --dataset_tag $tag --labels activity
python prepare_dataset.py --split targets --dataset_tag $tag --labels partiality \
   --validation_target_list validation_target_list_activity.csv

python pretrain.py --dataset_tag $tag --labels activity
python predict.py --dataset_tag $tag --labels activity 

python finetune.py --dataset_tag  $tag --labels partiality
python predict.py --dataset_tag $tag --labels partiality

```

#### 2.1.3 Train using ligand split

```
tag=202411210920_ligands0
python prepare_dataset.py --split liagnds --dataset_tag $tag --labels activity
python prepare_dataset.py --split ligands --dataset_tag $tag --labels partiality \
  --validation_ligand_list validation_ligand_list_activity.csv
```


### 2.2 Calculate performance on 'proper' training
```
python calculate_metrics.py  --dataset_list  trained_models.yaml \
   --output_overall metrics_overall.csv --output_byprotein metrics_byprotein.csv

```


### 2.3 Train efficacy classifier without fine tuning (nopretrain)
```

```


#### calc performance on 'nopretrain' training
```
python calculate_metrics.py --dataset_list trained_models_nopretrain.yaml \
   --output_overall metrics_overall_nopretrain.csv --output_byprotein metrics_byprotein_nopretrain.csv

```



### 2.4 Few-shot fine-tuning
```

# generate target-specific dataset in new folder
for i in 1 2 3 4 5 ; do
  # proteincode=CNR2
  # proteincode=CNR1
  # proteincode=OPRM
  # proteincode=OPRK
  # proteincode=OPRD
  # proteincode=APJ
  # proteincode=TAAR1
  proteincode=HRH4

  ti=`date | awk '{print $5}' | sed 's/://g' | cut -c 1-4`
  tag=20241203${ti}_fewshot_${proteincode}_random$i
  python prepare_dataset.py --split random --dataset_tag $tag --labels partiality --target_list ${proteincode} --random_state ${ti} --validation_size 0.8


  #copy trained model from folder
  mkdir ./weights/$tag
  cp ./weights/202411211838_random4/cp_finetune_weigthed_train-0010.ckpt* ./weights/$tag
  cp ./weights/202411211838_random4/checkpoint ./weights/$tag

  python finetune.py --dataset_tag  $tag --labels partiality
  python predict.py --dataset_tag $tag --labels partiality

done

```

#### Calculate performance on few-shot trained models
```

python collect_trained_models.py --dataset_folder ./datasets/fewshot/ --output_yaml trained_models_fewshot.yaml

python calculate_metrics.py  --dataset_list  trained_models_fewshot.yaml \
    --no-do_overall \
    --do_byprotein --output_byprotein metrics_byprotein_fewshot.csv \
    --no-do_byfamily 
```


## 3. baseline models

```
proteincode=OPRM
proteincode=APJ
proteincode=CNR1
proteincode=CNR2
proteincode=OPRK
proteincode=OPRD
proteincode=TAAR1
for i in 1 2 3 4 5; do
  echo "Number: $i"

  ti=`date | awk '{print $5}' | sed 's/://g' | cut -c 1-4`
  tag=20241204${ti}_${proteincode}_random$i

  python prepare_dataset.py --split random --dataset_tag $tag  --labels activity --target_list ${proteincode} --random_state ${ti}
  python prepare_dataset.py --split random --dataset_tag $tag --labels partiality --target_list ${proteincode} --random_state ${ti}
  python train_baseline.py --dataset_tag $tag --labels activity --predict_vali --train
  python train_baseline.py --dataset_tag $tag --labels partiality --pretrain_epochs 5 --predict_vali --train
  #python train_baseline.py --dataset_tag $tag --labels partiality --pretrain_epochs 5 --predict_vali --no-train

done
```

```
tag=benchmark_baseline
python prepare_dataset.py --split random --dataset_tag $tag --labels partiality --random_state 1234
python train_baseline.py --dataset_tag $tag --labels partiality --pretrain_epochs 1 --predict_vali --train

python prepare_dataset.py --split random --dataset_tag $tag --labels activity --random_state 1234
python train_baseline.py --dataset_tag $tag --labels activity --pretrain_epochs 1 --predict_vali --train

```


### calc performance for baseline models
```
python collect_trained_models.py --dataset_folder ./datasets/baseline/ --output_yaml trained_models_baseline.yaml

python calculate_metrics.py  --dataset_list  trained_models_baseline.yaml \
    --no-do_overall \
    --do_byprotein --output_byprotein metrics_byprotein_baseline.csv \
    --no-do_byfamily 
```




## 4. Train regressors 

### Using random split

```
# prepare datasets, activity for pretrain using classification task; 
tag=202411271233_regression_centered_random0
python prepare_dataset.py --split random --dataset_tag $tag --labels activity --random_state=98765
python prepare_dataset_regressor.py --dataset_tag $tag --split random --labels logactivity --channel Gprotein --center_mean 7 --center_sd 1.4 --random_state=98765
python prepare_dataset_regressor.py --dataset_tag $tag --split random --labels logactivity --channel Arrestin --center_mean 7 --center_sd 1.4 --random_state=98765

python pretrain.py --dataset_tag $tag --labels activity
python finetune_regressor.py --dataset_tag $tag --labels logactivity --channel Gprotein 
python predict_regressor.py --dataset_tag $tag --labels logactivity --channel Gprotein

python finetune_regressor.py --dataset_tag $tag --labels logactivity --channel Arrestin
python predict_regressor.py --dataset_tag $tag --labels logactivity --channel Arrestin

```


### train regressor with uncertainty

```
# prepare datasets, activity for pretrain using classification task;
tag=202412051407_regression_centered_random0
python prepare_dataset.py --split random --dataset_tag $tag --labels activity --random_state=1407
python prepare_dataset_regressor.py --dataset_tag $tag --split random --labels logactivity --channel Gprotein --center_mean 7 --center_sd 1.4 --random_state=1407
python prepare_dataset_regressor.py --dataset_tag $tag --split random --labels logactivity --channel Arrestin --center_mean 7 --center_sd 1.4 --random_state=1407

python pretrain.py --dataset_tag $tag --labels activity

```





