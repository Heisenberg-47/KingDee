# XML修改继承模板

### 下面用《物料外贸信息》修改**基础资料带组织模板**到**树形基础资料带组织模板为例**

####  首先找到目标模板的*ID*与*InheritPath*,去模板中的XML中查找  本例中`ID = 19181b8e000099ac`

ex:

![image-20241230183521096](https://s2.loli.net/2024/12/30/39qcQxBLEPjU5kN.png)

####  导出需要修改模板的单据，打开*.dym*文件

####  替换所有的 *parentid* 为模板*id* `19181b8e000099ac`
#### 所有的*InheritPath* 追加模板的 *InheritPath* ,替换最后一项为 模板id `19181b8e000099ac`

![image-20241230183102049](https://s2.loli.net/2024/12/30/QEtmcUuKTR1Zjvq.png)

压缩重新导入，刷新页面，检查是否生效。