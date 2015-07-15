# RDO-Kilo

###これはなに
RDO PackstackでOpenStack Kiloの色々な環境を作る手順書のようなものです。といってもまだ書き始めたばかりです。

###環境について
最小インストールした以下の環境でPackstackコマンドを実行して環境を作ります。

- CentOS 7/7.1
- Fedora 21

そのうち、もう少しいろいろなディストリビューションで動作確認します。

OpenStack Compute上では上記以外のLinuxディストリビューションも動作します。

###動作可否確認状況(2015/07/14現在)

構成             | CentOS 7.1   | Fedora 21   | Fedora 22   
--------------- | ------------ | ----------- | ----------- 
All-in-One      | OK           | TBD         | ERR        
One Node        | OK           | OK          | TBD        
Multi-3Node     | OK           | TBD         | TBD        
Other Case      | TBD          | TBD         | TBD        

All-in-One:

[公式の手順](https://www.rdoproject.org/Quickstart)に従ってインストールできた

One Node:

[1-1の手順](1-1-RDO-QuickStart-Local.md)で動作した

Multi-3Node:

3台構成を構築して動作した

Info:

特になし

###RDOってなに？

[公式サイト](https://www.rdoproject.org/Main_Page)をご覧ください。

