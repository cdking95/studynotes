---
description: Doctrine Migrations数据迁移组件的简单使用说明
---

# Doctrine Migrations组件的简单使用

## 简单流程

1. 执行下面的命令，`/data/migrations` 目录下会生成`Versionxxxx.php`格式的迁移脚本文件。

   ```bash
   ./vendor/bin/doctrine-migrations migrations:generate
   ```

2. 编辑迁移脚本，写入迁移的代码，详情参见[迁移脚本编写方式](doctrinemigrations-jian-dan-shi-yong.md#qian-yi-jiao-ben-bian-xie)。
3. 执行迁移命令，修改数据库。

   ```bash
   ./vendor/bin/doctrine-migrations migrations:migrate
   ```

{% hint style="info" %}
注意，在参照简单流程编写迁移脚本时，需要遵循[迁移规范](doctrinemigrations-jian-dan-shi-yong.md#qian-yi-gui-fan)。
{% endhint %}

## 常用命令

```bash
# 生成迁移脚本
./vendor/bin/doctrine-migrations migrations:generate
# 执行迁移到最新版本
./vendor/bin/doctrine-migrations migrations:migrate
# 回到最初版本
./vendor/bin/doctrine-migrations migrations:migrate first
# 回到上一个版本
./vendor/bin/doctrine-migrations migrations:migrate prev
# 切换到下一个版本
./vendor/bin/doctrine-migrations migrations:migrate next
# --dry-run是空转参数，只显示操作结果，不执行修改
./vendor/bin/doctrine-migrations migrations:migrate --dry-run
# 不执行操作，只写入文件，对于生产环境需要手动验证并执行的场景有用
./vendor/bin/doctrine-migrations migrations:migrate --write-sql=file.sql
# 执行到最新版本，和不使用latest参数效果一样
./vendor/bin/doctrine-migrations migrations:migrate latest
# 查看详细信息
./vendor/bin/doctrine-migrations status
```

{% hint style="info" %}
migrations:generate 和 migrations:migrate 是最常用的命令。

如果使用phpstorm，可以[集成命令](doctrinemigrations-jian-dan-shi-yong.md#phpstorm-ji-cheng-qian-yi-ming-ling)，使用起来更方便。
{% endhint %}

## 迁移脚本编写

生成的迁移脚本有两个方法，`up`和`down`方法。

### up\(\)

up方法会在执行迁移命令时自动调用。在此方法中写入此次需要修改的脚本，例如创建表，修改字段，删除表，添加需要版本控制数据等等。

创建表示例：

```php
public function up(Schema $schema)
{
    $table = $schema->createTable($this->tableName);
    $table->addColumn('id', 'integer')->setUnsigned(true)->setAutoincrement(true);
    $table->addColumn('uid', 'integer')->setDefault(0)->setUnsigned(true)->setComment('关联user.id');
    $table->addColumn('title', 'string')->setComment('标题');
    $table->addColumn('status', 'tinyint')->setComment('状态：1.测试1,2.测试2');
    $table->addColumn('score', 'decimal')->setLength(5)->setScale(2)->setDefault(0.00)->setComment('分数');
    // 添加主键和索引-必须在字段添加之后执行添加，索引一般最后操作索引。
    $table->setPrimaryKey(['id'])->addUniqueIndex(['uid'])->addIndex(['status']);
}
```

 修改表示例：

```php
public function up(Schema $schema)
{
    $table = $schema->getTable($this->tableName);
    $table->changeColumn('status', ['type' => Type::getType('integer')])
        ->dropColumn('score');
}
```

插入数据示例：

```php
public function up(Schema $schema)
{
    $this->addSql('INSERT INTO `admin_method`(`method` , `name` , `type`) VALUES( :method, :name, :type )', [
        'method' => 'admin.test',
        'name' => '测试方法',
        'type' => 4,
    ]);
}
```

### down\(\)

down方法会在执行版本回退时自动调用。在此方法需要对于up方法，写入逆向操作的脚本。例如up创建了表，则down需要删除对应的表。如果up修改了字段，down需要修改对应的字段为原始值。

示例：

```php
public function down(Schema $schema)
{
    if ($schema->hasTable($this->tableName)) {
        $schema->dropTable($this->tableName);
    }
}
```

## 迁移规范

* 迁移脚本必须在本地测试通过才可提交。
* 迁移脚本一旦push，就不能回退版本，如果要修改之前版本的sql，必须再生成一个脚本。只有未提交的迁移脚本，可以在本地使用回退反复修改。
* 每次pull代码之后执行`./vendor/bin/doctrine-migrations migrations:migrate`命令，更新至最新版sql。

## PhpStorm集成迁移命令

1. `PhpStorm` -&gt; `Preferences` -&gt; `Command Line Tool Support` -&gt; 添加
2. Choose tool选择Tool based on Symfony Console,点击OK
3. Alias输入m, Path to PHP executable 选择php路径，path to script选择跟目录下的migrations.php文件
4. 点击OK，提示 Found 9 commands 则配置成功。
5. `PhpStorm` -&gt; `Tools` -&gt; `Run Command`，弹出命令窗口，输入m即会出现命令提示。



