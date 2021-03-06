Please read [TrainingTesseract 4.00](https://github.com/tesseract-ocr/tesseract/wiki/TrainingTesseract-4.00) before trying the following.

Hindi to Bihari
----------

This example uses hin.traineddata (Hindi) for 4.0.0-alpha and builds on it with modified langdata for bih (Bihari languages) using the replace top layer method of training to create bin.traineddata.

---

Have copies of the 4.0.0 alpha langdata, tessdata and tesseract-ocr repositories in the following directory structure.

```
./langdata
./tessdata
./tesseract-ocr
./tesseract-ocr/tessdata
./tesseract-ocr/tessdata/configs/
```
Check that lstm.train is available under configs.

Setup appropriate TESSDATA_PREFIX directory.
```
cp ./tessdata/eng.traineddata ./tesseract-ocr/tessdata
cp ./tessdata/hin.traineddata ./tesseract-ocr/tessdata/bih.traineddata
```

Change to the tesseract-ocr directory.

```
cd ./tesseract-ocr
```

Then give the following commands:

Step 1.
-----
```
training/tesstrain.sh --fonts_dir /usr/share/fonts --lang bih  --linedata_only \
  --langdata_dir ../langdata --tessdata_dir ./tessdata \
  --fontlist "Lohit Devanagari" \
  --output_dir ~/tesstutorial/bihtest
```
This creates the .lstmf files in the output directory using the given training_text. The box/tiff pairs are created in a /tmp/sometmpdir/hin/ directory and are not copied to the output directory. You can modify [tesstrain_utils.sh](https://github.com/tesseract-ocr/tesseract/blob/master/training/tesstrain_utils.sh) to save these alongwith the .lstmf files, if needed.

Do not use `--fontlist` if you want to train on the list of Devanagari fonts from [language_specific.sh](https://github.com/tesseract-ocr/tesseract/blob/master/training/language-specific.sh).

Step 2.
-----
```
mkdir -p ~/tesstutorial/bihlayer_from_hin 

combine_tessdata -e ../tessdata/hin.traineddata \
  ~/tesstutorial/bihlayer_from_hin/hin.lstm
  
training/lstmtraining -U ~/tesstutorial/bihtest/bih.unicharset \
  --script_dir ../langdata --debug_interval 0 \
  --continue_from   ~/tesstutorial/bihlayer_from_hin/hin.lstm \
  --append_index 5 --net_spec '[Lfx256 O1c105]' \
  --model_output ~/tesstutorial/bihlayer_from_hin/bihlayer \
  --train_listfile ~/tesstutorial/bihtest/bih.training_files.txt \
  --max_iterations 50000
```
The above commands extract the existing LSTM model for Hindi from ./tessdata and changes the top layer for it for Bihari using the .lstmf files created earlier, given in the `--train_listfile`. The `--model_output` directory holds the intermittent 'best' language models and the latest checkpoint.

Use either `--targer_error_rate 0.001` or `--max_iterations 50000`, the values can be changed.

Use `--debug_interval -1` if you want to see verbose output for every iteration. 
Try it with `--max_iterations 50` to see a sample of debug output.

This training process can be stopped at any time. Giving the command again restarts the process from the last saved checkpoint. 

Currently this process has a bug and gets aborted when `--eval_listfile` is given as a parameter.

Step 3.
-----
``` 
lstmtraining --model_output ~/tesstutorial/bihlayer_from_hin/bihlayer.lstm \
  --continue_from ~/tesstutorial/bihlayer_from_hin/bihlayer_checkpoint \
  --stop_training
```
The above command creates the new LSTM model from the output after replacing the top layer of the model.

Step 4.
-----
```
combine_tessdata -o ./tessdata/bih.traineddata \
  ~/tesstutorial/bihlayer_from_hin/bihlayer.lstm \
  ~/tesstutorial/bihtest/bih.lstm-number-dawg \
  ~/tesstutorial/bihtest/bih.lstm-punc-dawg \
  ~/tesstutorial/bihtest/bih.lstm-word-dawg 
```  
Finally, the new LSTM model and new dawg files can be combined with the existing Bihari traineddata (hin.traineddata copied as bih.traineddata) in ./tesseract-ocr/tessdata using the above command. The old bih.traineddata file in ./tesseract-ocr/tessdata is renamed.

Step 5.
-----

```
training/lstmeval --model ~/tesstutorial/bihlayer_from_hin/hin.lstm \
  --eval_listfile ~/tesstutorial/bihtest/hin.training_files.txt  
  
training/lstmeval --model ~/tesstutorial/bihlayer_from_hin/bihlayer_checkpoint \
  --eval_listfile ~/tesstutorial/bihtest/hin.training_files.txt  
  
training/lstmeval --model ~/tesstutorial/bihlayer_from_hin/bihlayer.lstm \
  --eval_listfile ~/tesstutorial/bihtest/hin.training_files.txt  
``` 

The above three commands evaluate the LSTM models, 
first with the original Hindi LSTM model copied for Bihari, 
second with the checkpoint model created by replacing top layer and 
third with the Bihari model created from the above checkpoint.

Results of second and 3third lstmeval should be the same. 
Hopefully they are an improvement over the original.

