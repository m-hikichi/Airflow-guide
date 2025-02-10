# [タスク](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html)

タスクは、Airflowにおける実行の基本単位です。タスクは[DAG](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html)（有向非巡回グラフ）に配置され、その中でタスク同士の実行順序を示すために、上流（upstream）および下流（downstream）の依存関係を設定します。

タスクには、基本的に3種類があります。

- **[オペレーター（Operators）](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/operators.html)**  
  DAGのほとんどの部分を素早く構築できる、あらかじめ定義されたタスクのテンプレートです。

- **[センサー（Sensors）](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/sensors.html)**  
  オペレーターの特別なサブクラスで、外部のイベントが発生するのを待機することに特化しています。

- **[TaskFlow](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/taskflow.html)で装飾された`@task`**  
  カスタムPython関数をタスクとしてパッケージ化したものです。

内部的には、これらはすべてAirflowの`BaseOperator`を継承したサブクラスです。タスクとオペレーターの概念はほぼ同じ意味を持ちますが、別々の概念として考えることが有用です。基本的には、オペレーターとセンサーはテンプレートであり、DAGファイル内でこれらを呼び出すとタスクが作成されます。

## 関係性

タスクを使う際の重要な部分は、タスク同士の関係を定義することです。これが依存関係であり、Airflowではこれを上流タスク（upstream tasks）および下流タスク（downstream tasks）と呼びます。まずタスクを定義し、その後にそれらの依存関係を定義します。

> [!NOTE]  
> 上流タスクは、他のタスクの直前に実行されるタスクを指します。以前はこれを親タスクと呼んでいましたが、この概念はタスク階層で上位にあるタスク（つまり直接の親タスク）を指すものではないことに注意してください。下流タスクについても同様で、他のタスクの直下にあるタスクを指します。

依存関係を宣言する方法は2種類あります。1つ目は、ビットシフト演算子（`>>`、`<<`）を使う方法です。

```python
first_task >> second_task >> [third_task, fourth_task]
```

2つ目は、より明示的に`set_upstream`や`set_downstream`メソッドを使う方法です。

```python
first_task.set_downstream(second_task)
third_task.set_upstream(second_task)
```

どちらも同じことを実行しますが、一般的にはビットシフト演算子を使う方が読みやすいので、こちらを推奨します。

