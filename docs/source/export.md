# 导出

### export_config

```protobuf
export_config {
}
```

- batch_size: 导出模型的batch_size，默认是-1，即可以接收任意batch_size
- exporter_type: 导出类型,  best | final | latest | none，默认final
  - best 导出最好的模型
  - final 训练结束后导出
  - latest 导出最新的模型
  - none 不导出
- best_exporter_metric: 当exporter_type为best的时候，确定最优导出模型的metric，注意该metric要在eval_config的metrics_set设置了才行
- metric_bigger: 确定最优导出模型的metric是越大越好，还是越小越好，默认是越大越好
- exports_to_keep: 当exporter_type为best或lastest时，保留n个最好或最新的模型，默认为1
  ```protobuf
  export_config {
    exporter_type: "best"
    best_exporter_metric: "auc"
    exports_to_keep: 1
  }
  ```
- multi_placeholder: 使用一个placeholder还是多个placeholder。默认为true，即对每个特征使用单个placeholder
- multi_value_fields: 针对tagFeature，指定一个字段集合，使得导出的placeholder可以接收二维数组，而不是训练时用的字符串类型，这样可以节省字符串拆分和类型转换的时间。
  ```protobuf
  export_config {
    multi_value_fields {
       input_name: ["field1", "field2", "field3"]
    }
  }
  ```
- placeholder_named_by_input: true时利用data_config.input_fields.input_name来命令每个placeholder，false时每个placeholder名字为"input_X"，"X"为data_config.input_fields的编号(0-input_num)。默认为False
- asset_files: 需要导出的asset文件, 可设置多个
- enable_early_stop: 根据early_stop_func的返回值判断是否要提前结束训练
  - 示例:
    - samples/model_config/custom_early_stop_on_taobao.config
    - samples/model_config/multi_tower_early_stop_on_taobao.config
  - 应用场景:
    - 数据量比较小时，需要训练多个epoch时, 打开early_stop可以防止过拟合;
    - 使用[超参搜索](./automl/pai_nni_hpo.md)时, 打开可以提前终止收敛比较差的参数, 显著提升搜索效率
- early_stop_func: 判断是否要提前结束训练的函数
  - 返回值:
    - True, 结束训练
    - False, 继续训练
  - 默认值:
    - metric_bigger为true时, easy_rec.python.compat.early_stopping.stop_if_no_increase_hook
    - metric_bigger为false时, easy_rec.python.compat.early_stopping.stop_if_no_decrease_hook
  - 自定义early_stop_func:
    - 示例: easy_rec.python.test.custom_early_stop_func.custom_early_stop_func
    - 参数: 框架传入两个参数
      - eval_results: 模型评估结果
      - func_param: 自定义参数(即export_config.early_stop_params)
- max_check_steps: 训练max_check_steps之后评估指标没有改善，即停止训练; 仅适用于内置early_stop_func, 不适用于自定义early_stop_func
  - stop_if_no_increase_hook: 对应max_steps_without_increase, 当间隔max_check_steps训练步数评估指标没有提升，即停止训练
  - stop_if_no_decrease_hook: 对应max_steps_without_decrease, 当间隔max_check_steps训练步数评估指标没有下降, 即停止训练
- early_stop_params: 传递给early_stop_func的自定义参数

### 导出命令

#### Local

```bash
python -m easy_rec.python.export --pipeline_config_path dwd_avazu_ctr_deepmodel.config --export_dir ./export
```

- --pipeline_config_path: config文件路径
- --model_dir: 如果指定了model_dir将会覆盖config里面的model_dir，一般在周期性调度的时候使用
- --export_dir: 导出的目录

#### PAI

```sql
pai -name easy_rec_ext -project algo_public
-Dconfig=oss://easyrec/easy_rec_test/dwd_avazu_ctr_deepmodel_ext.config
-Dcmd=export
-Dexport_dir=oss://easyrec/easy_rec_test/export
-Dcluster='{"worker" : {"count":1, "cpu":1000, "memory":40000}}'
-Darn=acs:ram::xxx:role/ev-ext-test-oss
-Dbuckets=oss://easyrec/
-DossHost=oss-cn-beijing-internal.aliyuncs.com
```

- -Dconfig: 同训练
- -Dcmd: export 模型导出
- -Dexport_dir: 导出的目录
- -Dcheckpoint_path: 使用指定的checkpoint_path
- -Darn: rolearn  注意这个的arn要替换成客户自己的。可以从dataworks的设置中查看arn。
- -DossHost: ossHost地址
- -Dbuckets: config所在的bucket和保存模型的bucket; 如果有多个bucket，逗号分割
- 如果是pai内部版,则不需要指定arn和ossHost, arn和ossHost放在-Dbuckets里面
  - -Dbuckets=oss://easyrec/?role_arn=acs:ram::xxx:role/ev-ext-test-oss&host=oss-cn-beijing-internal.aliyuncs.com
