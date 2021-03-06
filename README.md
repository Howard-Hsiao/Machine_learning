# OpenAI Gym challenges
## Team member
* [邱廷翔](https://github.com/lewiechiu)
* [蕭昀豪](https://github.com/Howard-Hsiao)
* 陳柄瑞

## Abstract
從openai gym多樣化的遊戲環境，揀選CartPole-v1、LunarLander-v2、BipedalWalker-v2。為上述遊戲各自實作有能力自行破關的遊戲機器人。其中CartPole-v1成功破關。LunarLander-v2由原本平均得分-200的狀態提升至+20、BipedalWalker-v2則尚未有重大突破。

## Projects
### Cartpole-v1
<img src="./img/cartpole.jpg" alt="cartpole" width="400">

CartPole是OpenAI Gym中最簡單的一個遊戲，遊戲目標是藉由移動車子來保持上方木棒平衡
#### 環境:
此遊戲的observation只有4種、action只有2種(控制車子左右)。

#### 策略:
首先用隨機選取action的方式進行遊戲，若分數超過某定值則將當時環境中的observation、當下做出的action記下。重複多次後將這些資料當作訓練資料訓練一個神經網路。最後進行遊戲時，把環境中的observation當作輸入，用訓練出的神經網路預測action進行遊戲。

#### 成果:
成功將此遊戲破關。(此遊戲破關的定義為：連續195場，拿下195分以上)<br>
[破關程式碼](https://nbviewer.jupyter.org/github/lewiechiu/MLDL/blob/master/openai_gym_cartpole.ipynb?fbclid=IwAR1360xysqwyp49tu9EfHYwlDZW0umu0IQEoRcwpg7WLTFiwEFQFHO6XLjg)

### LunarLander-v2
<img src="./img/lunarlander.jpg" alt="lunarlander" width="400" >

LunarLander-v2亦為OpenAI Gym中的遊戲，遊戲方式為控制太空船不同方向的引擎，使太空船順利降落在兩旗竿之間。

#### 環境:
此遊戲的observation有8種、action有4種(控制各方向引擎)。
相較於Bi-pedalwalker，Lunar-lander的observation較少，且4種action皆為離散值。

我們的目標為利用

1. 當下模型狀態state
2. 採取動作action
3. 從(state、action)獲得的反饋reward
上述3種已知資訊創建一個遊戲模型，使其經由輸入當下狀態，得到最佳動作，並在此模型基礎嘗試deep Q learning、CNN等技巧。
輸入:當下狀態(state)、採取動作(action)
輸出:在state下採取action能得到的反饋(reward)

<img src="./img/our_lunar_solution1.jpg" alt="our_lunar_solution1" width="500">

#### 程式發展歷程:

在遊戲過程，我們希望當輸入state, action資訊時，我們的machine learning model能告知我們最佳動作。為此，我們搭建一模型，其架構如上圖所示。每一次當我們向模型輸入state及action，model便會返還該次行動的reward(reward越大代表該行動越好)。因此，我們僅須在接收到當下狀態時，將其配合動作集內所有動作個別輸入，找出能產出最大reward的組合，該組合對應的動作就是我們想要的最佳action。值得一提的是，我們利用functional API特性為模型實作雙輸入特性。

* [with 10 round per train](https://github.com/lewiechiu/MLDL/blob/master/Lunar%20Landing%20Final%20Project/DQN-10.ipynb)
* [with 10 round using CNN](https://nbviewer.jupyter.org/github/lewiechiu/MLDL/blob/master/Lunar%20Landing%20Final%20Project/DQN%20functional%20Input%20CNN%20state.ipynb)
* [model with Deep Q Learning(未獲得顯著提升)](https://nbviewer.jupyter.org/github/lewiechiu/MLDL/blob/master/Lunar%20Landing%20Final%20Project/DQN-10%20DDQN.ipynb)

#### 在基本模型之上嘗試的額外特性
##### Target Network and Engine Network (2 NNs)
在training以及predicting stage過程，我們採用兩個NN，理由如下。

* ###### 加快遊戲速度
在2NN架構下，Engine Network負責利用實際數據training，另一方面，Target Network 負責predict結果。 相比於原本每做一個動作train一次，目前兩個神經網路這樣的架構可以讓遊玩50場的時間落在1分鐘以內(在GPU:Tesla P100的環境運作)，相較原本最多1 分鐘內玩10 場的train and play at the same time策略，採行2NNs大大增加遊玩效率。

* ###### 解決Overfitting
另外一個我們採用兩個相同神經網路，但是不同步的更新策略原因是想要避免overfitting。在原本單神經網路的環境下，模型會針對過去所有的memory一而再再而三地訓練。如此一來，神經網路將無法用更general的模式去學出我們想要approximate的Q-table。然而假如我們為了解決上述問題，每次只將模型新預測出的資料擬合入模型之中，模型將有很大概率出現嚴重偏誤。為了避免該問題，當engine network每train一次更新完參數，我們就把DQN agent裡原有的memory全部洗掉，讓target engine可以根據更新後的參數，繼續蒐集新的記憶。如此一來，模型每一次train的內容都將根據不同的training data進行擬合。

##### Deep Q Learning簡介
為了讓我們的模型能夠不短視近利，我們導入Q function處理模型的輸出，也就是各個動作能得到的reward。
具體實作方式為引入定義域為[0, 1]的gamma值，代表模型重視未來選擇的程度。其中當gamma為0時，代表模型完全忽視未來選擇的影響，而gamma為1時，表示模型重視現在與未來選擇的程度相同。當我們欲擬合輸出資訊時，假設該筆紀錄的done不為False，也就是該動作之後仍有未來(下一局)，就將reward加上gamma * model.predict(next_state)。最後將得出的結果擬合入模型，即完成Deep Q Learning流程。

最後結果 -> 未導入Deep Q Learning能獲得較佳結果

#### 專案進行時曾遇過的難關
實作LunarLander-v2時，我們原先搭建的神經網絡是以state為輸入來預測action，但如此一來神經網絡僅能學會「隨機生成資料時，各種state與action的隨機組合」，也就是在這個神經網絡沒有reward參與的情況下，沒有評斷action的標準。因此神經網絡學不會「怎樣才是好的應對方式」。目前這個問題已經被我們在上述的解法中修正。

[錯誤程式碼-遺漏變數](https://nbviewer.jupyter.org/github/lewiechiu/MLDL/blob/master/Lunar%20Landing%20Final%20Project/50d250d250d2.ipynb)

#### 成果:
解法2成功通關，解法1失敗

### BipedalWalker-v2
<img src="./img/bi_pedal_walker.jpg" alt="bi_pedal_walker" width="400">

Bi-pedalwalker是OpenAI Gym中另一個較複雜的遊戲，遊戲方式為控制機器人腳步，使得機器人能走越遠越好。

#### 環境
此遊戲的observation共計23種、action則有4種(控制機器人腳步)

#### 改進演算法歷程
我們實作BipedalWalker-v2的過程可分為以下3階段：
1. 發現誤把Time當作reward
2. 發現遺漏變數

首先我們使用與進行Cartpole時一樣的方式進行遊戲，一開始即取得良好成果。然而當使用env.render()指令將遊戲畫面視覺化時，我們發現因為我們誤把遊戲時間當成reward，導致機器人的遊戲策略為「如何能存活最久」，而非「如何走得最遠」。最終結果為機器人選擇按兵不動，藉此避免跌倒以延長存活時間。

將reward修正後，我們發現執行結果依然不甚理想，在train的過程中，walker的表現沒有辦法往合理的方向收斂。持續對演算法進行偵錯，我們發現現今演算法在訓練資料時僅以state及reward進行擬合。缺少了action作為輸入，如此產生出的模型在我們進行預測時自然不可能知道在狀態state下進行各action的好壞。

#### 錯誤排除歷程
* [錯把生存時間作為reward的錯誤演算法](https://nbviewer.jupyter.org/github/lewiechiu/MLDL/blob/master/Bipedal%20walker%20.ipynb)
* [遺漏變數](https://nbviewer.jupyter.org/github/lewiechiu/MLDL/blob/master/Bipedal-v2.ipynb)

#### 成果
尚未成功。BipedalWalker-v2為我們額外練習的遊戲，可惜因為時間不足，沒有額外時間調整deep Q learning架構使其符合BipedalWalker-v2環境
