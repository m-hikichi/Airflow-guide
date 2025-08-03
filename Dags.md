# [DAGs](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html)

DAG（Directed Acyclic Graph、有向非循環グラフ）はAirflowの中心的な概念で、[タスク](./Tasks.md)を集め、依存関係と関係性に基づいて、どのようにタスクを実行するかを定義します。

以下は基本的なDAGの例です：

![](https://airflow.apache.org/docs/apache-airflow/stable/_images/basic_dag.png)

このDAGは、A、B、C、Dの4つのタスクを定義し、それらが実行される順番や、どのタスクがどのタスクに依存しているかを決めます。また、DAGがどの頻度で実行されるかも指定します。例えば、「明日から5分ごとに実行する」や「2020年1月1日から毎日実行する」といった形です。

DAG自体はタスク内で何が行われるかには関心を持ちません。DAGは単にタスクをどの順番で実行するか、何回リトライするか、タイムアウトがあるかなど、実行方法に関心を持っています。

## DAGの宣言方法

DAGは以下の3つの方法で宣言できます。

一つ目は`with`文（コンテキストマネージャ）を使う方法で、これにより`with`の内部で定義された内容が暗黙的にDAGに追加されます：

```python
 import datetime

 from airflow.sdk import DAG
 from airflow.providers.standard.operators.empty import EmptyOperator

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

 from airflow.sdk import DAG
 from airflow.providers.standard.operators.empty import EmptyOperator

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

from airflow.sdk import dag
from airflow.providers.standard.operators.empty import EmptyOperator

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
from airflow.sdk import cross_downstream

# 次の依存関係を作成
# [op1, op2] >> op3
# [op1, op2] >> op4
cross_downstream([op1, op2], [op3, op4])
```

依存関係をチェーンのようにつなげる場合は、`chain`を使用します：

```python
from airflow.sdk import chain

# 次の依存関係を作成
# op1 >> op2 >> op3 >> op4
chain(op1, op2, op3, op4)

# 動的にチェーンを作成することも可能
chain(*[EmptyOperator(task_id='op' + str(i)) for i in range(1, 6)])
```

チェーンは、リストのサイズが同じ場合にペアワイズで依存関係を作成することもできます（これは`cross_downstream`で作成される交差依存関係とは異なります）：

```python
from airflow.sdk import chain

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

## DAGの割り当て

すべての **Operator（演算子）/Task（タスク）** は、**必ずDAGに割り当てられている必要があります**。割り当てられていないと、実行されません。Airflow では、DAGを明示的に渡さなくても、自動でDAGを計算するいくつかの方法があります：

* `with DAG` ブロックの中で Operator を定義した場合
* `@dag` デコレーターの中で Operator を定義した場合
* DAG が割り当てられている別の Operator を `upstream`（前段）または `downstream`（後段）に指定した場合

それ以外の場合は、**`dag=` パラメータで明示的にDAGを各Operatorに渡す必要があります**。

## デフォルト引数

DAG内の多くのOperatorは、**同じデフォルト引数**（例：リトライ回数など）を必要とすることがよくあります。これを毎回個別に設定する代わりに、DAGの作成時に `default_args` を渡すことで、それに紐づくすべてのOperatorに自動で適用されます。

```python
import pendulum

with DAG(
    dag_id="my_dag",
    start_date=pendulum.datetime(2016, 1, 1),
    schedule="@daily",
    default_args={"retries": 2},
):
    op = BashOperator(task_id="hello_world", bash_command="Hello World!")
    print(op.retries)  # 2
```

## DAG デコレーター

従来の `with DAG` ブロックや `DAG()` コンストラクタを使ったDAG定義に加えて、**関数に `@dag` デコレーターを付けることで、DAGを生成する関数として定義することができます**。

```python
from typing import TYPE_CHECKING, Any
import httpx
import pendulum

from airflow.models.baseoperator import BaseOperator
from airflow.providers.standard.operators.bash import BashOperator
from airflow.sdk import dag, task

if TYPE_CHECKING:
    from airflow.sdk import Context


class GetRequestOperator(BaseOperator):
    """指定されたURLにGETリクエストを送信するカスタムオペレーター"""

    template_fields = ("url",)

    def __init__(self, *, url: str, **kwargs):
        super().__init__(**kwargs)
        self.url = url

    def execute(self, context: Context):
        return httpx.get(self.url).json()


@dag(
    schedule=None,
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
    tags=["example"],
)
def example_dag_decorator(url: str = "http://httpbin.org/get"):
    """
    IPアドレスを取得して、BashOperatorで表示するDAG

    :param url: IPアドレスを取得するためのURL（デフォルト: "http://httpbin.org/get"）
    """
    get_ip = GetRequestOperator(task_id="get_ip", url=url)

    @task(multiple_outputs=True)
    def prepare_command(raw_json: dict[str, Any]) -> dict[str, str]:
        external_ip = raw_json["origin"]
        return {
            "command": f"echo '今日、Airflowを実行しているサーバーはIP {external_ip} から接続されています'",
        }

    command_info = prepare_command(get_ip.output)

    BashOperator(task_id="echo_ip_info", bash_command=command_info["command"])

example_dag = example_dag_decorator()
```

DAG を簡潔に作成できる新しい方法であると同時に、**関数内で定義した任意のパラメータを DAG のパラメータとして設定**することもできます。これにより、[DAG をトリガーする際にこれらのパラメータを設定することができ](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dag-run.html#dagrun-parameters)、Python コード内から、あるいは [Jinja テンプレート](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/operators.html#concepts-jinja-templating)内の `{{ context.params }}` からアクセスできます。

> [!NOTE]
> Airflow は DAG ファイルの[トップレベル]()に現れる DAG のみを読み込みます。
> つまり、関数に `@dag` を付けるだけでは不十分で、**その関数を少なくとも1回呼び出し、DAG ファイル内のトップレベルオブジェクトに代入する必要があります**。上記の例でそれが行われていることがわかります。

## 制御フロー

デフォルトでは、**DAGは依存するすべてのタスクが成功したときにのみ、タスクを実行します**。ただし、これを変更する方法はいくつかあります：

* **[ブランチ処理](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#concepts-branching)** – 条件に基づいて、どのタスクに進むかを選択する
* **[トリガールール](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#concepts-trigger-rules)** – タスクを実行する条件を設定する
* **[セットアップおよびティアダウン](https://airflow.apache.org/docs/apache-airflow/stable/howto/setup-and-teardown.html)** – セットアップ・後処理の関係性を定義する
* **[Latest Only](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#concepts-latest-only)** – 現在実行されている DAG のみでタスクを実行する特殊なブランチ処理
* **[Depends On Past](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#concepts-depends-on-past)** – タスクが前回の実行結果に依存できるようにする

### ブランチ処理

ブランチ処理を使うことで、すべての下流タスクを実行するのではなく、**一部のパスだけを選択して実行させることができます**。ここで `@task.branch` デコレーターが登場します。

`@task.branch` デコレーターは `@task` と似ていますが、**デコレーターが付けられた関数は、タスクID（またはそのリスト）を返すことが求められます**。指定されたタスクが実行され、それ以外のパスはスキップされます。`None` を返すことで、すべての下流タスクをスキップすることも可能です。

Python関数から返される `task_id` は、**`@task.branch` が付けられたタスクの直接下流である必要があります**。

> [!NOTE]
> あるタスクが、ブランチオペレーターの下流であり、かつ選択されたタスクの下流でもある場合、**そのタスクはスキップされません**：
> ![](https://airflow.apache.org/docs/apache-airflow/stable/_images/branch_note.png)
> ブランチタスクのパスは `branch_a`、`join`、および `branch_b` です。`join` は `branch_a` の下流タスクであるため、**ブランチの選択結果として返されなかった場合でも実行されます**。

`@task.branch` は **XCom と併用することで、上流タスクの結果に基づいて動的にブランチを決定**することもできます：

```python
@task.branch(task_id="branch_task")
def branch_func(ti=None):
    xcom_value = int(ti.xcom_pull(task_ids="start_task"))
    if xcom_value >= 5:
        return "continue_task"
    elif xcom_value >= 3:
        return "stop_task"
    else:
        return None


start_op = BashOperator(
    task_id="start_task",
    bash_command="echo 5",
    do_xcom_push=True,
    dag=dag,
)

branch_op = branch_func()

continue_op = EmptyOperator(task_id="continue_task", dag=dag)
stop_op = EmptyOperator(task_id="stop_task", dag=dag)

start_op >> branch_op >> [continue_op, stop_op]
```

分岐機能を持つ独自のオペレーターを実装したい場合は、`BaseBranchOperator` を継承することができます。これは `@task.branch` デコレーターと同様の動作をしますが、**`choose_branch` メソッドの実装を提供することが求められます**。

> [!NOTE]
> `@task.branch` デコレーターの使用が推奨されており、`BranchPythonOperator` の直接インスタンス化は推奨されません。後者は、**カスタムオペレーターの実装時にのみ継承して使うべきです**。

`@task.branch` の呼び出し可能オブジェクトと同様に、このメソッドは**下流タスクの ID、またはそのリストを返すことができ、それらのタスクが実行され、それ以外はスキップされます**。また、`None` を返すことで**すべての下流タスクをスキップすることも可能です**。

```python
class MyBranchOperator(BaseBranchOperator):
    def choose_branch(self, context):
        """
        月初めの日に追加のブランチを実行する
        """
        if context['data_interval_start'].day == 1:
            return ['daily_task_id', 'monthly_task_id']
        elif context['data_interval_start'].day == 2:
            return 'daily_task_id'
        else:
            return None
```

通常の Python コード用の `@task.branch` デコレーターと同様に、**仮想環境を使用する `@task.branch_virtualenv` や、外部の Python 実行環境を使用する `@task.branch_external_python` といったブランチ用のデコレーターも存在します**。

### Latest Only

Airflow の DAG 実行（DAG Run）は、**現在の日付とは異なる日付に対して実行されることがよくあります**。たとえば、過去1か月分のデータをバックフィルするために、毎日1回ずつ DAG を実行する場合などです。

ただし、DAG の一部またはすべてのパーツを **過去の日付に対しては実行したくない場合**があります。そのようなケースでは、`LatestOnlyOperator` を使用することができます。

この特殊なオペレーターは、**現在が「最新」の DAG 実行でない場合、下流のすべてのタスクをスキップします**（つまり、現在の壁時計の時刻がその `execution_time` と次のスケジュールされた `execution_time` の間にあり、かつ外部トリガーによる実行でない場合）。

```python
import datetime
import pendulum

from airflow.providers.standard.operators.empty import EmptyOperator
from airflow.providers.standard.operators.latest_only import LatestOnlyOperator
from airflow.sdk import DAG
from airflow.utils.trigger_rule import TriggerRule

with DAG(
    dag_id="latest_only_with_trigger",
    schedule=datetime.timedelta(hours=4),
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
    tags=["example3"],
) as dag:
    latest_only = LatestOnlyOperator(task_id="latest_only")
    task1 = EmptyOperator(task_id="task1")
    task2 = EmptyOperator(task_id="task2")
    task3 = EmptyOperator(task_id="task3")
    task4 = EmptyOperator(task_id="task4", trigger_rule=TriggerRule.ALL_DONE)

    latest_only >> task1 >> [task3, task4]
    task2 >> [task3, task4]
```

この DAG の場合：

* `task1` は `latest_only` の直接の下流にあり、**最新の実行時以外はスキップされます**。
* `task2` は `latest_only` にまったく依存していないため、**すべてのスケジュールされた実行で実行されます**。
* `task3` は `task1` と `task2` の下流にあり、**デフォルトのトリガールールが `all_success` であるため、`task1` がスキップされると連鎖的にスキップされます**。
* `task4` も `task1` と `task2` の下流ですが、**トリガールールが `all_done` に設定されているためスキップされません**。

![](https://airflow.apache.org/docs/apache-airflow/stable/_images/latest_only_with_trigger.png)

### Depends On Past

タスクが、**前回の DAG 実行においてそのタスク自身が成功していた場合にのみ実行されるようにする**こともできます。これを実現するには、**そのタスクに `depends_on_past` 引数を `True` に設定するだけです**。

> [!NOTE]
> DAG のライフサイクルの最初、つまり**初めて自動実行されるタイミングでは、依存すべき過去の実行が存在しないため、タスクは実行されます**。

### トリガールール

デフォルトでは、Airflow は **すべての上流（直接の親）タスクが[成功](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html#concepts-task-states)した場合にのみ**、あるタスクを実行します。

ただし、これはデフォルトの挙動にすぎず、**タスクに `trigger_rule` 引数を指定することで制御することができます**。
`trigger_rule` に指定できるオプションは以下のとおりです：

* **`all_success`（デフォルト）**：すべての上流タスクが成功したときに実行
* **`all_failed`**：すべての上流タスクが `failed` または `upstream_failed` 状態のときに実行
* **`all_done`**：すべての上流タスクの実行が完了したとき（成功・失敗・スキップを問わない）
* **`all_skipped`**：すべての上流タスクが `skipped` 状態のときに実行
* **`one_failed`**：少なくとも1つの上流タスクが `failed` 状態になったときに実行（すべての上流が完了するのを待たない）
* **`one_success`**：少なくとも1つの上流タスクが `success` 状態になったときに実行（完了を待たない）
* **`one_done`**：少なくとも1つの上流タスクが `success` または `failed` のときに実行
* **`none_failed`**：すべての上流タスクが `failed` または `upstream_failed` でない（成功またはスキップ）ときに実行
* **`none_failed_min_one_success`**：すべての上流タスクが `failed` または `upstream_failed` でなく、さらに少なくとも1つが成功したときに実行
* **`none_skipped`**：上流タスクに `skipped` が含まれていない（すべてが成功・失敗・upstream\_failed）ときに実行
* **`always`**：依存関係にかかわらず、いつでもこのタスクを実行する

これらは、[Depends On Past](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#concepts-depends-on-past) 機能と組み合わせて使用することもできます。

> [!NOTE]
> **ブランチ操作によってスキップされたタスクと `trigger_rule` の相互作用には注意が必要です。**
> 特に、ブランチ処理の下流で `all_success` や `all_failed` を使うことはほとんど望ましくありません。
>
> スキップされたタスクは、`all_success` や `all_failed` のトリガールールにも伝播し、**それらもスキップされてしまいます**。
以下の DAG を見てください：
> 
> ```python
> # dags/branch_without_trigger.py
> 
> import pendulum
> from airflow.sdk import task, DAG
> from airflow.providers.standard.operators.empty import EmptyOperator
> 
> dag = DAG(
>     dag_id="branch_without_trigger",
>     schedule="@once",
>     start_date=pendulum.datetime(2019, 2, 28, tz="UTC"),
> )
> 
> run_this_first = EmptyOperator(task_id="run_this_first", dag=dag)
> 
> @task.branch(task_id="branching")
> def do_branching():
>     return "branch_a"
> 
> branching = do_branching()
> 
> branch_a = EmptyOperator(task_id="branch_a", dag=dag)
> follow_branch_a = EmptyOperator(task_id="follow_branch_a", dag=dag)
> branch_false = EmptyOperator(task_id="branch_false", dag=dag)
> join = EmptyOperator(task_id="join", dag=dag)
> 
> run_this_first >> branching
> branching >> branch_a >> follow_branch_a >> join
> branching >> branch_false >> join
> ```
>
> `join` は `follow_branch_a` および `branch_false` の下流にあります。`join` タスクの `trigger_rule` はデフォルトで `all_success` に設定されているため、**ブランチ処理によって発生したスキップが伝播し、`join` タスクもスキップされたものとして表示されます**。
> ![](https://airflow.apache.org/docs/apache-airflow/stable/_images/branch_without_trigger.png)
> `join` タスクの `trigger_rule` を `none_failed_min_one_success` に設定することで、**意図したとおりの動作を実現できます**。
> ![](https://airflow.apache.org/docs/apache-airflow/stable/_images/branch_with_trigger.png)

### セットアップとティアダウン

データワークフローでは、**リソース（たとえばコンピュートリソース）を作成し、それを使って作業を行い、その後で破棄する**という処理が一般的です。Airflow はこのニーズに対応するために、**セットアップ（setup）タスクとティアダウン（teardown）タスク**を提供しています。

この機能の使用方法についての詳細は、メイン記事 [Setup and Teardown](https://airflow.apache.org/docs/apache-airflow/stable/howto/setup-and-teardown.html) をご覧ください。
