# doctrine-dbal-clickhouse

Doctrine DBAL driver for ClickHouse -- an open-source column-oriented database management system by Yandex (https://clickhouse.yandex/)

Doctrine and [smi2/phpclickhouse](https://packagist.org/packages/smi2/phpclickhouse) are required.

*It is alpha-version!*


## Initialization
### Custom PHP script
```php
$connectionParams = [
    'host' => 'localhost',
    'port' => 8123,
    'user' => 'default',
    'password' => '',
    'dbname' => 'default',
    'driverClass' => 'Mochalygin\DoctrineDBALClickHouse\Driver',
    'wrapperClass' => 'Mochalygin\DoctrineDBALClickHouse\Connection'
];
$conn = \Doctrine\DBAL\DriverManager::getConnection($connectionParams, new \Doctrine\DBAL\Configuration());
```

### Symfony 2/3
configure...
```yml
# app/config/config.yml
doctrine:
    dbal:
        connections:
            clickhouse:
                host:     localhost
                port:     8123
                user:     default
                password: ""
                dbname:   default
                driver_class: Mochalygin\DoctrineDBALClickHouse\Driver
                wrapper_class: Mochalygin\DoctrineDBALClickHouse\Connection
            #mysql:
            #   ...
```
...and get from the service container
```php
$conn = $this->get('doctrine.dbal.clickhouse_connection');
```


## Usage

### Create new table
```php
// ***short***
$fromSchema = $conn->getSchemaManager()->createSchema();
$toSchema = clone $fromSchema;

// create new table object
$newTable = $toSchema->createTable('new_table_name');

// add columns
$newTable->addColumn('id', 'integer', ['unsigned' => true]);
$newTable->addColumn('payload', 'string');

//set primary key
$newTable->setPrimaryKey(['id']);

// execute migration SQLs to create table in ClickHouse
$sqlArray = $fromSchema->getMigrateToSql($toSchema, $conn->getDatabasePlatform());
foreach ($sqlArray as $sql) {
    $conn->exec($sql);
}
```

```php
// ***additional parameters***

//specify table engine
$newTable->addOption('engine', 'MergeTree');
// *if not specified -- default engine 'ReplacingMergeTree' will be used


// add Date column for partitioning
$newTable->addColumn('event_date', 'date', ['default' => 'toDate(now())']);
$newTable->addOption('eventDateColumn', 'event_date');
// *if not specified -- default Date column named EventDate will be added

//specify index granularity
$newTable->addOption('indexGranularity', 4096);
// *if not specified -- default value 8192 will be used
```

### Data Retrieval
```php
echo $conn->fetchColumn('SELECT SUM(views) FROM articles');
```

### Dynamic Parameters and Prepared Statements
```php
$stmt = $conn->prepare('SELECT authorId, SUM(views) AS total_views FROM articles WHERE category_id = :categoryId AND publish_date = :publishDate GROUP BY authorId');

$stmt->bindValue('categoryId', 123);
$stmt->bindValue('publishDate', new \DateTime('2017-02-29'), 'datetime');
$stmt->execute();

while ($row = $stmt->fetch()) {
    echo $row['authorId'] . ': ' . $row['total_views'] . PHP_EOL;
}
```
### Insert
```php
$conn->insert('articles', ['category_id' => 123, 'author_id' => 777, 'views' => 100500]);
// INSERT INTO articles (category_id, author_id, views) VALUES (?, ?, ?) (123, 777, 100500)
```

### More information in Doctrine DBAL documentation:
* [Data Retrieval And Manipulation](http://docs.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/data-retrieval-and-manipulation.html)
* [SQL Query Builder](http://docs.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/query-builder.html)
* [Schema-Representation](http://docs.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/schema-representation.html)
