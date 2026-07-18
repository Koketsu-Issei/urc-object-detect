# URC物体検出

## 概要

Intel RealSenseとYOLOv8を使用して、ROS2上で物体検出を行う。
・ブリックハンマー
・オレンジハンマー
・ペットボトル、
・アルコマーカー
を検出する。
ブリックハンマーは精度がかなり低くなっている。

## １．CUDA対応PyTorchの確認
```
python3 -c "import torch; print('PyTorch:', torch.__version__); print('CUDA:', torch.cuda.is_available())"
```
正常な例：
PyTorch: 〇〇
CUDA: True
CUDA: True なら、JetsonのGPUをPyTorchから使用できる。

## ２．Ultralyticsの確認
```
python3 -c "from ultralytics import YOLO; print('Ultralytics OK')"
```
以下が表示されたらUltralyticsのインストール済みであるため、再インストールは不要。多分入ってる。
```
Ultralytics OK
```
以下が表示された場合Ultralyticsのインストールに進む。
```
ModuleNotFoundError: No module named 'ultralytics'
```
### Ultralyticsのインストール
```
python3 -m pip install --user --no-deps ultralytics
```
インストール後の確認。
```
python3 -c "from ultralytics import YOLO; print('Ultralytics OK')" python3 -c "import torch; print('CUDA:', torch.cuda.is_available())"
```
Ultralyticsを二重でインストールすると環境が壊れる可能性がある。

## 3. RealSenseの確認
RealSense SDKが導入済みか確認する。
```
realsense-viewer
```
映像が表示される場合、RealSense SDKの再インストールは不要。多分入ってる。
次に、ROS 2用RealSenseパッケージを確認する。
```
ros2 pkg prefix realsense2_camera
```
パスが表示された場合はインストール済み。多分入ってる。
パッケージが見つからない場合のみ、ROS 2 Humble用RealSenseパッケージをインストールする。
```
sudo apt update
sudo apt install ros-humble-realsense2-*
```
インストール確認
```
ros2 pkg list | grep realsense
```
次のようなパッケージが表示される。
```
realsense2_camera
realsense2_camera_msgs
realsense2_description
```

## 4.リポジトリのクローン
best.ptはGitLFSで管理されされている。
Jetson側では、クローンする前にGit LFSを入れる。
```
sudo apt update
sudo apt install -y git-lfs
git lfs install
```
ワークスペースの作成
```
mkdir -p ~/camera_ws/src
cd ~/camera_ws/src
```
リポジトリをクローン
```
git clone https://github.com/Koketsu-Issei/urc-object-detect.git
```
モデルが取得できているか確認。
```
ls -lh ~/camera_ws/src/urc-object-detect/yolov8_ros2/models/best.pt
```
成功していたら以下の様に表示される。
```
-rw-rw-r-- 1 USERNAME USERNAME 6.0M  7月 18 13:08 /home/USERNAME/camera_ws/src/urc-object-detect/yolov8_ros2/models/best.pt
```
ファイルサイズが数百バイト程度の場合は、Git LFSの本体ではなく「参照用の小さいファイル」しか取得できていない可能性がある。その場合は次を実行する。
```
cd ~/camera_ws/src/urc-object-detect
git lfs pull
```

## 5.モデルパスの設定
学習済みモデルは、次の場所に配置する。はじめから配置されているはず。
```
~/camera_ws/src/urc-object-detect/yolov8_ros2/models/best.pt
```
object_detection.pyで、best.ptの場所を指定する。
object_detection.py 17行目
```
self.detection_model = YOLO("/home/bossketsu/camera_ws/src/urc-object-detect/yolov8_ros2/models/best.pt")
```
これをjetson内のbest.ptの正しい位置に書き換える。
```
self.detection_model = YOLO("/home/jetsonのユーザー名/camera_ws/src/urc-object-detect/yolov8_ros2/models/best.pt")
```
これで合ってるはず。
ユーザー名の確認は
```
whoami
```
## 6.依存パッケージのインストール
```
cd ~/camera_ws

source /opt/ros/humble/setup.bash

rosdep update
rosdep install --from-paths src --ignore-src -r -y
```

## 7.ビルド
```
cd ~/camera_ws
source /opt/ros/humble/setup.bash
colcon build --symlink-install
source ~/camera_ws/install/setup.bash
```
パッケージを確認する。
```
ros2 pkg list | grep yolov8_ros2
```
次のように表示されれば正常。
```
yolov8_ros2
```

## 8.ノードの実行
カメラの起動
```
source /opt/ros/humble/setup.bash
ros2 launch realsense2_camera rs_launch.py
```
YOLO物体検出ノードの実行
2つ目のターミナルを開く。
```
cd ~/camera_ws
source /opt/ros/humble/setup.bash
source ~/camera_ws/install/setup.bash
ros2 run yolov8_ros2 object_detection \
  --ros-args \
  -r image_raw:=/camera/camera/color/image_raw
```

## 9.新しいターミナルを開いた場合
新しいターミナルを開くたびに、次の2つを実行する。
```
source /opt/ros/humble/setup.bash
source ~/camera_ws/install/setup.bash
```
毎回入力するのが面倒な場合は、.bashrcに追加できる。
```
echo 'source /opt/ros/humble/setup.bash' >> ~/.bashrc
echo 'source ~/camera_ws/install/setup.bash' >> ~/.bashrc
source ~/.bashrc
```
