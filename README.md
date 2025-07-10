# portfolio NBAにおけるMVP候補選手の平均スタッツの特徴


目的
- 投票制で決まるMVP選手のスタッツに、投票率に影響を与える指標はあるのか  
- MVP候補とならなかった選手との違いに、ポジションごとにどんな特徴があるのか  

背景
- 小学校から高校卒業までの10年間バスケットボールを経験し、趣味のNBA観戦から興味を持った  
- 大学の卒業研究で「スタッツと勝率の関係性」を分析した延長として、本テーマに着手

データ概要
- NBA_Stats.csv　（1行＝選手の基本情報＋シーズン単位の平均スタッツ)
- MVP_Share.csv　（1行＝選手の基本情報＋そのシーズンのMVP投票割合)

主なカラム一覧

| カラム名     | 型     | 説明                                                                                  
|--------------|--------|--------------------------------------------------------------------------------------
| Player       | 文字列 | 選手名                                                                                 
| Pos          | 文字列 | ポジション（PG, SG, SF, PF, C）                                                        
| PTS          | 数値   | 平均得点                                                                               
| 2P           | 数値   | 平均ツーポイントシュート数                                                             
| 3P           | 数値   | 平均スリーポイントシュート数                                                           
| AST          | 数値   | 平均アシスト                                                                           
| REB          | 数値   | 平均リバウンド                                                                         
| BLK          | 数値   | 平均ブロック                                                                           
| STL          | 数値   | 平均スティール                                                                         
| FG%          | 数値   | フィールドゴール成功率                                                       　　　　  
| 2P%          | 数値   | ツーポイントシュート成功率                                                             
| 3P%          | 数値   | スリーポイントシュート成功率                                                           
| Season       | 文字列 | シーズン                                                               　　　　　　　　
| Share        | 数値   | MVP投票割合                                                               　　　　　　 


開発環境

- Google Colab
  - Python 3.8 
  - pandas 1.3.x
  - numpy 1.21.x
  - scikit-learn 1.0.x
  - sqlite3

- Tableau Public


---

前処理ステップ

1.CSV 読み込み

import sqlite3
import pandas as pd
from google.colab import files

nba_df = pd.read_csv('/content/NBA_Stats.csv')
mvp_df = pd.read_csv('/content/MVP_Stats.csv')


2.メモリ上に SQLite を作成し、データを書き込む

conn = sqlite3.connect(':memory:')
nba_df.to_sql('nba_stats', conn, if_exists='replace', index=False)
mvp_df.to_sql('mvp_stats', conn, if_exists='replace', index=False)


3.投票率は別のcsvファイルにまとめられているためLEFTJOINで Share を結合し、NULLを0に置換
 .MVP候補とそうでない選手を分けられるように列MVP_Nominatedを追加しShare＞0なら1それ以外は0を格納

query = """
SELECT
  n.*,
  COALESCE(m.Share, 0) AS Share,
  CASE
    WHEN COALESCE(m.Share, 0) > 0 THEN 1
    ELSE 0
  END AS MVP_Nominated
FROM nba_stats AS n
LEFT JOIN mvp_stats AS m
  ON n.Player = m.Player;
"""
merged_df = pd.read_sql_query(query, conn)


4.CSV として出力し、ダウンロード
merged_df.to_csv('NBA_Merged.csv', index=False)
files.download('NBA_Merged.csv')


5.データ分析がしやすい数字にしリネーム、条件に「平均プレイ時間(MP)が10分以上」を追加

query = """
SELECT
  Pos AS Position,
  Share AS MVP_Nominated,
  ROUND("FG%" * 100, 2)  AS FG_Pct,
  ROUND("3P%" * 100, 2)  AS "3P_Pct",
  ROUND("2P%" * 100, 2)  AS "2P_Pct",
  ROUND(PTS,  2)  AS PTS,
  ROUND("3P", 2)  AS "3P",
  ROUND("2P", 2)  AS "2P",
  ROUND(AST,  2)  AS AST,
  ROUND(STL,  2)  AS STL,
  ROUND(BLK,  2)  AS BLK,
  ROUND(TRB,  2)  AS REB,
  Player,
  Season,
  ROUND(Share * 100, 2)  AS Vote_Share_Pct,
  ROUND(MP, 2)           AS MP_per_Game
FROM NBA_Merged
WHERE MP >= 10;
"""
result_df = pd.read_sql_query(query, conn)

conn.close()


6.csv形式で出力

result_df.to_csv('nba_filtered.csv', index=False)
files.download('nba_filtered.csv')



Tableau 可視化フロー


1.nba_filtered.csv を読み込み

データソース画面で FG_Pct～REB 列をピボット
　新規列を Stat_Type／Stat_Value にリネーム


2.計算フィールド作成

Stat_Diff（MVP候補−非候補の差分）
Z_Score（標準化値）
Stat Parameter（ドロップダウン切り替え用）
Selected Value（パラメーター連動抽出）
Season_Date（"23-24"を日付型変換）
Position　Highlight（ダッシュボード統合時にハイライト, フィルターするポジション選択を1つにまとめるため）


3.シート作成　計5枚

1.Avg_Comparison … 数値系スタッツの平均比較
2.Avg_Comparison_(PCT) … ％系スタッツの平均比較
3.ZScore_Heatmap … ポジション×スタッツのZスコアヒートマップ
4.Corr_Plot … 投票率 × 選択スタッツの散布図
5.Trend_Linechart … ポジションごとの各スタッツのシーズン推移


4.ダッシュボード統合

1段目：Avg_Comparison /　Avg_Comparison_(PCT) / ZScore_Heatmap
2段目：Corr_Plot /  Trend_Linechart

パラメーター＆フィルターを上部に配置

GitHubのURL：
https://github.com/USHIZAKI-jp/portfolio

Tableau Public の公開 URL：
https://public.tableau.com/views/A_17500823910050/sheet0?:language=ja-JP&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link

