# mysql运维

[toc]



## mysqldump

列出匹配的表名：

```shell
mysql -uroot -pkS4pKJF_3kfdsfOoJ -D ccloud -Bse "SHOW TABLES LIKE 'wp_led_flowfee%'"
```

mysqldump:

```shell
#导出指定表：
mysqldump -uroot -pkS4pKJF_3kfdsfOoJ ccloud $(mysql -uroot -pkS4pKJF_3kfdsfOoJ -D ccloud -Bse "SHOW TABLES LIKE 'wp_led_flowfee%'") > /var/lib/mysql/sea_flowfee.sql
```

mysqldump过滤条件：

```shell
mysqldump -uroot -pkS4pKJF_3kfdsfOoJ ccloud wp_postmeta --where=" meta_key in ('_terminal_group','_terminal_led') " > /var/lib/mysql/wp_postmeta_beu.sql
```

## mysql查看配置项

```mysql
#比如查看最大连接数
show variables like 'max_connections';
```



## 存储过程

创建热力图分表：

```mysql
CREATE DEFINER=`root`@`localhost` PROCEDURE `createHeatMapTable`()
BEGIN
    DECLARE
        table_prefix VARCHAR ( 30 );
    DECLARE
        table_number INT DEFAULT 0;
    DECLARE
        table_name VARCHAR ( 30 );
    DECLARE
        sql_text1 VARCHAR ( 2000 );
    
    SET table_prefix = 'clt_gps_heat_map';
    WHILE
        table_number < 128 DO
            
    SET table_name = CONCAT( table_prefix, table_number );
        
        SET @sql_text1 = CONCAT( 'CREATE TABLE IF NOT EXISTS ', table_name, ' (
            `ID` int UNSIGNED NOT NULL AUTO_INCREMENT,
            `term_id` int NOT NULL,
            `datetime` datetime NOT NULL ON UPDATE CURRENT_TIMESTAMP,
            `s2_cell_id` varchar(16) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
            `count` int UNSIGNED NULL DEFAULT NULL,
            PRIMARY KEY (`ID`) USING BTREE,
            INDEX `term_date_geohash_index`(`term_id`, `datetime`, `s2_cell_id`) USING BTREE
        ) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE =utf8mb4_general_ci ROW_FORMAT = Dynamic' );
        PREPARE stmt 
        FROM
            @sql_text1;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
        
        SET table_number = table_number + 1;
        
    END WHILE;
    
END
```

mysql默认分隔符是`;`，一般会重新定义分隔符防止存储过程被分开：

```mysql

delimiter //
CREATE PROCEDURE `hello`(IN n int)
BEGIN
存储过程代码
END
//

CALL hello(100);
```

