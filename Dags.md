# [DAGs](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html)

DAG（Directed Acyclic Graph、有向非循環グラフ）はAirflowの中心的な概念で、[タスク](./Tasks.md)を集め、依存関係と関係性に基づいて、どのようにタスクを実行するかを定義します。

以下は基本的なDAGの例です：

![](https://airflow.apache.org/docs/apache-airflow/stable/_images/basic-dag.png)

このDAGは、A、B、C、Dの4つのタスクを定義し、それらが実行される順番や、どのタスクがどのタスクに依存しているかを決めます。また、DAGがどの頻度で実行されるかも指定します。例えば、「明日から5分ごとに実行する」や「2020年1月1日から毎日実行する」といった形です。

DAG自体はタスク内で何が行われるかには関心を持ちません。DAGは単にタスクをどの順番で実行するか、何回リトライするか、タイムアウトがあるかなど、実行方法に関心を持っています。

## DAGの宣言方法

DAGは以下の3つの方法で宣言できます。

一つ目は`with`文（コンテキストマネージャ）を使う方法で、これにより`with`の内部で定義された内容が暗黙的にDAGに追加されます：

```python
import datetime
from airflow import DAG
from airflow.operators.empty import EmptyOperator

with DAG(
    dag_id="my_dag_name",
    start_date=datetime.datetime(2021, 1, 1),
    schedule="@daily",
):
    EmptyOperator(task_id="task")
```

二つ目は標準のコンストラクタを使用し、DAGを任意のオペレーターに渡す方法です：

```python
import datetime
from airflow import DAG
from airflow.operators.empty import EmptyOperator

my_dag = DAG(
    dag_id="my_dag_name",
    start_date=datetime.datetime(2021, 1, 1),
    schedule="@daily",
)
EmptyOperator(task_id="task", dag=my_dag)
```

三つ目は`@dag`デコレータを使用して、[関数をDAG生成器に変換する方法](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#concepts-dag-decorator)です：

```python
import datetime
from airflow.decorators import dag
from airflow.operators.empty import EmptyOperator

@dag(start_date=datetime.datetime(2021, 1, 1), schedule="@daily")
def generate_dag():
    EmptyOperator(task_id="task")

generate_dag()
```

DAGは、実行する[タスク](./Tasks.md)がなければ意味がありません。タスクは通常、[オペレーター](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/operators.html)、[センサー](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/sensors.html)、または[タスクフロー](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/taskflow.html)として提供されます。

### タスクの依存関係

タスクやオペレーターは通常単独で存在することはなく、他のタスクに依存しています（上流のタスクや下流のタスク）。タスク間の依存関係を宣言することで、DAG構造（有向非循環グラフの辺）が形成されます。

タスクの依存関係を宣言する方法は主に2つあります。推奨される方法は`>>`および`<<`オペレーターを使う方法です：

```python
first_task >> [second_task, third_task]
third_task << fourth_task
```

また、`set_upstream`と`set_downstream`メソッドを使って、より明示的に依存関係を宣言することもできます：

```python
first_task.set_downstream([second_task, third_task])
third_task.set_upstream(fourth_task)
```

さらに、より複雑な依存関係を宣言するためのショートカットもあります。複数のタスクが別のタスクに依存するようにしたい場合、前述の方法ではなく、`cross_downstream`を使う必要があります：

```python
from airflow.models.baseoperator import cross_downstream

# 次の依存関係を作成
# [op1, op2] >> op3
# [op1, op2] >> op4
cross_downstream([op1, op2], [op3, op4])
```

依存関係をチェーンのようにつなげる場合は、`chain`を使用します：

```python
from airflow.models.baseoperator import chain

# 次の依存関係を作成
# op1 >> op2 >> op3 >> op4
chain(op1, op2, op3, op4)

# 動的にチェーンを作成することも可能
chain(*[EmptyOperator(task_id='op' + str(i)) for i in range(1, 6)])
```

チェーンは、リストのサイズが同じ場合にペアワイズで依存関係を作成することもできます（これは`cross_downstream`で作成される交差依存関係とは異なります）：

```python
from airflow.models.baseoperator import chain

# 次の依存関係を作成
# op1 >> op2 >> op4 >> op6
# op1 >> op3 >> op5 >> op6
chain(op1, [op2, op3], [op4, op5], op6)
```

## DAGの読み込み

Airflowは、設定された`DAG_FOLDER`内からPythonのソースファイルを探し、DAGを読み込みます。各ファイルを実行して、DAGオブジェクトを読み込みます。

これにより、1つのPythonファイル内に複数のDAGを定義することができたり、非常に複雑なDAGを複数のPythonファイルに分割し、インポートを使って管理することも可能です。

ただし、AirflowがPythonファイルからDAGを読み込む際には、そのファイルのトップレベルにあるDAGインスタンスのみを対象とします。例えば、以下のDAGファイルを見てみましょう：

```python
dag_1 = DAG('this_dag_will_be_discovered')

def my_function():
    dag_2 = DAG('but_this_dag_will_not')

my_function()
```

このファイルがアクセスされると、`dag_1`と`dag_2`の両方のDAGコンストラクタが呼び出されますが、`dag_1`のみがトップレベル（`globals()`）にあるため、Airflowに追加されるのは`dag_1`のみです。`dag_2`は読み込まれません。

> [!NOTE]  
> Airflowが`DAG_FOLDER`内でDAGを検索する際、最適化のために「`airflow`」と「`dag`」という文字列（大文字小文字を区別しません）が含まれるPythonファイルのみを対象にします。  
> もしすべてのPythonファイルを対象にしたい場合は、`DAG_DISCOVERY_SAFE_MODE`設定フラグを無効にしてください。

また、DAG_FOLDERやそのサブフォルダ内に`.airflowignore`ファイルを配置することで、読み込まないファイルのパターンを指定できます。この設定はそのディレクトリとその下のすべてのサブフォルダに適用されます。[`.airflowignore`](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#concepts-airflowignore)ファイルの構文については以下で説明しています。

もし`.airflowignore`ファイルが目的に合わない場合や、PythonファイルがAirflowによって解析されるべきかどうかをより柔軟に制御したい場合は、設定ファイルで`might_contain_dag_callable`を設定することができます。このコールバック関数は、Airflowのデフォルトの判定方法（Pythonファイルに`airflow`および`dag`という文字列が含まれているかどうか）を置き換えます。

```python
def might_contain_dag(file_path: str, zip_file: zipfile.ZipFile | None = None) -> bool:
    # file_path内にDAGが定義されているかどうかを確認するロジック
    # Trueを返すと、そのファイルは解析される
    # Falseを返すと、そのファイルは解析されない
```

## DAGの実行

DAGは主に2つの方法で実行されます：

- 手動またはAPIを介してトリガーされた場合
- DAGで定義されたスケジュールに従って実行される場合

DAGにはスケジュールが必須ではありませんが、スケジュールを定義することは非常に一般的です。スケジュールは、`schedule`引数を使用して次のように定義できます：

```python
with DAG("my_daily_dag", schedule="@daily"):
    ...
```

`schedule`引数にはさまざまな有効な値があります：

```python
with DAG("my_daily_dag", schedule="0 0 * * *"):
    ...

with DAG("my_one_time_dag", schedule="@once"):
    ...

with DAG("my_continuous_dag", schedule="@continuous"):
    ...
```

> [!TIP]  
> スケジュールに関する詳細な情報については、「[Authoring and Scheduling](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/index.html)」を参照してください。

DAGを実行するたびに、そのDAGの新しいインスタンスが作成され、これをAirflowでは「[DAG Run](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dag-run.html)」と呼びます。同じDAGでも複数のDAG Runが並行して実行されることがあり、それぞれに「データ間隔」が定義され、このデータ間隔がタスクが処理すべきデータの期間を特定します。

これがなぜ便利かを具体例で説明します。例えば、日々の実験データを処理するDAGがあるとしましょう。そのDAGが改修され、過去3ヶ月分のデータで実行したい場合でも、AirflowはDAGをバックフィルし、過去3ヶ月の日々のデータを一度に実行することができます。

これらのDAG Runはすべて同じ実際の日に開始されますが、各DAG Runにはその3ヶ月間の中で1日をカバーするデータ間隔が割り当てられ、そのデータ間隔がDAG内のタスク、オペレーター、センサーが実行時に参照する対象となります。

DAGが実行されるたびにDAG Runがインスタンス化されるのと同様に、DAG内で定義されたタスクもそれに応じて「[タスクインスタンス](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html#concepts-task-instances)」としてインスタンス化されます。

DAG Runには開始日と終了日があり、この期間がDAGが実際に「実行された」時間を示します。DAG Runの開始日と終了日とは別に、「論理日付（正式にはexecution date）」という別の日付が存在します。この論理日付は、DAG Runがスケジュールまたはトリガーされた時点を示します。この日付が「論理日付」と呼ばれる理由は、その抽象的な性質にあり、DAG Runの文脈によって意味が異なるからです。

例えば、DAG Runがユーザーによって手動でトリガーされた場合、その論理日付はDAG Runがトリガーされた日付と時刻となり、その値はDAG Runの開始日と一致します。しかし、DAGが自動的にスケジュールされ、特定のスケジュール間隔が設定されている場合、論理日付はデータ間隔の開始時刻を示し、そのDAG Runの開始日は「論理日付 + スケジュール間隔」となります。

> [!TIP]
> `logical date`に関する詳細な情報については、「[Data Interval](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dag-run.html#data-interval)」と「[What does execution_date mean?](https://airflow.apache.org/docs/apache-airflow/stable/faq.html#faq-what-does-execution-date-mean)」を参照してください。
