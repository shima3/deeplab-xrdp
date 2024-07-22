# deeplab-xrdp

# はじめに

DeepLab v3+ を用いた画像の領域分割（semantic segmentation）のためのイメージです。
学習済みモデルを用いたサンプル画像の領域分割、競技用データ集を用いた学習モデルの作成、自作データ集を用いた学習モデルの作成ができます。

# サーバの起動

以下のコマンドでサーバを起動します。
```
docker run --name deeplab-xrdp --mount type=volume,src=etc,dst=/etc -p 3389:3389 --device /dev/fuse --cap-add SYS_ADMIN -d --shm-size="1gb" --restart always --mount type=volume,src=home,dst=/home --hostname deeplab-xrdp kshima/deeplab-xrdp sleep infinity
docker exec deeplab-xrdp service xrdp start
```
次のコマンドでユーザを追加し、パスワードを設定します。
```
docker exec deeplab-xrdp useradd -m -s /bin/bash ユーザ名
docker exec deeplab-xrdp passwd ユーザ名
New password: パスワード
Retype new password: パスワード
```

# クライアントの接続

Windowsのリモートデスクトップ接続や、その互換アプリ（Mac用Microsoft Remote Desktopなど）を使ってサーバに接続します。
ユーザ名とパスワードを入力してログインすると Ubuntu のデスクトップが表示されます。
以降、MenuのシステムツールにあるMATE端末を使ってコマンドを実行します。
端末のメニューバーにあるEditを開くとコピペが使えます。
デスクトップにある「ユーザ名's Home」をダブルクリックすると、ホームディレクトリが開きます。

# 学習済みモデルを用いたサンプル画像の領域分割

## Step 1
testフォルダで作業を行うため、端末で次のコマンドを実行してください。
```
cd
mkdir -p test
cd test
```
ホームディレクトリを開いて、testフォルダがあればOKです。

## Step 2
次のコマンドで、サンプル画像を取得します。
```
cp -r /opt/tensorflow/models/research/deeplab/g3doc/img JPEGImages
```
testフォルダを開くと、JPEGImagesフォルダが作成されています。
JPEGImagesフォルダを開いて、image1.jpg, image2.jpg, image3.jpgがあればOKです。

## Step 3
次のコマンドで、学習済みモデルを取得します。
```
cp -r /opt/tensorflow/models/research/deeplab/datasets/pascal_voc_seg/init_models/deeplabv3_pascal_train_aug TrainedModel
```
testフォルダを開くと、TrainedModelフォルダが作成されています。
TrainedModelフォルダを開いて、frozen_inference_graph.pbがあればOKです。

## Step 4
次のコマンドで、サンプル画像を領域分割します。
```
predict.sh
```
testフォルダを開くと、PredictedSegmentationフォルダが作成されています。
PredictedSegmentationフォルダを開いて、image1.png, image2.png, image3.pngがあればOKです。

領域の種類（class）を示す色をラベル、ラベルで領域を示す画像をラベル画像と呼びます。
test/JPEGImagesのimage1.jpg, image2.jpg, image3.jpgに対応するラベル画像がtest/PredictedSegmentationのimage1.png, image2.png, image3.pngに保存されます。
ラベルと色との対応は、<a href="https://qiita.com/otakoma/items/2f40e583980013acb2f7">こちら</a>を参照してください。

# 競技用データ集を用いた学習モデルの作成

## Step 1
VOC2012という競技で用いられたデータを取得するため、次のコマンドで実行してください。
```
cd
cp -r /opt/VOCdevkit/VOC2012 .
```
ホームディレクトリにVOC2012フォルダが作成されます。
VOC2012フォルダにJPEGImagesフォルダとSegmentationClassフォルダがあります。
JPEGImagesフォルダに写真約1万7千枚、SegmentationClassにラベル画像約3千枚があればOKです。

## Step 2
ラベル画像をグレースケール画像に変換するため、次のコマンドを実行してください。
```
cd VOC2012
remove_gt_colormap.sh
```
VOC2012のSegmentationClassRawに画像約3千枚ができます。
グレースケール画像の黒い範囲は輝度でインデックス値を示しています。
インデックス値とラベル（色）の対応については、<a href="https://qiita.com/mine820/items/725fe55c095f28bffe87">こちら</a>を参照してください。
グレースケール画像の白い線（輝度255）は、対応するラベルがないことを示しています。

## Step 3
グレースケール画像をTFRecords形式に変換するため、次のコマンドを実行してください。
```
build_voc2012_data.sh --image_format="jpg"
```
VOC2012のTFRecordsにtrain-0000で始まるファイルが4つあればOKです。

## Step 4
学習させる前に、TrainingLogフォルダの有無を確認してください。
TrainingLogフォルダがあると、前回の学習の続きから始めるので、学習が早く終わる代わりに、前回の学習の影響を受けます。
元画像やラベル画像を変更した場合などで、最初からやり直したい時はTrainingLogフォルダを削除してから学習させてください。

学習させるため、次のコマンドを実行してください。
```
time train.sh --step=30
```
上のコマンドの --step=30 で学習回数を指定しています。
ここでは学習済みモデルを利用しているので、少ない回数でも高い精度を得られますが、通常は数千回の学習が必要です。

