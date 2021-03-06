---
description: '[Doctrine Migrations]数据库迁移组件的深入解析二：自定义集成'
---

# 数据迁移组件二：自定义集成

## 自定义命令脚本

### 目录结构

目前的项目结构是这样的（参照[代码库](https://github.com/tbphp/studyMigrations/tree/725659ad104700a2d4d4fbbcf26b5a6c5498de83)）：

![](../.gitbook/assets/wx20180609-004114-2x.png)

其中，`db/migrations`文件夹是迁移类文件夹，`config/db.php`是我们项目原有的db配置，`migrations.php`和`migrations-db.php`是迁移组件需要的配置文件。

### 编写自定义命令脚本

现在先在根目录新建文件：migrate，没有后缀名，并且添加可执行权限。

并且参照组件原有的命令脚本`vendor/doctrine/migrations/doctrine-migrations.php`，首先获取项目原有的数据库配置信息，替换掉migrations-db.php数据库配置文件：

```php
$db_config = include 'config/db.php';
$db_params = [
    'driver' => 'pdo_mysql',
    'host' => $db_config['host'],
    'port' => $db_config['port'],
    'dbname' => $db_config['dbname'],
    'user' => $db_config['user'],
    'password' => $db_config['password'],
];
try {
    $connection = DriverManager::getConnection($db_params);
} catch (DBALException $e) {
    echo $e->getMessage() . PHP_EOL;
    exit;
}
```

然后配置组件，替换掉migrations.php配置文件：

```php
$configuration = new Configuration($connection);
$configuration->setName('Doctrine Migrations');
$configuration->setMigrationsNamespace('db\migrations');
$configuration->setMigrationsTableName('migration_versions');
$configuration->setMigrationsDirectory('db/migrations');
```

 最后是完整的命令 脚本代码：

```php
#!/usr/bin/env php
<?php
require_once 'vendor/autoload.php';

use Doctrine\DBAL\DBALException;
use Doctrine\DBAL\DriverManager;
use Doctrine\DBAL\Migrations\Configuration\Configuration;
use Doctrine\DBAL\Migrations\Tools\Console\ConsoleRunner;
use Doctrine\DBAL\Migrations\Tools\Console\Helper\ConfigurationHelper;
use Doctrine\DBAL\Tools\Console\Helper\ConnectionHelper;
use Symfony\Component\Console\Helper\HelperSet;
use Symfony\Component\Console\Helper\QuestionHelper;

// 读取数据库配置信息
$db_config = include 'config/db.php';
$db_params = [
    'driver' => 'pdo_mysql',
    'host' => $db_config['host'],
    'port' => $db_config['port'],
    'dbname' => $db_config['dbname'],
    'user' => $db_config['user'],
    'password' => $db_config['password'],
];
try {
    $connection = DriverManager::getConnection($db_params);
} catch (DBALException $e) {
    echo $e->getMessage() . PHP_EOL;
    exit;
}

// 迁移组件配置
$configuration = new Configuration($connection);
$configuration->setName('Doctrine Migrations');
$configuration->setMigrationsNamespace('db\migrations');
$configuration->setMigrationsTableName('migration_versions');
$configuration->setMigrationsDirectory('db/migrations');

// 创建命令脚本
$helper_set = new HelperSet([
    'question' => new QuestionHelper(),
    'db' => new ConnectionHelper($connection),
    new ConfigurationHelper($connection, $configuration),
]);
$cli = ConsoleRunner::createApplication($helper_set);
try {
    $cli->run();
} catch (Exception $e) {
    echo $e->getMessage() . PHP_EOL;
}
```

现在执行迁移相关命令时，用`./migrate`替换之前的`vendor/bin/doctrine-migrations`部分即可。

 同时`migrations.php`和`migrations-db.php`这两个配置文件也可以删除了。

## PhpStorm集成迁移命令

如果你使用PhpStorm，命令行还可以集成到开发工具中去，会有自动提示，大大提高工作效率：

1. `PhpStorm` -&gt; `Preferences` -&gt; `Command Line Tool Support` -&gt; 添加
2. Choose tool选择Tool based on Symfony Console,点击OK
3. Alias输入m, Path to PHP executable 选择php路径，path to script选择跟目录下的migrate文件
4. 点击OK，提示 Found 9 commands 则配置成功。
5. `PhpStorm` -&gt; `Tools` -&gt; `Run Command`，弹出命令窗口，输入m即会出现命令提示。

## 结语

到此，数据迁移组件就已经很灵活的集成到我们自己的项目中了，而且你还可以根据自己的项目灵活修改自定义命令脚本文件。

但是如果使用久了会发现现在的数据迁移会有两个问题：

1. 不支持tinyint类型，在相关[issues](https://github.com/doctrine/dbal/issues/2327)中，官方开发人员有回复：

   > "Tiny integer" is not supported by many database vendors. DBAL's type system is an abstraction layer for SQL types including type conversion from PHP to database value and back. I am assuming that your issue refers to MySQL, where we do not have a distinct native SQL type to use for BooleanType mapping. Therefore MySQL's TINYINT is used for that purpose as is fits best from all available native types and in DBAL we do not have an abstraction for tiny integers as such \(as mentioned above\).
   >
   > What you can do is implement your own tiny integer custom type and tell DBAL to use column comments \(for distinction from BooleanType in MySQL\). That should work. See the [documentation](http://doctrine-orm.readthedocs.io/projects/doctrine-dbal/en/latest/reference/types.html#custom-mapping-types).
   >
   > To tell DBAL to use column comments for the custom type, simply override the requiresSQLCommentHint\(\) method to return true.

   所以只能添加自定义类型。

2. 另外一个问题，就是不支持enum类型。深入代码之后，发现enum类型的特殊格式并不好融入到组件中去，自定义类型也不能支持，但是如果你是半途中加入数据迁移组件，现在项目中如果已经有了数据表结构，且包含enum类型的字段，那么使用迁移组件是会报错。

那么，[下一章](shu-ju-qian-yi-zu-jian-san-zi-ding-yi-shu-ju-zi-duan-lei-xing.md)我们就来完成这两件事：添加tinyint的自定义类型，解决enum报错的问题。



{% hint style="info" %}
在[我的代码库](https://github.com/tbphp/studyMigrations/tree/01dcd0e0950f2ccceebcdca36838e17a7eb5f0ec)可以查看这篇文章的详细代码，欢迎star。
{% endhint %}



