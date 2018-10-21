# SageMaker is ?

バッチ学習タイプの機械学習プロジェクトをプロトタイピング〜サービス運用まで、一気通貫で回すためのプラットフォーム。

機械学習自体があまり良くわからない、という方へ	

<iframe src="https://docs.google.com/presentation/d/e/2PACX-1vQVA-jfUWZ95JLGkoqMtrRI7Q6hSZ0nJIU9di7y-Q8ME3BG02oZ5e_LD7VttQnlpHrLJ6jWVlXOQKhl/embed?start=false&loop=false&delayms=3000" frameborder="0" width="640" height="389" allowfullscreen="true" mozallowfullscreen="true" webkitallowfullscreen="true"></iframe>

[Slide](https://drive.google.com/file/d/1VVYj-OcKxOqVeA7lo0V_ENyeoCYNVCe5/view?usp=sharing)

ざっくりまとめると、SageMakerが提供する機能は次の3つ。

- Managed jupyter
- Ondemand Training resource
- Managed Prediction Endpoint service

## Managed jupyter

AWSがマネージする環境で、Jupyterを利用可能。kernelはいくつかプリセットが用意されており、ユーザーはこれらを適宜選択してnotebookを開始できる。

jupyter環境へのアクセス制限はIAMを介して制御する。SGではないため注意。

実行環境のバリエーション(e.x: Python 2 or 3, TensorFlow or MXNet)がある程度揃っており、それらの環境を管理する必要がない点がユーザーにとって利点となる。

プリセットの環境で不足がある場合でも、notebook上のマジックコマンドで _pip install_ を行ったり、SageMakerの機能である _Lifecycle Configuration_ を用いることで(おそらく)任意のPython	パッケージは利用可能。

参考: [AWS Machine Learning Blog - Customize your Amazon SageMaker notebook instances with lifecycle configurations and the option to disable internet access
](https://aws.amazon.com/jp/blogs/machine-learning/customize-your-amazon-sagemaker-notebook-instances-with-lifecycle-configurations-and-the-option-to-disable-internet-access/)

## Ondemand Training resource

実体はDockerコンテナであり、entrypointを通してトレーニングジョブを実行する。リソースはトレーニングのジョブを走らせた時のみ起動する。同一アカウント内の別ユーザーの使用状況にも影響を受けないため、必要に応じて必要なだけのコンピューティングリソースが手に入る。オンプレミスでの分析環境と比較すると、クラウドの利点はそのままSageMakerのセールスポイントになる。

ユーザーは必要とするリソース量や入出力先(S3)の設定などを行い、「ジョブ」を定義する。コンピューティングリソースの管理（ローンチ処理と後片付け）はSageMakerが代行してくれる。

また、SageMakerがBuilt-inでサポートする分析手法(+実装)を用いた場合はコンテナイメージの開発すら不要。やりたいことが達成できるのであればBuild-inを用いる方が良い。SageMakerのお作法に適合するコードを書く必要はあるが、それを差し引いても手間が少ない。

どのような手法がBuilt-inサポートされているかは、[公式ドキュメント - Using Built-in Algorithms with Amazon SageMaker
](https://docs.aws.amazon.com/sagemaker/latest/dg/algos.html) や [Blackbelt](https://www.slideshare.net/AmazonWebServicesJapan/20180308-aws-black-belt-online-seminar-amazon-sagemaker-90045719/23) を参考に。

Built-inで満足できない場合は自前でコンテナ定義を行い、任意のトレーニングジョブを実行することができる。データの最終的な入出力はS3で固定される。コンテナ起動時に、SageMakerプラットフォームは指定されたS3 keyからデータをダウンロードし、コンテナ内の特定のパスに配置する。ユーザーのトレーニング用コードは、そのディレクトリからのデータ読み出しという形で入力を受け取る。

出力時も同様で、SageMakerの仕様として規定される特定パスに対してartifact（シリアライズされたモデルなど）を書き込むことで、SageMakerはそのパスにあるファイルをS3に対してputする。**コンテナ内において、データの入出力は通常のファイルシステムへの入出力とまったく変わらない**。

※S3にダミーのファイルを配置して、実際のデータソースはコンテナ内でRedShiftに接続しに行って取得する...といったハックっぽいやり方も可能。SageMaker的にはあまり推奨ではないと思われます


## Managed Prediction Endpoint service

完成したモデルを実サービスとして利用していくためには、外部に対してpredictionのインタフェースを提供する必要がある。機械学習とは関係のないエンジニアリング領域のタスクであるため、多くの分析担当者にとっては非常に重たいか、実施困難な工程になる。逆に、エンジニアから見ても機械学習まわりのコードはロジックが見えず手がつけづらい可能性があり、このタスクに対する分業は課題となる。SageMakerは、予測用APIの提供や、裏側のホスティング用リソースの管理を引き受ける。

こちらも実体はDockerである。SageMaker Built-inのアルゴリズムを利用する限り、ユーザーはDockerまわりを意識する必要が(ほとんど)ない。

モデルのデプロイは容易であり、差し替え時にダウンタイムがないことが特長。

エンドポイントの裏側に複数モデル(≒docker)を立てることが可能。これによってA/Bテストを行うことができる。
