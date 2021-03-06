Please read [TrainingTesseract 4.00](https://github.com/tesseract-ocr/tesseract/wiki/TrainingTesseract-4.00) before trying the following.

Have copies of the 4.0.0 alpha langdata, tessdata and tesseract-ocr repositories in the following directory structure.

```
./langdata
./tessdata
./tesseract-ocr
./tesseract-ocr/tessdata
./tesseract-ocr/tessdata/configs/
```

Make a copy of English and Arabic 4.0.0alpha traineddata files in ./tesseract-ocr/tessdata

Check that lstm.train is available under configs.

Setup appropriate TESSDATA_PREFIX directory.
```
cp ./tessdata/eng.traineddata ./tesseract-ocr/tessdata
cp ./tessdata/ara.traineddata ./tesseract-ocr/tessdata
```

Change to the tesseract-ocr directory and then follow the given commands.

```
cd ./tesseract-ocr

training/tesstrain.sh --fonts_dir /usr/share/fonts --lang ara  --linedata_only \
  --training_text ../langdata/ara/arabic1.txt \
  --langdata_dir ../langdata --tessdata_dir ./tessdata \
  --fontlist "Times New Roman," \
  --output_dir ~/tesstutorial/aratest
```
This creates the .lstmf files in the output directory using the given training_text. The box/tiff pairs are created in a /tmp/<tmpdir>/ara/ directory and are not copied to the output directory.
```
mkdir -p ~/tesstutorial/aratuned_from_ara 

combine_tessdata -e ../tessdata/ara.traineddata \
  ~/tesstutorial/aratuned_from_ara/ara.lstm
  
lstmtraining --model_output ~/tesstutorial/aratuned_from_ara/aratuned \
  --continue_from ~/tesstutorial/aratuned_from_ara/ara.lstm \
  --train_listfile ~/tesstutorial/aratest/ara.training_files.txt \
  --target_error_rate 0.01 
```

The above commands extract the existing LSTM model for Arabic from ./tessdata and finetune it using the .lstmf files created earlier, given in the train_listfile.
``` 
lstmtraining --model_output ~/tesstutorial/aratuned_from_ara/aratuned.lstm \
  --continue_from ~/tesstutorial/aratuned_from_ara/aratuned_checkpoint \
  --stop_training
```
The above command creates the new LSTM model from the finetuning output.
```
combine_tessdata -o ./tessdata/ara.traineddata \
  ~/tesstutorial/aratuned_from_ara/aratuned.lstm \
  ~/tesstutorial/aratest/ara.lstm-number-dawg \
  ~/tesstutorial/aratest/ara.lstm-punc-dawg \
  ~/tesstutorial/aratest/ara.lstm-word-dawg 
```  
Finally, the new LSTM model and new dawg files can be combined with the existing Arabic traineddata in ./tesseract-ocr/tessdata. The old ara.traineddata file in ./tesseract-ocr/tessdata is renamed.

```
training/lstmeval --model ~/tesstutorial/aratuned_from_ara/ara.lstm \
  --eval_listfile ~/tesstutorial/aratest/ara.training_files.txt  
  
training/lstmeval --model ~/tesstutorial/aratuned_from_ara/aratuned_checkpoint \
  --eval_listfile ~/tesstutorial/aratest/ara.training_files.txt  
  
training/lstmeval --model ~/tesstutorial/aratuned_from_ara/aratuned.lstm \
  --eval_listfile ~/tesstutorial/aratest/ara.training_files.txt  
``` 

The above three commands evaluate the LSTM models, first with the original Arabic LSTM model, second with the checkpoint model during finetuning and third with the finetuned Arabic model.

```
time tesseract --tessdata-dir ../tessdata /tmp/<tmpdir.ara/ara.Times_New_Roman.exp0.tif out-4alpha.txt -l ara
time tesseract --tessdata-dir ./tessdata /tmp/<tmpdir>/ara/ara.Times_New_Roman.exp0.tif out-tuned.txt -l ara  
```

The above runs OCR on the tif file created during training with the original traineddata and finetuned traineddata 

