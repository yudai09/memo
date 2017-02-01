# Redmineのバージョンアップに伴うDBのマイグレーション

Redmineをv0.8.7からv3.3.2にアップグレードする際にDBのマイグレーションが大変だったので対応履歴をメモする。
多分見返せばもう少し効率的な方法があったんだと思う。

DBを空の状態にして、新しくインストールしてできたデータをバックアップしておく。
これは後でエラーが起きたときに、現行のデータと新しいデータのスキーマの違いを把握するために使う。
```
mysqldump -u redmine -ptest123 redmine > redmigre.sql.new
```

ダンプを突っ込む。
```
mysql -u redmine -ptest123 redmine < redmigre.sql
```

これでredmineテーブルは現行のDBのデータが入り、すんなりRedmineが起動するわけはないが、とりあえず起動してみる。
コンテナ立ち上げ

```
docker service create --name sv_redmine_custom \
--reserve-memory 100m \
--replicas 1 \
--network nw_galera \
-e 'REDMINE_DB_MYSQL=sv_galera' \
-e 'REDMINE_DB_USERNAME=redmine' \
-e 'REDMINE_DB_PASSWORD=test123' \
-e 'REDMINE_DB_DATABASE=redmine' \
-e 'REDMINE_DB_ENCODING=utf8mb4' \
-p 8888:3000 \
redmine:custom
```

予想通り、以下のようなエラーが起きるため、Redmineが起動しない
```
Mysql2::Error: Duplicate column name 'editable': ALTER TABLE `custom_fields` ADD `editable` tinyint(1) DEFAULT 1
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:80:in `_query'
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:80:in `block in query'
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:79:in `handle_interrupt'
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:79:in `query'
```

selectすると何も入っていなかったので `drop custom_fields`

