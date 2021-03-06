# 2021/02/09

## Share about split the audio

I train by google colab using [Theo Viel's npz dataset](https://www.kaggle.com/c/rfcx-species-audio-detection/discussion/198048)(32 kHz, 128 mels), because I don't have GPU machine and enough monery to rent Cloud machine.
The dataset can be regarded as 128x3751 size image.

I set the window size to 512 and cut out the entire range of 60's audio data by covering it little by little.
Cover 49 pixels each, considering that important sounds are located at the boundaries of the division.
```python
N_SPLIT_IMG = 8
WINDOW = 512
COVER = 49

slide_img_pos = [[0, WINDOW]]
for idx in range(1, N_SPLIT_IMG):
    h, t = slide_img_pos[idx-1][0], slide_img_pos[idx-1][1]
    h = t - COVER
    t = h + WINDOW
    slide_img_pos.append([h, t])

print(slide_img_pos)
# [[0, 512], [463, 975], [926, 1438], [1389, 1901], [1852, 2364], [2315, 2827], [2778, 3290], [3241, 3753]]
```

I train and predict those images which cut by 128x512.

|idx|pixcel|time(s)|
|--|--|--|
|0|0〜512|0〜8|
|1|463〜975|7〜15|
|2|926〜1438|14〜23|
|3|1389〜1901|22〜30|
|4|1852〜2364|29〜37|
|5|2315〜2827|37〜45|
|6|2778〜3290|44〜52|
|7|3241〜3753|51〜60|

### training phase

Images were cut and padded to train by batch.

```python
def split_and_padding(X, y):
    x_lst = []
    y_lst = []
    for h, t in slide_img_pos:
        _X = X[:, :, :, h:t]
        _y = y[:, :, h:t]
        if _X.shape[3] != WINDOW:
            x_pad = torch.zeros(list(_X.shape[:-1]) + [WINDOW - _X.shape[3]])
            _X = torch.cat([_X, x_pad], axis=3)
            y_pad = torch.zeros(list(_y.shape[:-1]) + [WINDOW - _y.shape[2]])
            _y = torch.cat([_y, y_pad], axis=2)
        x_lst.append(_X)
        y_lst.append(_y)
    
    X = torch.cat(x_lst, axis=0)
    y = torch.cat(y_lst, axis=0)
    return X, y

for X, y in train_data_loader:
    # X shape is (2, 3, 128, 3751) batch_size, channel, image_height, image_width
    # y shape is (2, 24, 3751) batch_size, label, sequence
    X, y = split_and_padding(X, y)
    # X shape is (16, 3, 128, 512)
    # y shape is (16, 24, 512) 
    
    outouts = model(X)
    loss = loss_fn(outouts, y)
      .
      .
      .
```

### prediction phase

I predict each sliding window and extract max-pooling in the frame.
I call this soft frame-wise prediction.

```python
for X, _ in test_data_loader:
    preds = []
    for h, t in slide_img_pos:
        with torch.no_grad():
            outputs = model(X[:,:,:,h:t])
        _p, _ = outputs["segmentwise_output_ti"].sigmoid().max(2)
        preds.append(_p)
    test_pred, _  = torch.max(torch.stack(preds), dim=0)
      .
      .
      .
```

My model has two output, clipwise_output and framewise_output.
I use clipwise_output in training, and I use framewise_output in prediction.
This aproach came from [hinmura0's discussion thread](https://www.kaggle.com/c/rfcx-species-audio-detection/discussion/209684).


---

By the way, we can change this N_SPLIT_IMG, WINDOW, COVER. I tried  N_SPLIT_IMG=16, WINDOW=256, and COVER=23 but not good.

|WINDOW|model|lwlap|precision|recall|LB|memo|
|--|--|--|--|--|--|--|
|512|resnet18|0.9693|0.9590|0.7336|0.936|my best single model|
|256|resnet18|0.9457|0.5744|0.08893|0.921|↑same method other than split size|

## 2nd stage

The purpose of stage2 is to improve model and make pseudo label by this model.

The key point I think is to calculate gradient loss only labeled frame.
The positive label was trained from tp_train.csv only and the negative label was trained from fp_train.csv only.  
I put `1` to positive label and `-1` to negative label.


```python
tp_dict = {}
for recording_id, df in train_tp.groupby("recording_id"):
    tp_dict[recording_id+"_posi"] = df.values[:, [1,3,4,5,6]]

fp_dict = {}
for recording_id, df in train_fp.groupby("recording_id"):
    fp_dict[recording_id+"_nega"] = df.values[:, [1,3,4,5,6]]
    
def extract_seq_label(label, value):
    seq_label = np.zeros((24, 3751))  # label, sequence
    middle = np.ones(24) * -1
    for species_id, t_min, f_min, t_max, f_max in label:
        h, t = int(3751*(t_min/60)), int(3751*(t_max/60))
        m = (t + h)//2
        middle[species_id] = m
        seq_label[species_id, h:t] = value
    return seq_label, middle.astype(int)

# extract positive label and middle point
fname = "00204008d" + "_posi"
posi_label, posi_middle = extract_seq_label(tp_dict[fname], 1) 

# extract negative label and middle point
fname = "00204008d" + "_nega"
nega_label, nega_middle = extract_seq_label(fp_dict[fname], -1) 
```

And, loss function is that:

```python
def rfcx_2nd_criterion(outputs, targets):
    clipwise_preds_att_ti = outputs["clipwise_preds_att_ti"]
    posi_label = ((targets == 1).sum(2) > 0).float().to(device)
    nega_label = ((targets == -1).sum(2) > 0).float().to(device)
    posi_y = torch.ones(clipwise_preds_att_ti.shape).to(device)
    nega_y = torch.zeros(clipwise_preds_att_ti.shape).to(device)
    posi_loss = nn.BCEWithLogitsLoss(reduction="none")(clipwise_preds_att_ti, posi_y)
    nega_loss = nn.BCEWithLogitsLoss(reduction="none")(clipwise_preds_att_ti, nega_y)
    posi_loss = (posi_loss * posi_label).sum()
    nega_loss = (nega_loss * nega_label).sum()
    loss = posi_loss + nega_loss
    return loss
```

### cycle re-labeing

I imporove label by repeat re-labeling and training.

- 2nd-1. train from original tp_train.csv and fp_train.csv
- 2nd-2. train from original tp_train.csv, fp_train.csv and the label which put by 2nd-1 model  
- 2nd-3. train from original tp_train.csv, fp_train.csv and the label which put by 2nd-2 model 

I tried it the 4th time(2nd-4), but it's not good.

I shared `2nd-3` labels to kuto.  
dataset:  
https://github.com/trtd56/RFCX/blob/main/dataset/exp0153_resnet18_focal_mixup_pseudo0.5_thr0.5.pkl  
how to use(sorry, it is written Japanese):  
https://github.com/trtd56/RFCX/blob/main/work/RFCX_PseudoLabel_HowToUse.ipynb  


#### re-labelig

I predict each `slide_img_pos` and put the pseudo label, so I got 8 labels in one 60's file.

- Positive label `1` when 3 or more 5-fold models are predicted to be POSITIVE_THRESHOLD or more.
- Negative label `-1` when 3 or more 5-fold models are predicted to be NEGATIVE_THRESHOLD or more.
- Other than the above `0`

I use pseudo positive label only now.
The pseudo negative label did not work for me.

re-labeling code:  
https://github.com/trtd56/RFCX/blob/main/work/RFCX_re_labeling_by_resnet18.ipynb  
re-labeling model:  
https://drive.google.com/drive/u/1/folders/1ZAiBYUrluhwqeCcsv8aDq4gNuErv4pXt  
each prediction resule(**The pseudolabel file before threshold**):  
https://drive.google.com/drive/folders/1PYAtx9rxgk6tTVt467zcj_BsWLhIVo4v?usp=sharing

I prepared two type pseudo labels clipwise and framewise. We use clipwise label now.
- clipwise label shape is (5090, 8, 24): data, soft-frame, label
- framewise label shape is (5090, 8, 24, 512): data, soft-frame, label, pixcel

## A different approach from kudo

### CV

I use [iterative-stratification](https://github.com/trent-b/iterative-stratification)'s MultilabelStratifiedKFold and random seed is 416.
Validation data is made from tp_train only and fp_train data is used training in all fold.

```python
N_LABEL = 24
SEED = 416

tp_fnames, tp_labels = [], []
for recording_id, df in train_tp.groupby("recording_id"):
    v = sum([np.eye(N_LABEL)[i] for i in df["species_id"].tolist()])
    v = (v  == 1).astype(int).tolist()
    tp_fnames.append(recording_id+"_posi")
    tp_labels.append(v)

fp_fnames, fp_labels = [], []
for recording_id, df in train_fp.groupby("recording_id"):
    v = sum([np.eye(N_LABEL)[i] for i in df["species_id"].tolist()])
    v = (v  == 1).astype(int).tolist()
    fp_fnames.append(recording_id+"_nega")
    fp_labels.append(v)
    
mskf = MultilabelStratifiedKFold(n_splits=5, shuffle=True, random_state=SEED)
for fold, (train_index, valid_index) in enumerate(mskf.split(tp_fnames, tp_labels)):
    train_fname = np.array(tp_fnames)[train_index].tolist() + fp_fnames
    valid_fname = np.array(tp_fnames)[valid_index].tolist()
      .
      .
      .
```

### loss function


There is more positive label than the negative label.I think there is more positive label than the negative label.
I use positive loss decay.  

```diff
+pos_w = 0 if posi_label.sum() == 0 else 1/posi_label.sum()
+loss = posi_loss*pos_w + nega_loss
-loss = posi_loss + nega_loss
```

I don't use kudo's zero loss method for ambiguous labels.

## Part to be improved
- re-labeling fold
  - I'm using the voting for 5-fold models when I put pseudo labels. However, each model also predicts pseudo labels for trained data.
- CV strategy
  - I think now strategy is not good because tp_train and fp_train include duplicate files.
