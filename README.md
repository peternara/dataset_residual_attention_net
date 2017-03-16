# Dataset

# Basic usage

A dataset consists of an index - an 1-d sequence with unique keys for each data item -
and a batch class which process the data.

There are some ready-to-use index (e.g. FilesIndex) and batch classes (e.g. ArrayBatch and DataFrameBatch),
but you are likely to need your own class with specific action methods.

So let's define a class to work with your unique data (CT scans)
```python
import numpy as np
import pandas as pd
import dataset as ds

class PatientScans(ds.Batch):
  ...
  @action
  def load(path):
  ...
  @action
  def resize(h, w, d):
  ...
  @action
  def segment_lungs():
  ...
```

The scans are too huge to load into memory at once.
That is why we need a dataset which has an index and a specific action class to process a small subset of data (batch)
```python
CT_SCAN_PATH = '/path/to/CT_scans'
# each CT_SCAN_PATH's subdirectory contains one patient scans and its name is the patient id
patient_index = ds.FilesIndex(CT_SCAN_PATH, dirs=True, sort=False)
scans_dataset = ds.Dataset(index=patient_index, batch_class=PatientScans)
```

In real life we rarely have a clean and ready-to-use dataset. 
Usually we have to make some preprocessing - a workflow with some actions.
That is where the batch class comes into handy.
```python
scans_pp = (ds.Preprocessing(scans_dataset).
                load(path=CT_SCAN_PATH).  # you can build index from one dir and then load data from another dir
                resize(256, 256, 128).    # resize the image
                segment_lungs().          # remove from the image everything except the lungs
                dump('/path/to/processed/scans')   # save preprocessed scans to disk
```
All the actions are lazy so that they are not executed unless their results are needed.

`load`, `resize`, `segment_lungs`, and `dump` are the methods of PatientScans marked with `@action` decorator.
`Preprocessing` knows nothing about your data, where it is stored and how to process it.
It's just a convenient wrapper.

By the way, as nothing has been executed yet, there were no batches either.
Everything is lazy!

So let's run it!
```python
scans_preprocessing.run(batch_size=64, shuffle=False, one_pass=True)
```
Inside `run` the dataset is divided into batches and all the actions are executed for each batch


Moving further, you need to make a combined dataset which contains scans and labels
So we create a dataset with labels as pandas.DataFrame
```python
labels_dataset = ds.Dataset(index=patient_index, batch_class=DataFrameBatch)
# Define how you need to process labels (here you just load them from a file)
labels_processing = Preprocessing(labels_dataset).load('/path/to/labels.csv', fmt='csv')
```

To train the model you might need some data augmentation. Let's define it as a lazy process too.
```python
scan_augm = (Preprocessing(scans_dataset).
               load('path/to/processed/scans').
               random_crop(64, 64, 64).
               random_rotate(-pi/4, pi/4))
```

Now define the combined dataset
```python
full_data = ds.FullDataset(data=scan_augm, target=labels_processing)
```
And again, nothing has been executed or loaded yet.
You don't have to waste CPU or GPU cycles unless you need the processed data.

Before training you might want to split the dataset into train / test / validation subsets
```python
full_data.cv_split([0.7, 0.2])
```

Start the training 
```python
NUM_ITERS = 10000
for i in range(NUM_ITERS):
    # get random batches which contains some augmented scans and corresponding labels
    # all the actions are executed now
    bscans, blabel = full_data.train.next_batch(batch_size=32, shuffle=True)
    # put it into neural network
    sess.run(optimizer, feed_dict={x: bscans, y: blabel['labels'].values}))

```

For more advanced cases see [the documentation](doc/INDEX.md)