```
mysql> select * from custom_fields;
Empty set (0.00 sec)
mysql>  ALTER TABLE `custom_fields` ADD `editable` tinyint(1) DEFAULT 1
    -> ;
ERROR 1060 (42S21): Duplicate column name 'editable'
mysql> ALTER TABLE `custom_fields` DELETE `editable`;                    
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'DELETE `editable`' at line 1
mysql> ALTER TABLE `custom_fields` DROP `editable`;  
Query OK, 0 rows affected (0.07 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

今度はこれが出た。
```
Mysql2::Error: Table 'member_roles' already exists: CREATE TABLE `member_roles` (`id` int(11) auto_increment PRIMARY KEY, `member_id` int(11) NOT NULL, `role_id` int(11) NOT NULL) ENGINE=InnoDB
```

```
mysql> select * from member_roles;
Empty set (0.00 sec)
mysql> drop table member_roles;
```

今度はDBではないところでエラーが出た。

```
undefined method `inherit_members?' for #<Project:0x00000004e8d7b8>
/usr/local/bundle/gems/activemodel-4.2.7.1/lib/active_model/attribute_methods.rb:433:in `method_missing'
/usr/src/redmine/app/models/member_role.rb:66:in `block in add_role_to_subprojects'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/relation/delegation.rb:46:in `each'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/relation/delegation.rb:46:in `each'
/usr/src/redmine/app/models/member_role.rb:65:in `add_role_to_subprojects'
/usr/local/bundle/gems/activesupport-4.2.7.1/lib/active_support/callbacks.rb:432:in `block in make_lambda'
/usr/local/bundle/gems/activesupport-4.2.7.1/lib/active_support/callbacks.rb:228:in `call'
/usr/local/bundle/gems/activesupport-4.2.7.1/lib/active_support/callbacks.rb:228:in `block in halting_and_conditional'
/usr/local/bundle/gems/activesupport-4.2.7.1/lib/active_support/callbacks.rb:506:in `call'
/usr/local/bundle/gems/activesupport-4.2.7.1/lib/active_support/callbacks.rb:506:in `block in call'
/usr/local/bundle/gems/activesupport-4.2.7.1/lib/active_support/callbacks.rb:506:in `each'
/usr/local/bundle/gems/activesupport-4.2.7.1/lib/active_support/callbacks.rb:506:in `call'
/usr/local/bundle/gems/activesupport-4.2.7.1/lib/active_support/callbacks.rb:92:in `__run_callbacks__'
/usr/local/bundle/gems/activesupport-4.2.7.1/lib/active_support/callbacks.rb:778:in `_run_create_callbacks'
```

member_roleテーブルに何か足りないのかも。

見てみる
```
mysql> desc member_roles;
+-----------+---------+------+-----+---------+----------------+
| Field     | Type    | Null | Key | Default | Extra          |
+-----------+---------+------+-----+---------+----------------+
| id        | int(11) | NO   | PRI | NULL    | auto_increment |
| member_id | int(11) | NO   |     | NULL    |                |
| role_id   | int(11) | NO   |     | NULL    |                |
+-----------+---------+------+-----+---------+----------------+
3 rows in set (0.00 sec)
```

新しい方のテーブル定義は以下となっている。
```
CREATE TABLE `member_roles` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `member_id` int(11) NOT NULL,
  `role_id` int(11) NOT NULL,
  `inherited_from` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `index_member_roles_on_member_id` (`member_id`),
  KEY `index_member_roles_on_role_id` (`role_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
/*!40101 SET character_set_client = @saved_cs_client */;
```

とりあえず、新しい定義でテーブルを作っておけばいい気がする。
```
mysql> drop table member_roles;
Query OK, 0 rows affected (0.02 sec)

mysql> CREATE TABLE `member_roles` (
    ->   `id` int(11) NOT NULL AUTO_INCREMENT,
    ->   `member_id` int(11) NOT NULL,
    ->   `role_id` int(11) NOT NULL,
    ->   `inherited_from` int(11) DEFAULT NULL,
    ->   PRIMARY KEY (`id`),
    ->   KEY `index_member_roles_on_member_id` (`member_id`),
    ->   KEY `index_member_roles_on_role_id` (`role_id`)
    -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.07 sec)
```

似たようなエラーだがエラー内容から察するにProjectにメンバが足りないっぽいな。

```
undefined method `inherit_members?' for #<Project:0x00000005a318a8>
/usr/local/bundle/gems/activemodel-4.2.7.1/lib/active_model/attribute_methods.rb:433:in `method_missing'
/usr/src/redmine/app/models/member_role.rb:66:in `block in add_role_to_subprojects'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/relation/delegation.rb:46:in `each'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/relation/delegation.rb:46:in `each'
/usr/src/redmine/app/models/member_role.rb:65:in `add_role_to_subprojects'
```

同様に現在の定義と新しい定義を比較。
```
mysql> desc projects;
+-------------+--------------+------+-----+---------+----------------+
| Field       | Type         | Null | Key | Default | Extra          |
+-------------+--------------+------+-----+---------+----------------+
| id          | int(11)      | NO   | PRI | NULL    | auto_increment |
| name        | varchar(30)  | NO   |     |         |                |
| description | text         | YES  |     | NULL    |                |
| homepage    | varchar(255) | YES  |     |         |                |
| is_public   | tinyint(1)   | NO   |     | 1       |                |
| parent_id   | int(11)      | YES  |     | NULL    |                |
| created_on  | datetime     | YES  |     | NULL    |                |
| updated_on  | datetime     | YES  |     | NULL    |                |
| identifier  | varchar(20)  | YES  |     | NULL    |                |
| status      | int(11)      | NO   |     | 1       |                |
| lft         | int(11)      | YES  |     | NULL    |                |
| rgt         | int(11)      | YES  |     | NULL    |                |
+-------------+--------------+------+-----+---------+----------------+
12 rows in set (0.00 sec)

CREATE TABLE `projects` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) NOT NULL DEFAULT '',
  `description` text,
  `homepage` varchar(255) DEFAULT '',
  `is_public` tinyint(1) NOT NULL DEFAULT '1',
  `parent_id` int(11) DEFAULT NULL,
  `created_on` datetime DEFAULT NULL,
  `updated_on` datetime DEFAULT NULL,
  `identifier` varchar(255) DEFAULT NULL,
  `status` int(11) NOT NULL DEFAULT '1',
  `lft` int(11) DEFAULT NULL,
  `rgt` int(11) DEFAULT NULL,
  `inherit_members` tinyint(1) NOT NULL DEFAULT '0',
  `default_version_id` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `index_projects_on_lft` (`lft`),
  KEY `index_projects_on_rgt` (`rgt`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;
```

いかが足らない。 `alter ` しとくか。
```
  `inherit_members` tinyint(1) NOT NULL DEFAULT '0',
  `default_version_id` int(11) DEFAULT NULL,
```

```
mysql> ALTER TABLE `projects` ADD `inherit_members` tinyint(1) NOT NULL DEFAULT '0';
mysql> ALTER TABLE `projects` ADD `default_version_id` int(11) DEFAULT NULL;
```

今度はこれ
```
Mysql2::Error: Table 'groups_users' already exists: CREATE TABLE `groups_users` (`group_id` int(11) NOT NULL, `user_id` int(11) NOT NULL) ENGINE=InnoDB
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:80:in `_query'
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:80:in `block in query'
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:79:in `handle_interrupt'
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:79:in `query'
```

空なので削除
```
mysql> select * from groups_users;
Empty set (0.00 sec)

mysql> drop table groups_users;
Query OK, 0 rows affected (0.00 sec)
```

今度はこれ
```
Mysql2::Error: Duplicate column name 'inherited_from': ALTER TABLE `member_roles` ADD `inherited_from` int(11)
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:80:in `_query'
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:80:in `block in query'
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:79:in `handle_interrupt'
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:79:in `query'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/connection_adapters/abstract_mysql_adapter.rb:309:in `block in execute'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/connection_adapters/abstract_adapter.rb:484:in `block in log'
/usr/local/bundle/gems/activesupport-4.2.7.1/lib/active_support/notifications/instrumenter.rb:20:in `instrument'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/connection_adapters/abstract_adapter.rb:478:in `log'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/connection_adapters/abstract_mysql_adapter.rb:309:in `execute'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/connection_adapters/mysql2_adapter.rb:231:in `execute'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/connection_adapters/abstract/schema_statements.rb:407:in `add_column'
```

さっき追加したやつが重複していると言われているのが意味不明。
どうすればいいのか？
一旦カラムだけ消しておく？

```
mysql> ALTER TABLE `member_roles` DROP `inherited_from`;
```

次はこれ
```
Index name 'index_member_roles_on_member_id' on table 'member_roles' already exists
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/connection_adapters/abstract/schema_statements.rb:954:in `add_index_options'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/connection_adapters/abstract_mysql_adapter.rb:540:in `add_index'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:665:in `block in method_missing'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:634:in `block in say_with_time'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:634:in `say_with_time'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:654:in `method_missing'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:416:in `method_missing'
/usr/src/redmine/db/migrate/20091017213716_add_missing_indexes_to_member_roles.rb:3:in `up'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:571:in `up'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:611:in `exec_migration'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:592:in `block (2 levels) in migrate'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:591:in `block in migrate'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/connection_adapters/abstract/connection_pool.rb:292:in `with_connection'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:590:in `migrate'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:768:in `migrate'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:998:in `block in execute_migration_in_transaction'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:1046:in `ddl_transaction'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:997:in `execute_migration_in_transaction'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:959:in `block in migrate'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:955:in `each'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:955:in `migrate'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:823:in `up'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:801:in `migrate'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/tasks/database_tasks.rb:137:in `migrate'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/railties/databases.rake:44:in `block (2 levels) in <top (required)>'
```

以下が余計だったか？
```
    ->   KEY `index_member_roles_on_member_id` (`member_id`),
    ->   KEY `index_member_roles_on_role_id` (`role_id`)
```

```
mysql> ALTER TABLE `member_roles` DROP INDEX `index_member_roles_on_member_id`;
mysql> ALTER TABLE `member_roles` DROP INDEX `index_member_roles_on_role_id`;
```

今度はこれ
```
Index name 'index_custom_fields_on_id_and_type' on table 'custom_fields' already exists
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/connection_adapters/abstract/schema_statements.rb:954:in `add_index_options'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/connection_adapters/abstract_mysql_adapter.rb:540:in `add_index'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:665:in `block in method_missing'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:634:in `block in say_with_time'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:634:in `say_with_time'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:654:in `method_missing'
/usr/local/bundle/gems/activerecord-4.2.7.1/lib/active_record/migration.rb:416:in `method_missing'
```

```
mysql> show index from custom_fields;
+---------------+------------+------------------------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table         | Non_unique | Key_name                           | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+---------------+------------+------------------------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| custom_fields |          0 | PRIMARY                            |            1 | id          | A         |           0 |     NULL | NULL   |      | BTREE      |         |               |
| custom_fields |          1 | index_custom_fields_on_id_and_type |            1 | id          | A         |           0 |     NULL | NULL   |      | BTREE      |         |               |
| custom_fields |          1 | index_custom_fields_on_id_and_type |            2 | type        | A         |           0 |     NULL | NULL   |      | BTREE      |         |               |
+---------------+------------+------------------------------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
3 rows in set (0.00 sec)

mysql> ALTER TABLE `custom_fields` DROP INDEX `index_custom_fields_on_id_and_type`;
```

今度はこれ
```
Mysql2::Error: Duplicate column name 'visible': ALTER TABLE `custom_fields` ADD `visible` tinyint(1) DEFAULT 1 NOT NULL
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:80:in `_query'
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:80:in `block in query'
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:79:in `handle_interrupt'
/usr/local/bundle/gems/mysql2-0.3.21/lib/mysql2/client.rb:79:in `query'
```

```
mysql> ALTER TABLE `custom_fields` DROP `visible`;
```

今度はこれ
```
Mysql2::Error: Table 'changeset_parents' already exists: CREATE TABLE `changeset_parents` (`changeset_id` int(11) NOT NULL, `parent_id` int(11) NOT NULL) ENGINE=InnoDB
```

```
mysql> select * from changeset_parents;
Empty set (0.00 sec)
mysql> drop table changeset_parents;
Query OK, 0 rows affected (0.01 sec)
```

今度はこれ
```
Mysql2::Error: Duplicate column name 'multiple': ALTER TABLE `custom_fields` ADD `multiple` tinyint(1) DEFAULT 0
```

```
mysql> ALTER TABLE `custom_fields` DROP `multiple`;
```

```
Mysql2::Error: Duplicate column name 'inherit_members': ALTER TABLE `projects` ADD `inherit_members` tinyint(1) DEFAULT 0 NOT NULL
```

```
mysql> ALTER TABLE `projects` DROP `inherit_members`;
```

```
Mysql2::Error: Table 'queries_roles' already exists: CREATE TABLE `queries_roles` (`query_id` int(11) NOT NULL, `role_id` int(11) NOT NULL) ENGINE=InnoDB
```

```
mysql> select * from queries_roles;
Empty set (0.00 sec)

mysql> drop table queries_roles;
```


```
Mysql2::Error: Table 'custom_fields_roles' already exists: CREATE TABLE `custom_fields_roles` (`custom_field_id` int(11) NOT NULL, `role_id` int(11) NOT NULL) ENGINE=InnoDB
```

なんかエラーの内容が行ったり来たり

```
mysql> DROP TABLE `custom_fields_roles`;
```

```
Mysql2::Error: Duplicate column name 'format_store': ALTER TABLE `custom_fields` ADD `format_store` text
```

```
mysql> ALTER TABLE `custom_fields` DROP `format_store`;
```

```
Mysql2::Error: Duplicate column name 'description': ALTER TABLE `custom_fields` ADD `description` text
```

```
mysql> ALTER TABLE `custom_fields` DROP `description`;
```

```
Mysql2::Error: Table 'email_addresses' already exists: CREATE TABLE `email_addresses` (`id` int(11) auto_increment PRIMARY KEY, `user_id` int(11) NOT NULL, `address` varchar(255) NOT NULL, `is_default` tinyint(1) DEFAULT 0 NOT NULL, `notify` tinyint(1) DEFAULT 1 NOT NULL, `created_on` datetime NOT NULL, `updated_on` datetime NOT NULL) ENGINE=InnoDB
```

```
mysql> select * from email_addresses;
+----+---------+-------------------+------------+--------+---------------------+---------------------+
| id | user_id | address           | is_default | notify | created_on          | updated_on          |
+----+---------+-------------------+------------+--------+---------------------+---------------------+
|  4 |       4 | admin@example.net |          1 |      1 | 2017-01-18 13:24:38 | 2017-01-18 13:24:38 |
+----+---------+-------------------+------------+--------+---------------------+---------------------+
1 row in set (0.01 sec)

mysql> drop table email_addresses;
```

```
Mysql2::Error: Table 'roles_managed_roles' already exists: CREATE TABLE `roles_managed_roles` (`role_id` int(11) NOT NULL, `managed_role_id` int(11) NOT NULL) ENGINE=InnoDB
```

```
mysql> select * from roles_managed_roles;
Empty set (0.00 sec)

mysql> drop table roles_managed_roles;
```

```
Mysql2::Error: Table 'imports' already exists: CREATE TABLE `imports` (`id` int(11) auto_increment PRIMARY KEY, `type` varchar(255), `user_id` int(11) NOT NULL, `filename` varchar(255), `settings` text, `total_items` int(11), `finished` tinyint(1) DEFAULT 0 NOT NULL, `created_at` datetime NOT NULL, `updated_at` datetime NOT NULL) ENGINE=InnoDB
```

```
mysql> select * from imports;
Empty set (0.00 sec)

mysql> drop table imports;
```

```
Mysql2::Error: Table 'import_items' already exists: CREATE TABLE `import_items` (`id` int(11) auto_increment PRIMARY KEY, `import_id` int(11) NOT NULL, `position` int(11) NOT NULL, `obj_id` int(11), `message` text) ENGINE=InnoDB
```

```
mysql> select * from import_items;
Empty set (0.00 sec)

mysql> drop table import_items;
```

```
Mysql2::Error: Table 'custom_field_enumerations' already exists: CREATE TABLE `custom_field_enumerations` (`id` int(11) auto_increment PRIMARY KEY, `custom_field_id` int(11) NOT NULL, `name` varchar(255) NOT NULL, `active` tinyint(1) DEFAULT 1 NOT NULL, `position` int(11) DEFAULT 1 NOT NULL) ENGINE=InnoDB
```

```
mysql> select * from custom_field_enumerations;
Empty set (0.00 sec)

mysql> drop table custom_field_enumerations;
Query OK, 0 rows affected (0.01 sec)
```

これでエラーが出なくなった。
Redmineにアクセスすると、これまでのデータが反映されていることが確認できた。

あとは添付ファイルのマイグレーションをやるが、S3用のプラグインを入れているので、その手順は別途検証することにする。

1.7GBのファイルも同じ方法でマイグレーションできるので、とりあえず方法は割愛する。
