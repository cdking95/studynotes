---
description: '[Doctrine Migrations]数据库迁移组件的深入解析三：自定义数据字段类型'
---

# 数据迁移组件三：自定义数据字段类型

## 自定义type

根据[官方文档](http://doctrine-orm.readthedocs.io/projects/doctrine-dbal/en/latest/reference/types.html#custom-mapping-types)，新建TinyIntType类，集成Type，并重写`getName`，`getSqlDeclaration`，`convertToPHPValue`，`getBindingType`等方法。

TinyIntType.php完整代码：

```php
<?php
namespace db\types;

use Doctrine\DBAL\ParameterType;
use Doctrine\DBAL\Platforms\AbstractPlatform;
use Doctrine\DBAL\Types\Type;

/**
 * 扩展DBAL组件类型
 *
 * 迁移组件依赖的DBAL组件默认并不支持tinyint类型
 * 此类是为mysql支持tinyint类型而扩展的类
 *
 * @author tangbo<admin@tbphp.net>
 */
class TinyIntType extends Type
{

    public function getName()
    {
        return 'tinyint';
    }

    public function getSQLDeclaration(array $fieldDeclaration, AbstractPlatform $platform)
    {
        $sql = 'TINYINT';
        if (is_numeric($fieldDeclaration['length']) && $fieldDeclaration['length'] > 1) {
            $sql .= '(' . ((int) $fieldDeclaration['length']) . ')';
        } else {
            $sql .= '(3)';
        }
        if (!empty($fieldDeclaration['unsigned'])) {
            $sql .= ' UNSIGNED';
        }
        if (!empty($fieldDeclaration['autoincrement'])) {
            $sql .= ' AUTO_INCREMENT';
        }
        return $sql;
    }

    public function convertToPHPValue($value, AbstractPlatform $platform)
    {
        return (null === $value) ? null : (int) $value;
    }

    public function getBindingType()
    {
        return ParameterType::INTEGER;
    }
}
```

 其中`getSqlDeclaration`方法是用于生成sql语句，需要根据传入的参数处理sql拼接。

## 注册自定义类型

```php
Type::addType(TinyIntType::TYPENAME, 'db\types\TinyIntType');
$connection->getDatabasePlatform()->registerDoctrineTypeMapping(TinyIntType::TYPENAME, TinyIntType::TYPENAME);
```

这样，你在编写迁移类代码的时候就可以使用tinyint了，例如：

```php
public function up(Schema $schema): void
{
    $table = $schema->createTable('test1');
    $table->addColumn('id', 'integer')->setUnsigned(true)->setAutoincrement(true);
    $table->addColumn('status', 'tinyint')->setLength(2)->setDefault(1);
    $table->setPrimaryKey(['id']);
}
```

## 解决enum类型冲突

迁移组件不支持enum类型，也无法自定义该类型。如果集成迁移组件的时候数据库里已经存在表且有enum类型的字段，那么执行迁移命令时就会报错。

为了解决这个问题，我们需要把enum映射为string类型即可：

```php
$connection->getDatabasePlatform()->registerDoctrineTypeMapping('enum', 'string');
```

## 结语

至此，我们已经完成了迁移组件的所有迁移工作，并且已经能很好的在项目中使用它。

目前集成的是版本管理的方式，是迁移组件默认支持的，也是laravel框架集成的迁移方式。还有之前说到的另一种diff方式的迁移，diff方式能够更精准的控制表结构。

[下一章](shu-ju-qian-yi-zu-jian-si-ji-cheng-diff-fang-shi-qian-yi-zu-jian.md)我们来详细研究如何集成diff方式的数据迁移组件。



{% hint style="info" %}
在[我的代码库](https://github.com/tbphp/studyMigrations/tree/acd4d2c9026c8930a796bd6383cfdd112420aa5c)可以查看这篇文章的详细代码，欢迎star。
{% endhint %}



