---
title: 'setup-go-admin'
parent: 學習心得
---

## Why did you choose this?

我試過

* [GoAdminGroup/go-admin](https://github.com/GoAdminGroup/go-admin): 自動生成管理table的code的部分悲劇，一直失敗，還要自己改生出來的code
* [qor/admin](): 跑readme的範例就直接爆類似segment fault的錯!? 這真的是現代PL嗎?

目前有成功透過自動生成code的功能去操作db的就只有[這一款](https://github.com/go-admin-team/go-admin)

## Setup DB (mysql 15.1)

### 用mysql的密碼認證
原本是用`auth_socket`會看linux的帳號與mysql的密碼，所以如果是用root去連
要用`sudo mysql -u root -p`，sudo不可少，所以會讓其他程式沒辦法用root登

這裡是為了測試go-admin所以先用比較簡單(十分不安全)的方法，應該要另外開個這帳號給go-admin，但這只是測試so....

如果想開另一個帳號與設定mysql見[這裡](https://stackoverflow.com/questions/39281594/error-1698-28000-access-denied-for-user-rootlocalhost)的第二個方法

```
$ sudo mysql -u root # I had to use "sudo" since is new installation

mysql> USE mysql;
mysql> UPDATE user SET plugin='mysql_native_password' WHERE User='root';
mysql> FLUSH PRIVILEGES;
mysql> exit;
```

### init DB & 建新的database
```
$ sudo mysql_secure_installation

$ mysql -u root -p
mysql> craete database forgo;
```

## Setup go-admin

### clone codes
```
mkdir demo && cd demo
git clone https://github.com/go-admin-team/go-admin.git
git clone https://github.com/go-admin-team/go-admin-ui.git
```

### change config && build
```yaml
# ....
  database:
    driver: mysql
    source: root:mypw@tcp(127.0.0.1:3306)/forgo?charset=utf8&parseTime=True&loc=Local&timeout=1000ms # change this
  gen:
    dbname: forgo # change this
    frontpath: ../go-admin-ui/src
```

改完後，在go-admin的資料夾中跑api server
```
go build
./go-admin migrate # 生table
./go-admin server
```

之後是到go-admin-ui跑ui server
```
npm i
npm run dev
```

### generate code for our table
假設有以下table
```sql
CREATE TABLE `article` (
  `title` varchar(128) DEFAULT NULL COMMENT '标题',
  `author` varchar(128) DEFAULT NULL COMMENT '作者',
  `content` varchar(255) DEFAULT NULL COMMENT '内容',

  /* 下面的column是go-admin要用的，不知道能不能自定義 */
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '编码',
  `created_at` timestamp NULL DEFAULT NULL,
  `updated_at` timestamp NULL DEFAULT NULL,
  `deleted_at` timestamp NULL DEFAULT NULL,
  `create_by` int(11) unsigned DEFAULT NULL,
  `update_by` int(11) unsigned DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_article_deleted_at` (`deleted_at`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4 COMMENT='文章';
```

1. 先選要被管理的table，就找圖上的順序
![](https://i.imgur.com/67D9pU5.png)

2. 接著是產生code
![](https://i.imgur.com/AL7gESZ.png)

3. 放到旁邊的菜單
![](https://i.imgur.com/Voe23L2.png)

4. 關掉server，`go build`，再開一次
我不確定要不要重編，但保險?

剛加完菜單直接去按會吃

`2020-10-10 02:25:48 [WARING] GET /api/v1/sysfiledirList GetUserIdStr 缺少identity`

的錯...

5. 之後就可以CURD
![](https://i.imgur.com/t9sjkEm.png)