# el-select使用object作为value值


在做namespace克隆的时候需要用到两个下拉框选择源namespace和目标namespace,用到了element的el-select组件，发现了一个有问题，使用object作为值，选择下拉框都会只显示最后一个数据。

![a1360b19fc4ad65395daa3b57e15eb4277494900.png](/img/a1360b19fc4ad65395daa3b57e15eb4277494900.png)

数据如下：

![9891a5d34f831ad9f7aac9539072efed85c4dc88.png](/img/9891a5d34f831ad9f7aac9539072efed85c4dc88.png)

代码如下：

```
<el-select v-model="sourceCluster" placeholder="源集群">
            <el-option
            v-for="item in clusterOptions"
            :key="item.id"
            :label="item.cluster + '/' + item.namespace"
            :value="item">
            </el-option>
          </el-select>
        </el-form-item>
        <el-form-item label="克隆到">
          <el-select v-model="targetCluster" placeholder="目标集群">
            <el-option
              v-for="item in clusterOptions"
              :key="item.id"
              :label="item.cluster + '/' + item.namespace"
              :value="item">
            </el-option>
          </el-select>
```

查看官方文档发现，如果使用对象类型时，要添加`value-key` 作为value的唯一标识键名，修改后如下：

```
<el-select v-model="sourceCluster" value-key="id" placeholder="源集群">
            <el-option
            v-for="item in clusterOptions"
            :key="item.id"
            :label="item.cluster + '/' + item.namespace"
            :value="item">
            </el-option>
          </el-select>
        </el-form-item>
        <el-form-item label="克隆到">
          <el-select v-model="targetCluster" value-key="id" placeholder="目标集群">
            <el-option
              v-for="item in clusterOptions"
              :key="item.id"
              :label="item.cluster + '/' + item.namespace"
              :value="item">
            </el-option>
          </el-select>
```

可以正常选择了。

![297596a2c951c126c4e02456f814f0907c591d98.png](/img/297596a2c951c126c4e02456f814f0907c591d98.png)

