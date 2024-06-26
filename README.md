# 3d-ink-detection

(translated by ChatGPT)

擴展 ink detection 至 3D ，讓其能夠應用於所有區域和新卷軸。

在這個 repo 中，作者提供了數據準備、訓練和推理管道，還有一個經過非常充分訓練的 3D U-Net 模型，用於檢測 vc 卷軸的 3D 墨水，以及該模型已經運行過的 3D 墨水檢測體積。

## 卷軸1 ink detection 影片
[![卷軸1](https://img.youtube.com/vi/eXHKyKgKAr0/0.jpg)](https://www.youtube.com/watch?v=eXHKyKgKAr0)

## 卷軸2 ink detection 影片（由於推理提前取消，視頻略有截斷）
[![卷軸2](https://img.youtube.com/vi/WvDKq0YaoVA/0.jpg)](https://www.youtube.com/watch?v=WvDKq0YaoVA)

## 卷軸3 ink detection 影片
[![卷軸3](https://img.youtube.com/vi/TFgKJRuvXxU/0.jpg)](https://www.youtube.com/watch?v=TFgKJRuvXxU)

## 卷軸4 ink detection 影片
[![卷軸4](https://img.youtube.com/vi/bs7tYjuGDEo/0.jpg)](https://www.youtube.com/watch?v=bs7tYjuGDEo)

## 數據準備
為了訓練模型，你需要將原始掃描體積轉換為 HDF5 數組。這可以通過對此腳本 https://github.com/ryanchesler/LSM/blob/main/volume_to_hdf.py 進行一些路徑的微小修改來完成。此外，你還需要一些 2D 預測結果以映射回 3D volume。你可以使用任何優秀的 2D 模型的預測結果，在許多實驗中，作者使用了自己的 unetr-> segformer 管道，但另一個起點可以是大獎獲獎倉庫中提供的標籤 https://github.com/younader/Vesuvius-Grandprize-Winner/tree/main/all_labels。要將這些 2D 標籤映射回 3D，你需要目錄中每個部分的 PPM 文件。這些可以從官方數據服務器下載 https://scrollprize.org/data_scrolls#data-server

現在你已經收集好了這些數據，可以運行`python all_labels_to_hdf.py`並對路徑進行一些微小修改。這將初步創建一個新的訓練 HDF5 數組來表示3D中的標籤。為了製作一個驗證數組，我們需要對 all_labels_to_hdf 進行一些微小修改，以便將`20230904135535`設置為保留。重新運行該命令，現在你將擁有訓練和驗證數組。

最終的數據準備步驟是運行pred_find_coords.py。這將掃描標籤體積並記錄實際標記的坐標，我們可以隨機從空間中採樣來找到所有時間點，但更高效的方法是知道坐標並容易訪問有效的訓練點。對訓練體積運行一次，對驗證體積運行一次，並對路徑進行一些微小調整，以獲得代表有效訓練坐標的numpy數組。

![image](https://github.com/ryanchesler/3d-ink-detection/assets/26419230/1da29f11-483b-4099-b938-29c0fb2f2a5d)

## 訓練
現在你擁有了卷軸掃描的HDF5體積、從2D映射到3D的訓練和驗證標籤的HDF5體積以及坐標的numpy數組，你就可以準備進行訓練了。為此需要更新一些路徑並配置wandb。更新到你的新HDF5數組和numpy數組的路徑。關於訓練的幾個細節需要注意。強烈建議使用3D預訓練倉庫的檢查點，在較短的訓練過程中這會有顯著的影響，如果長時間訓練，似乎影響較小。此模型是一個3D U-Net模型，設計成接受128x128x128的輸入體積並輸出128x128x128的墨水檢測體積。其中一個主要困難是我們僅有來自這些2D片段的稀疏標籤數據，這些片段是從3D分割出來的。正確訓練這一點非常重要，將未標記的數據視為未標記數據，而不是因預測沒有墨水而懲罰模型。為此，所有標記區域都會得到實際值，所有未標記的錯誤會被屏蔽為-1。這些值將被排除在損失函數之外，因此不會為它們計算梯度來更新模型。否則模型會偏向總是預測沒有墨水。另一個關鍵因素是正確的驗證。為了確保我們真正能夠測量模型在未見過的卷軸區域上的性能，進行了質量檢查，排除了與任何驗證點在512歐氏距離內的坐標。這是相當激進的，從驗證體積中的任何標記點剝離出完整的512塊，但這對於看到模型在新數據上的實際性能是至關重要的。此代碼使用volumentation來使模型對多種不同的卷軸場景具有魯棒性。旋轉、翻轉、一些強度變化以及裁剪和重新縮放，使其具有一定的尺度不變性。

<img width="1544" alt="Screenshot 2024-04-30 at 11 26 51 PM" src="https://github.com/ryanchesler/3d-ink-detection/assets/26419230/c10bbedd-89ca-459d-ab20-bdfc4f869c41">

一些訓練結果

根據硬件情況，這可能需要數小時才能收斂

## 推理
現在有了一個訓練過的模型或重用共享的檢查點，你就可以對卷軸進行推理了。感興趣的卷軸掃描需要格式化為HDF5，如果在訓練過程中已經完成了這一過程，則可以重用它。如果你想在新的卷軸掃描上使用它，則需要再次運行該過程並更新到所需數組的路徑。現在這個過程將根據卷軸的大小和解析度運行一段時間。推理以滑動窗口方式進行，因為每次僅在128x128x128的大小上訓練。一個更加嚴格的推理可以通過減小步幅來完成，但這會大大增加推理時間。

## 3D預測回到2D
現在有了3D預測，你可以做幾件事。其一是使用現有的段PPM文件，將3D預測重新展平為平面。這將產生類似於這樣的結果

<img width="1617" alt="Screenshot_2024-02-12_at_9 29 38_AM" src="https://github.com/ryanchesler/3d-ink-detection/assets/26419230/4944117e-a3d2-4200-b1d0-8d35c2a7bec5">

這可以使用flatten_3d.py腳本來完成。一些路徑需要更新為預測體積和目標段的PPM文件。

你可以對3D預測進行的另一種操作是檢查連通組件以尋找新字母。connected_components_explore.py腳本將嘗試尋找合理大小的塊並將其導出為多視圖3D圖像，以便快速翻閱。這是發現未探索部分卷軸中新字母的好方法。到目前為止，它在訓練過的卷軸1中發現了新內容，但在卷軸2-4中的結果尚無定論。

<img width="745" alt="Screenshot_2024-03-07_at_1 03 53_PM" src="https://github.com/ryanchesler/3d-ink-detection/assets/26419230/e5ebc7fb-e302-40a5-b599

-800a2ae89446">

也可以將預測體積的子集導出為jpg或其他文件格式，並使用slicer的圖像堆棧擴展進行可視化。這提供了許多檢查和探測預測結果的工具。

<img width="793" alt="Screenshot_2024-03-07_at_12 49 16_PM" src="https://github.com/ryanchesler/3d-ink-detection/assets/26419230/a7f249e6-2fa8-4cf5-af93-774076794307">
