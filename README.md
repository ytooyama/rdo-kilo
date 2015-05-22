# rdo-kilo

###これはなに
RDO PackstackでOpenStack Kiloの色々な環境を作る手順書のようなものです。といってもまだ書き始めたばかりです。

###環境について
以下の環境でPackstackコマンドを実行して環境を作ります。

- CentOS 7以降

そのうち、もう少しいろいろなディストリビューションで動作確認します。

OpenStack Compute上では上記以外のLinuxディストリビューションも動作します。

###動作可否状況(2015/05/22現在)

構成             | CentOS 7.1   | Fedora 21   | Fedora 22   
--------------- | ------------ | ----------- | ----------- 
All-in-One      | OK           | TBD         | TBD        
Multi-3Node     | OK           | TBD         | TBD        
Other Case      | TBD          | TBD         | TBD        

Info:
現時点ではFedora 22ではPuppetスクリプトに問題があるようで動かないようです（[この辺り](https://www.redhat.com/archives/rdo-list/2015-May/msg00216.html)を参照）。

###RDOってなに？

[公式サイト](https://www.rdoproject.org/Main_Page)をご覧ください。