デフォルトでは、タスクはその上流タスク（親タスク）がすべて成功した場合に実行されますが、この動作は分岐を加えたり、特定の上流タスクの完了だけを待機したり、実行履歴に基づいて挙動を変更するなど、さまざまな方法で変更できます。詳細については「[制御フロー（Control Flow）](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#concepts-control-flow)」を参照してください。

タスクはデフォルトでは互いに情報を渡しません。各タスクは独立して実行されます。もし1つのタスクから別のタスクに情報を渡したい場合は、[XComs](./XComs.md)を使用する必要があります。

## タスクインスタンス

DAGが実行されるたびにDAGランがインスタンス化されるのと同様に、DAG内のタスクはタスクインスタンスとしてインスタンス化されます。

タスクのインスタンスは、特定のDAG（およびそのデータインターバル）におけるそのタスクの実行回数を指します。タスクインスタンスは、タスクがどの段階にあるかを示す状態を持つタスクの表現です。

タスクインスタンスの状態には次のようなものがあります：

- **none**: タスクはまだ実行のためにキューに入っていない（依存関係が満たされていない）
- **scheduled**: スケジューラーがタスクの依存関係を確認し、実行すべきと判断した
- **queued**: タスクはエグゼキュータに割り当てられ、ワーカーの準備を待っている
- **running**: タスクはワーカーで実行中（またはローカル/同期エグゼキュータで実行中）
- **success**: タスクはエラーなく実行を終了した
- **restarting**: 実行中のタスクが外部から再起動を要求された
- **failed**: タスク実行中にエラーが発生し、失敗した
- **skipped**: 分岐、LatestOnly、またはそれに類する理由でタスクがスキップされた
- **upstream_failed**: 上流タスクが失敗し、[トリガールール](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#concepts-trigger-rules)によりそのタスクが必要だった
- **up_for_retry**: タスクが失敗したが、再試行回数が残っており、再スケジュールされる予定
- **up_for_reschedule**: タスクは[センサー](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/sensors.html)で、再スケジュール待機中
- **deferred**: タスクが[トリガーにより延期された](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/deferring.html)
- **removed**: タ実行が開始されてから、タスクがDAGから削除された

![](https://airflow.apache.org/docs/apache-airflow/stable/_images/task_lifecycle_diagram.png)

理想的には、タスクは「`none`」から「`scheduled`」、「`queued`」、「`running`」を経て、最終的に「`success`」に進むべきです。

カスタムタスク（オペレーター）が実行中のとき、そのタスクはタスクインスタンスのコピーを受け取り、タスクのメタデータを調査できるだけでなく、[XComs](./XComs.md)などのためのメソッドも含まれています。

## 関係性の用語

各タスクインスタンスには、他のインスタンスとの間に2種類の関係があります。

まず、上流および下流のタスクがあります：

```python
task1 >> task2 >> task3
```

DAGが実行されると、それぞれのタスクの上流/下流のタスクに対してインスタンスが作成されますが、すべて同じデータインターバルを持ちます。

また、同じタスクのインスタンスが異なるデータインターバルに対して存在することもあります。それは他のDAGランからのインスタンスです。これらを「previous」および「next」と呼びますが、これは上流および下流とは異なる関係性です！

> [!NOTE]  
> 古いAirflowのドキュメントでは「previous」が「upstream」を意味する場合があります。これを見つけた場合は、修正を手伝ってください！

## タイムアウト

タスクに最大実行時間を設定したい場合は、そのタスクの`execution_timeout`属性に最大許容実行時間を示す`datetime.timedelta`の値を設定します。これは、センサーを含むすべてのAirflowタスクに適用されます。`execution_timeout`は、各実行に許可される最大時間を制御します。`execution_timeout`を超過すると、タスクはタイムアウトし、`AirflowTaskTimeout`が発生します。

さらに、センサーには`timeout`パラメータがあります。これは、`reschedule`モードのセンサーにのみ関係します。`timeout`は、センサーが成功するために許可される最大時間を制御します。`timeout`を超過すると、`AirflowSensorTimeout`が発生し、センサーはリトライせずに即座に失敗します。

以下に、`SFTPSensor`を使った例を示します。この`sensor`は`reschedule`モードにあり、成功するまで定期的に実行され、再スケジュールされます。
- センサーがSFTPサーバーにアクセスするたびに、最大60秒の実行時間が許可されます（これは`execution_timeout`で定義された時間です）。
- もし、センサーがSFTPサーバーにアクセスするのに60秒以上かかると、`AirflowTaskTimeout`が発生します。この場合、センサーはリトライ可能です。リトライ回数は`retries`で定義された回数まで行えます。
- 最初の実行開始から、最終的に成功するまで（つまり、`root/test`というファイルが現れるまで）、センサーには最大3600秒が許可されています（これは`timeout`で定義された時間です）。つまり、もしファイルがSFTPサーバーに3600秒以内に現れない場合、センサーは`AirflowSensorTimeout`を発生させます。このエラーが発生した場合、リトライは行われません。
- もし、センサーがネットワーク障害などの理由で3600秒の間に失敗した場合、リトライ回数は`retries`で定義された回数まで可能です。リトライは`timeout`をリセットしません。成功するために許可される最大時間は、合計で3600秒となります。

```python
sensor = SFTPSensor(
    task_id="sensor",
    path="/root/test",
    execution_timeout=timedelta(seconds=60),
    timeout=3600,
    retries=2,
    mode="reschedule",
)
```

もし、タスクが指定した時間を超えて実行される場合に通知を受け取るだけで、タスクはそのまま完了するまで実行させたい場合は、[SLAs](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html#concepts-slas)を使用した方がよいでしょう。
