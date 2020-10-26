---
title: "PostGIS 3.0で消えた関数"
emoji: "😀"
type: "tech"
topics: [PostGIS]
published: false
---
# はじめに

PostGIS 3.0では、以前は有効であったけれども削除された関数があります。

削除対象となったものは、以前に関数名の命名規則が変更されたことに伴い、非推奨関数化されたものです。突然に消えたものではなく、先に非推奨関数の注意書きが付されるようになり、マニュアルから削除され、しばらくしてから削除されたものです。

この影響で、PostGISを使用するソフトウェアでも古いバージョンのものは、関数が見つからない致命的なエラーを発生させるので、正常に動作しなくなります。

お使いのアプリケーションがPostGISのバージョンを挙げた途端に正常に動作しなくなった場合に、もしかしたら助けになるかも知れませんので、ここに記しておきます。

## 対象

PostGIS 3.0.0と PostGIS 2.5.3で比較しています。それぞれのSQLファイルから<code>CREATE OR REPLACE FUNCTION</code>を行頭に取る行を<code>grep</code>で抜き出しています。ラスタ関数も混じるので<code>rast</code>を含む行をはじきました（それでも若干残りました）。

ラスタ関数かどうかはざっと見ただけで判断しているので、結果に入れるべき関数を落としている可能性があります。

# 一覧

## "_"が消えた関数

次に示す関数は、代替関数名で"_"が消えています。

| 消えた2.5での関数 | 対応する3.0での関数 |
|------------------|-------------------|
| ST_Combine_BBox | ST_CombineBbox |
| ST_distance_sphere | ST_DistanceSphere |
| ST_distance_spheroid | ST_DistanceSpheroid |
| ST_find_extent | ST_FindExtent |
| ST_force_2d | ST_Force2D |
| ST_force_3d | ST_Force3D |
| ST_force_3dm | ST_Force3DM |
| ST_force_3dz | ST_Force3DZ |
| ST_force_4d | ST_Force4D |
| ST_force_collection | ST_ForceCollection |
| ST_length_spheroid | ST_LengthSpheroid |
| ST_line_interpolate_point | ST_LineInterpolatePoint |
| ST_line_locate_point | ST_LineLocatePoint |
| ST_line_substring | ST_LineSubstring |
| ST_mem_size | ST_MemSize |
| ST_point_inside_circle | ST_PointInsideCircle |
| ST_Shift_Longitude | ST_ShiftLongitude |

## "_"だけでなく"measure"が消えた関数

次に示す関数は、代替関数名で"_"だけでなく"measure"も消えています。いずれも、M値を持つジオメトリを引数に取る関数です。

| 消えた2.5での関数 | 対応する3.0での関数 |
|------------------|-------------------|
| ST_locate_along_measure | ST_LocateAlong |
| ST_locate_between_measures | ST_LocateBetween |

## Length系関数

ST_3DLength_spheroidの代替関数名で"3D"が消えていて、また、ST_length2d_spheroidは、代替関数がありません。

理由は調べていないのですが、ST_length2d_spheroidは、ST_LengthSpheroidに与えるジオメトリをあらかじめST_Force2D等で2次元化することで対応できるので、ST_length2d_spheroidとST_3DLength_spheroidとを統合させたのではないかとにらんでいます。

| 消えた2.5での関数 | 対応する3.0での関数 |
|------------------|-------------------|
| ST_3DLength_spheroid | ST_LengthSpheroid |
| ST_length2d_spheroid |  |

# おわりに

今回は、3.0.0で本当に削除されてしまった非推奨関数を見てみました。

ざっと見て頂いたら分かると思いますが、3.0.0で削除された関数のほとんどは、"\_"を消すと対応できます（"ST\_"の"\_"は除く）。

# 本記事のライセンス

![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by/4.0/88x31.png)
この記事は [クリエイティブ・コモンズ 表示 4.0 国際 ライセンス](http://creativecommons.org/licenses/by/4.0/">) の下に提供されています。
