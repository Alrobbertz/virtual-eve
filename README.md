# sw-irradiance

This is the repository containing code to reproduce the main results from the paper "A Deep Learning Virtual Instrument for Monitoring Extreme UV Solar Spectral Irradiance".  The code included here allows to train the best performing linear+CNN model presented in the paper to map AIA data to 15 channels of EVE MEGS-A spectra, as well as to deploy this model to perform inference on new AIA data.

![alt text](https://github.com/AlexSzen/sw-irradiance/blob/master/AIA_model_EVE.gif)


## Setup the data

- Download data from the stanford repo, which totals 6.5TB. Following are the links to download data for different years. 
  * 2010: https://purl.stanford.edu/vk217bh4910
  * 2011: https://purl.stanford.edu/jc488jb7715
  * 2012: https://purl.stanford.edu/dc156hp0190
  * 2013: https://purl.stanford.edu/km388vz4371
  * 2014: https://purl.stanford.edu/sr325xz9271
  * 2015: https://purl.stanford.edu/qw012qy2533
  * 2016: https://purl.stanford.edu/vf806tr8954
  * 2017: https://purl.stanford.edu/kp222tm1554
  * 2018: https://purl.stanford.edu/nk828sc2920

- [MANUAL] You should now have a folder with AIA data, 39 channels of EVE data, and separate .npy and .csv files for the integrated EVE MEGS-A irradiance.
  - I can't find the EVE Data in any year except 2010.
  - Missing the "integrated EVE MEGS-A irradiance" files for 2011-2014, which are `2011_eve_megsA_mean_irradiance.csv`, `2011_eve_megsA_mean_irradiance.npy`, etc. These are not in the stanford repo, and cannot be found in the original EVE MEGS-A data at `https://lasp.colorado.edu/eve/data_access/`. 

- [MANUAL] Create another folder for data in which we'll create symlinks for AIA images, so as not to mess up your "clean" data folder. For instance call it "data_30mn_cadence" because that's the cadence we'll be working with here.

- [X] In canonical_data/, run `python link_all.py --data clean_data_folder/ --base experimental_data_folder`. You should now have symlinks to AIA data in the experimental data folder.
  - [X] We're going to replace this with a cell in our Jupyter Notebook. 

- [X] In canonical_data/, run `python make_join.py --eve_root clean_data_folder/EVE/np/ --aia_root clean_data_folder/ --target experimental_data_folder/`. In the experimental data folder, you should now have csv files for 2011 through 2014, as well as "iso_10m.npy" and "irradiance_10m.npy".

- [X] Run `python merge_csv.py experimental_data_path/2011.csv experimental_data_path/2012.csv experimental_data_path/2013.csv experimental_data_path/2014.csv experimental_data_path/2011p4.csv` to merge the csv files from 2011 to 2014 into one csv called 2011p4.csv.

We now have a CSV file that puts in correspondence AIA images for a given time step, to 14 channels of EVE MEGS-A spectra for that same time step. 

We now have to take care of the integrated EVE MEGS-A irradiance which will be the 15th channel. This integrated irradiance is contained in the `201x_eve_megsA_mean_irradiance.csv` and `201x_eve_megsA_mean_irradiance.npy` files. We need to join it to the rest of the channels.

- [SKIP] In canonical_data, run `python concat_EVE_arrays_totirr.py --totirr_root clean_data_folder/ --target experimental_data_folder/`.  This script will join the total irradiance arrays for the different years and write it to your experimental data folder.

- [PASSTHROUGH] Run `python join_irradiances_totirr.py --data_root experimental_data_folder/`. You should now have have the files 'irradiance_30mn_14ptot.npy' and 'irradiance_30mn_14ptot.csv' in the experimental data folder. These put in correspondence AIA images for a given time step, with 14 channels of EVE MEGS-A spectra + the channel for integrated MEGS-A irradiance. 

- [X] In canonical_data/, run `python make_splits.py --src experimental_data_folder/irradiance_30mn_14ptot.csv --splits rve` to split the csv file into train test and validation csvs.
  - We now have 3 new CSVs `train.csv`, `val.csv`, and `test.csv` in the experimental data folder. 

- [X] In canonical_data/, run `python make_normalize.py --base experimental_data_folder/ --irradiance experimental_data_folder/irradiance_30mn_14ptot.npy`. This script computes normalization quantities on the training set. It should generate npy files for means and stds of AIA and EVE in your experimental data folder.
```sh
-rw-r--r--   1 andrewrobbertz  staff   200B Jul 17 14:57 aia_mean.npy
-rw-r--r--   1 andrewrobbertz  staff   200B Jul 17 14:56 aia_sqrt_mean.npy
-rw-r--r--   1 andrewrobbertz  staff   200B Jul 17 14:56 aia_sqrt_std.npy
-rw-r--r--   1 andrewrobbertz  staff   200B Jul 17 14:57 aia_std.npy
-rw-r--r--   1 andrewrobbertz  staff   188B Jul 17 14:38 eve_mean.npy
-rw-r--r--   1 andrewrobbertz  staff   188B Jul 17 14:38 eve_sqrt_mean.npy
-rw-r--r--   1 andrewrobbertz  staff   188B Jul 17 14:38 eve_sqrt_std.npy
-rw-r--r--   1 andrewrobbertz  staff   188B Jul 17 14:38 eve_std.npy
```

## Train and test the model

We first need to fit a linear model to output the 15 channels of EVE from the means and stds of the AIA images, with a Huber loss. We can then train a CNN to predict the residuals between EVE and the linear model's predictions of EVE.

- [X] In canonical_code/ run `python setup_residual_totirr --base experimental_data_folder/`. This will fit a linear model using means and stds of AIA images and a Huber loss, then save the means and stds as well as the model.

- [X] In canonical_code/, run `python cfg_residual_unified_train_totirr.py --src path_to_config_files/ --data_root experimental_data_folder/ --target path_to_train_results_folder/`. This will read the configuration JSON file, create the specified model using the specified parameters, and train it. It will save train and val losses as well as best performing model into the --target folder.

- [X] To test the model, run `python cfg_residual_unified_test_totirr.py --src path_to_config_files/ --models path_to_train_results_folder/ --data_root experimental_data_folder/ --target path_to_test_results_folder/ --eve_root clean_data_folder/EVE/np/ --phase test`. This will generate a text file with errors on the test set (or on whichever --phase you specified).

## Use the model for inference

If you want to deploy the model and run inference on new AIA data, you need to do the following.

- Create a data directoy for the year you want to run on, e.g. "data_folder/2015/". 
- In canonical_data/, run "python link_to_all.py --base path_to_clean_data/ --data path_to_year_folder/".
- In canonical_data/, run "python make_csv_inference.csv --data_root year_data_folder/ --target year_data_folder/ --year year_you're_running_on". This will create an index.csv file for that year.
- From the path_to_experimental_data/ you had for the training, you'll need to copy all the normalisation quantities, as well as the linear model. You can do this with "cp path_to_experimental_data/\*.np\* path_to_year_folder/"
- In canonical_code/, run python cfg_residual_unified_inference_totirr.py --src path_to_config_files/ --models path_to_train_results_folder/ --data_root path_to_year_folder/ --target path_to_inference_results_folder/". This will generate a numpy array containing the 15 channels of EVE for each AIA time step in index.csv



