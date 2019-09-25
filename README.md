# Kaggle-kkbox-churn-prediction-top5-percentage
[Kaggle-kkbox-churn-prediction](https://www.kaggle.com/c/kkbox-churn-prediction-challenge) <br>
E-mail : [yltsai0609@gmail.com](yltsai0609@gmail.com) <br>
 **********************************************
 結論 : KKBOX顧客流失預測挑戰賽共提供6張Tabel，檔案大小總和超過40G，
       透過GCP的雲端環境可以突破記憶體的限制進行分析建模，
       你可以在這裡找到搭建GCP環境的方法 : <br>
* [Initializing Projects of GCP and BigQuery Using Cloud Shell](https://medium.com/@yulongtsai/datalab-and-bigquery-to-analytics-d0802782d9bb) 
* [Analyzing KKBOX User_logs Using BigQuery and Datalab ](https://medium.com/@yulongtsai/datalab-bigquery-python-kkbox-churn-prediction-f2a7245c5d99) <br>
       本次預測主要使用了RandomForest及XGBoost並搭配資料清理及特徵工程，
       最終僅使用8樣特徵及達到Top5%，你可以在這裏找到過程的詳細解說 : 
* [Project-KKBOX churn prediction Top5% Based on GCP Environment](https://medium.com/@yulongtsai/kaggle-kkbox-churn-prediction-top5-c0ea4c9b3f1a)<br>
* 本次分析logloss -> 0.11, Top5%-Lift 9.2
* 你可以在這裡找到模型成效解釋[[模型解釋策略]Lift Chart, Permutation Importance, LIME](https://medium.com/@yulongtsai/lift-chart-permutation-importance-lime-c22be8bdaf48)

 **********************************************
<b>以下會針對目標、指標、資料清理、特徵工程等部分進行說明</b>
### 目標 

* 預測下個月使用者是否會流失
流失定義 : 方案到期之後沒有續約，且沒有在30天內進行續訂
舉例 : 

User A 在3月被標記為流失 --> User A 在 0104日會員到期，沒有進行續訂，且在0203之前沒有進行續約

User B 在3月被標記為流失 --> User B 在 0131日會員到期，沒有進行續訂，且在0302之前沒有進行續約

User C 在3月被標記為**留存** --> User C 在 0115日會員到期，沒有進行續訂，但在0212續訂了

### 指標
* logloss 
* Top5-Lift [你可以在這裡閱讀Top5-Lift應用於本專案的結果](https://medium.com/@yulongtsai/lift-chart-permutation-importance-lime-c22be8bdaf48)

### 原始資料
#### user_label

|特徵|描述|
|----|----|
|msno|編碼後使用者id|
|is_chrun|流失與否|

#### user_logs
* 共兩張 user_logs, user_logs_v2

|特徵|描述|
|----|----|
|date|使用者聽歌日期(日)|
|num_25|25%之內卡歌-歌曲數|
|num_50|50%之內卡歌-歌曲數|
|num_75|75%之內卡歌-歌曲數|
|num_985|98.5%之內卡歌-歌曲數|
|num_100|100%聽完-歌曲數|
|num_unq|獨立歌曲數|
|total_secs|總秒數|

#### transactions
* 共兩張 transactions, transactions_v2

特徵|描述|
|----|----|
|payment_method_id|編碼後付費方式|
|payment_plan_days|方案天數|
|plan_list_price|方案價格|
|actual_amount_paid|實付價格|
|transaction_date|交易日期|
|membership_expire_date|該交易之會員到齊日|
|is_cancel|是否為取消交易|
|is_auto_renew|是否為自動更新|

#### members

特徵|描述|
|----|----|
|city|註冊城市|
|bd|註冊年齡|
|gender|註冊性別|
|registered_via|經由?裝置註冊(編碼後)|
|registration_init_time|首次註冊日期|


### 資料清洗
|特徵|解釋|資料表|動機|
|-------|---|-----|----|
|mem_expire_date|合約到期日|transactions|有許多不合理的合約到期日，例如1971年, 2020年|
|bd|使用者註冊時年齡|members|年齡有低於0歲及超過100歲，我們過濾了0~95歲，其餘設為 -1，並且發現年齡不合理的紀錄有更低的流失率|
<br>
交易日期作為判斷使用者行為的重要基礎，有進行清洗的必要，並且我們發現，若該交易紀錄為取消的紀錄(is_cancel = 1)，到期日多半和交易日相同，或是到期日在交易日後5天之內，佔了不少比例。
這很合理，畢竟如果是取消，則在取消之後幾天內到期，給個緩衝。

#### 清洗方式
定義 合約天數    =    會員到期日     -  交易日期 <br>
     membership = mem_expire_date - trans_date <br>

|合約天數|處理|
|-------|---|
|x < -30 | 根據該交易是否為取消做近一步處理|
|-30 < x < 450| 無法判斷原因，當作噪聲(noise)處理|
| x > 450| 根據該交易是否為取消做進一步處理|

|情況|解釋|處理|
|---|---|----|
|is_cancel = 1|該交易為取消交易|修正到期日為交易日|
|is_cancel = 0|該交易<b>非</b>取消交易|按照原始合約天數，實際所付價格，推斷到期日|

### 特徵工程 
本次建模並沒有使用相當多的特徵，原本6張Tabel合併之後原始特徵就有20+項，然而大部分都沒有提供什麼訊息，
不過透過使用者ID(msno)，交易日期(trans_date)，上線聽歌紀錄(record)，
我們可以取出相當細緻的使用者行為，除了按照使用者ID將所有交易總和，聽歌記錄總和，得知對kkbox的依賴度之外，
我們亦可以透過時間窗口的大小(週，雙週，月，兩個月......)等來捕捉使用者行為，下表為特徵工程的構想動機，8項特徵之中就有6項與時間窗口有關 : 

|特徵|解釋|資料表|動機|
|-------|----|----|-----|
|six_month_day_listen|過去六個月內的使用者聽歌次數|User_logs|使用者在特定時間區間內的行為|
|six_month_user_latent_satisfaction|過去六個月內的聽歌滿意度|User_logs|使用者在特定時間區間內的行為|
|one_month_day_listen|過去一個月內的使用者聽歌次數|User_logs|使用者在特定時間區間內的行為|
|registered_via|(推測)經由?裝置註冊|members|使用者的經濟水準|
|age_under_26|年齡是否小於26歲|members|使用者的經濟水準|
|last_last_churn|前一個月是否有流失|transactions|使用者在特定時間區間內的行為|
|client_level_code|使用者續約次數|transactions|使用者整體行為|
|last_auto_renew|上次交易是否為自動更新|transactions|使用者在特定時間區間內的行為|


### 資料處理及效能比較
* [Concatenation of Bigquery Table](https://github.com/YLTsai0609/Kaggle-kkbox-churn-prediction-top5-percentage/blob/master/BQ%20Tables%20concat.ipynb)
* [Comparison Getting Dataframe Using bigquery and using bigquery and storage streaming](https://github.com/YLTsai0609/Kaggle-kkbox-churn-prediction-top5-percentage/blob/master/efficiency%20testing%20reading%20dataframe%20from%20bigquery.ipynb)

### 模型解釋
* [Lift Chart, Permutation Importance LIME](https://medium.com/@yulongtsai/lift-chart-permutation-importance-lime-c22be8bdaf48)
