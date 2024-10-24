# リソース制限

## コンテナ単位

### 可能な項目
- CPU
- メモリ
- Ephemeral Storage
- GPU(Plugin) .etc

### CPU指定

> 1vCPUは1つのCPU(仮想)コアで動作しており, そのコア一つの性能は, 何GHzの処理性能であるかやアーキテクチャ, ベンチマークスコア, IPC(1サイクルあたりの命令数)などに依存する.

1vCPU(仮想CPU)を1000millicoresとした単位<br>
->3GHzのCPUでも1コア相当を指定したいなら, 3000mではなく, 1000mとなる.
``` txt
1 = 1000m = 1vCPU
```

> 性能比較を行う際の何vCPUのCPUを使うか(`large`, `medium`など)は自由であるが, インスタンスファミリタイプは変更しないようにする.

### メモリ指定

``` txt
1G = 1000M(1Gi = 1024Mi)
```

### RequestとLimit
Requestは最低限必要なリソース, Limitはそのコンテナに与えられる上限を表し, Requestで指定されているリソースがノードに残っていない場合, スケジュールされず, ノードのAutoScaleが行われる.
Limitで指定したリソースは割り当て時に残っている必要はなく, Limitのうち, 指定したしきい値(%)を越えたリソースをコンテナが必要としたときに, Podのスケーリングが行われる.

> Requestのみ指定すれば, Limitは自動で決まらず, ホスト側の負荷の上限まで使い切る.
> Limitのみ設定すれば, Requestは同じ値に設定される.

### Ephemeral Storage
リソース制御は可能であるが, 通常はマウント先に書き込むことが多いので, 今回は設定しない.

### Eviction Manager
`kube-reserved`と`system-reserved`を差し引いたリソースが`Allocatable`で割当が可能(Nodeリソースから確認可能).

Eviction Managerは`Kubernetes`自体が停止しないように制限を越えそうな場合に`Pod`の停止を試みる(Podの強制終了).
> Podの終了順序は以下の順番.<br>
> 1. Requestに割り当てられている量よりも超過してリソースを消費しているもの
> 1. PodPriorityがより低いもの
> 1. Requestsに割り当てられている量よりも超過して消費しているリソース量がより多いもの

## Namespace単位

### LimitRange
> 新規に作成されるPodに適応されることに注意(Podを配置する前に適応しておく)

- DefaultのRequest/Limit, その比, 最小/最大のリソース

| Type | 利用可能な設定項目 |
| - | - |
| Container | default/defaultRequest/max/min/maxLimitRequestRatio |
| Pod | max/min/maxLimitRequestRatio |
| PersistentVolumeClaim(storage) | max/min |

> いくつかの環境では, CPUのdefaultRequestが設定されており, GKEでもデフォルトで100mに設定されたLimitRangeが作成されている.

### Resource Quta
- 作成可能なリソース数の制限
- リソース使用量の制限

## QoS Class
Podには, Limit/Requestで指定している項目によって, QoS Classが与えられる.