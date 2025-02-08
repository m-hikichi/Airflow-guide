# [XCom](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/xcoms.html)

XCom（クロス・コミュニケーションの略）は、[タスク](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html)同士がデータをやり取りできる仕組みです。デフォルトでは、タスクは完全に隔離されており、異なるマシンで実行されることもあります。

XComは、キー（実質的にはその名前）、タスクID、DAG IDによって識別されます。値としては、シリアライズ可能な任意のデータ型を使うことができます（例えば、`@dataclass`や`@attr.define`で装飾されたオブジェクトなど）。ただし、XComは少量のデータをやり取りすることを目的としており、大きな値（データフレームなど）を渡すためには使わないでください。

XComは、`xcom_push`と`xcom_pull`メソッドを使って、タスクインスタンスから明示的に「プッシュ」および「プル」されます。

### XComに値をプッシュする例
例えば、「task-1」というタスクで別のタスクが使用するデータをプッシュする場合：

```python
# "identifier as string"というキーでデータをXComにプッシュ
task_instance.xcom_push(key="identifier as string", value=any_serializable_value)
```

### 別のタスクでプッシュされた値をプルする例
上記のコードでプッシュした値を別のタスクでプルする場合：

```python
# "task-1"タスクから"identifier as string"というキーでXCom変数をプル
task_instance.xcom_pull(key="identifier as string", task_ids="task-1")
```

多くのオペレーターは、`do_xcom_push`引数が`True`（デフォルト）に設定されている場合、その結果を「return_value」というXComキーに自動的にプッシュします。`@task`関数も同様に動作します。`xcom_pull`メソッドは、キーが指定されていない場合、デフォルトで`return_value`を使用するため、以下のように書くことができます：

```python
# "pushing_task"からreturn_value XComをプル
value = task_instance.xcom_pull(task_ids='pushing_task')
```

XComは[テンプレート](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/operators.html#concepts-jinja-templating)内でも利用できます：

```sql
SELECT * FROM {{ task_instance.xcom_pull(task_ids='foo', key='table_name') }}
```

XComは、[Variables](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/variables.html)の親戚のようなものです。主な違いは、XComはタスクインスタンスごとにあり、DAGの実行内での通信を目的としている点です。一方、Variablesはグローバルであり、全体的な設定や値の共有を目的としています。

複数のXComを一度にプッシュしたり、プッシュするXComのキー名を変更したい場合は、`do_xcom_push`および`multiple_outputs`引数を`True`に設定し、値の辞書を返すことができます。

**注意点**

最初のタスク実行が成功しない場合、リトライ時にはXComがクリアされ、タスクが再実行しても同じ結果が得られるように処理されます。

## オブジェクトストレージ XComバックエンド

デフォルトのXComバックエンドは、`BaseXCom`クラスで、XComはAirflowデータベースに保存されます。これは少量のデータには問題ありませんが、大きなデータや多数のXComには問題が生じることがあります。

XComをオブジェクトストレージに保存したい場合、`xcom_backend`設定オプションを`airflow.providers.common.io.xcom.backend.XComObjectStorageBackend`に設定します。さらに、`xcom_objectstorage_path`に保存先のパスを設定する必要があります。接続IDは、提供するURLのユーザー部分から取得できます（例：`xcom_objectstorage_path = s3://conn_id@mybucket/key`）。また、`xcom_objectstorage_threshold`は-1より大きい値に設定する必要があります。バイト単位でこの閾値より小さいオブジェクトはデータベースに保存され、閾値より大きいオブジェクトはオブジェクトストレージに保存されます。この設定でハイブリッド構成が可能です。オブジェクトストレージに保存されたXComには、データベースに参照が保存されます。最後に、`xcom_objectstorage_compression`に`gzip`や`snappy`など、fsspecでサポートされている圧縮方法を設定することで、オブジェクトストレージに保存する前にデータを圧縮することができます。

例えば、以下の設定では、1MB以上のデータをS3に保存し、gzipで圧縮します：

```ini
[core]
xcom_backend = airflow.providers.common.io.xcom.backend.XComObjectStorageBackend

[common.io]
xcom_objectstorage_path = s3://conn_id@mybucket/key
xcom_objectstorage_threshold = 1048576
xcom_objectstorage_compression = gzip
```

**注意点**

圧縮を使用するには、Python環境に対応するライブラリがインストールされている必要があります。例えば、`snappy`圧縮を使用するには`python-snappy`をインストールする必要があります。`zip`、`gzip`、`bz2`は標準でサポートされています。

## カスタムXComバックエンド

XComシステムには交換可能なバックエンドがあり、`xcom_backend`設定オプションを使って使用するバックエンドを設定できます。

独自のバックエンドを実装したい場合は、`BaseXCom`をサブクラス化し、`serialize_value`および`deserialize_value`メソッドをオーバーライドする必要があります。

また、XComオブジェクトがUIやレポーティング目的でレンダリングされる際に呼ばれる`orm_deserialize_value`メソッドもあります。もしXComに大きな値や取得にコストがかかる値が含まれている場合、このメソッドをオーバーライドして、そのコードを呼び出さずに軽量で不完全な表現を返すようにすることで、UIがスムーズに動作するようにできます。

さらに、`clear`メソッドをオーバーライドして、指定したDAGやタスクの結果をクリアする際に使用できます。これにより、カスタムXComバックエンドはデータライフサイクルの処理を簡素化できます。

## コンテナ内でのカスタムXComバックエンドの確認

Airflowがどこにデプロイされているか（ローカル、Docker、K8sなど）に応じて、カスタムXComバックエンドが実際に初期化されているかを確認するのが有益です。例えば、コンテナ環境の複雑さにより、カスタムバックエンドがコンテナデプロイ中に正しく読み込まれているか確認するのが難しくなることがあります。幸いにも、以下の方法を使ってカスタムXCom実装の信頼性を確認できます。

Airflowコンテナ内でターミナルにアクセスできる場合、使用中のXComクラスをプリントすることができます：

```python
from airflow.models.xcom import XCom

print(XCom.__name__)
```