最初に大量の警告メッセージが出ますが無視してください。
次のようなメッセージが出始めたら、学習が始まっています。
```
INFO:tensorflow:global step 10: loss = 0.1566 (3.232 sec/step)
```
step の後の数値が増えていけば、学習が進んでいます。
--stepで指定した回数に達すると学習が終了し、次のようなメッセージが出ればOKです。
```
INFO:tensorflow:Finished training! Saving model to disk.
```
このメッセージの後に数行の警告メッセージが表示されますが無視してください。

## Step 5
学習済みのモデルを外部で利用できるようにするため、次のコマンドを実行してください。
```
export_model.sh
```
VOC2012のTrainedModelにfrozen_inference_graph.pbがあればOKです。
testの中のTrainedModelを、ここで作ったTrainedModelに置き換えてから、「学習済みモデルを用いたサンプル画像の領域分割」のStep 4を行うと、領域分割できます。

# 自作データ集を用いた学習モデルの作成

DeepLab フォルダで作業するため、ターミナルで次のコマンドを実行してください。
```
cd $HOME/DeepLab
```

## Step 1
JPEGImagesフォルダに学習用の元画像をJPEG形式で保存してください。

## Step 2
SegmentationClassRawフォルダに学習用のグレースケール画像をPNG形式で保存してください。
1から20のインデックス値を選び、グレースケール画像の輝度値として設定してください。

## Step 3
グレースケール画像をTFRecords形式に変換するため、次のコマンドを実行してください。
```
docker run --rm -it --volume $PWD/JPEGImages:/opt/tensorflow/models/research/deeplab/datasets/pascal_voc_seg/VOCdevkit/VOC2012/JPEGImages --volume $PWD/SegmentationClassRaw:/opt/tensorflow/models/research/deeplab/datasets/pascal_voc_seg/VOCdevkit/VOC2012/SegmentationClassRaw --volume $PWD/TFRecords:/opt/tensorflow/models/research/deeplab/datasets/pascal_voc_seg/tfrecord kshima/deeplab bash build_voc2012_data.sh --image_format="jpg"
```
上のコマンドの --image_format="jpg" でJPEGファイルの拡張子を指定しています。
拡張子が .jpeg の場合、 --image_format="jpeg" としてください。
コマンドの実行終了後、tfrecordにtrain-0000で始まるファイルが4つあればOKです。

## Step 4
学習させる前に、TrainingLog フォルダの有無を確認してください。
TrainingLog フォルダがあると、前回の学習の続きから始めるので、早く終わりますが、前回の影響を受けます。
元画像やラベル画像を変更した場合などで、最初からやり直したい時は TrainingLog フォルダを削除してから学習させてください。

学習させるため、次のコマンドを実行してください。
```
docker run -it --volume $PWD:/opt/tensorflow/models/research/deeplab/datasets/pascal_voc_seg/VOCdevkit/VOC2012 --volume $PWD/TFRecords:/opt/tensorflow/models/research/deeplab/datasets/pascal_voc_seg/tfrecord --volume $PWD/TrainingLog:/opt/tensorflow/models/research/deeplab/datasets/pascal_voc_seg/exp/train_on_trainval_set/train kshima/deeplab bash train.sh --step=30
```
上のコマンドの --step=30 で学習回数を指定しています。
ここでは学習済みモデルを利用しているので、少ない回数でも高い精度を得られますが、通常は数千回の学習が必要です。

最初に大量の警告メッセージが出ますが無視してください。
次のようなメッセージが出始めたら、学習が始まっています。
```
INFO:tensorflow:global step 10: loss = 0.1566 (3.232 sec/step)
```
step の後の数値が増えていけば、学習が進んでいます。
--stepで指定した回数に達すると学習が終了し、次のようなメッセージが出ればOKです。
```
INFO:tensorflow:Finished training! Saving model to disk.
```
このメッセージの後に数行の警告メッセージが表示されますが無視してください。

## Step 5
学習済みのモデルを外部で利用できるようにするため、次のコマンドを実行してください。
```
docker run -it --volume $PWD:/opt/tensorflow/models/research/deeplab/datasets/pascal_voc_seg/VOCdevkit/VOC2012 --volume $PWD/TFRecords:/opt/tensorflow/models/research/deeplab/datasets/pascal_voc_seg/tfrecord --volume $PWD/TrainingLog:/opt/tensorflow/models/research/deeplab/datasets/pascal_voc_seg/exp/train_on_trainval_set/train --volume $PWD/TrainedModel:/opt/tensorflow/models/research/deeplab/datasets/pascal_voc_seg/exp/train_on_trainval_set/export kshima/deeplab bash export_model.sh
```
コマンドの実行終了後、TrainedModelフォルダの中にfrozen_inference_graph.pbがあればOKです。

## Step 6
DeepLab/testの中のTrainedModelを、ここで作ったTrainedModelに置き換えてから、「学習済みモデルを用いたサンプル画像の領域分割」のStep 4を行うと、領域分割できます。
