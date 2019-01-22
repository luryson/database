# Migration 重构说明文档

## 目录结构

    |-database
        |-drivers               # 数据库驱动
        |-lib                   # 依赖jar包
        |-migration             # DDL 迁移仓库
            |-environments      # 环境配置 
            |-scripts           # 迁移脚本
        |-seeds                 # DML 迁移仓库
            |-development       # 测试环境数据仓库
                |-evironments   # 环境配置
                |-scripts       # 迁移脚本 
            |-production        # 生产环境数据仓库
                |-evironments   # 环境配置
                |-scripts       # 迁移脚本  
        migrate                 # *nix环境下执行ddl迁移
        migrate.cmd             # windows环境下执行ddl迁移
        seed                    # *nix环境下执行dml迁移
        seed.cmd                # windows环境下执行dml迁移
        
        
## DDL操作说明
> 说明: DDL保存在一个仓库中（`./migration/scripts`），保证各个环境之间数据库结构的一致
#### 命令行工具
> * *unix: `./migrate`
> * windows: `migrate`

#### 环境配置
* 对应目录 `./migration/environments`
* 新增配置时拷贝任意文件，重命名为对应环境名（`test.properties`）,windows用户需将文件内 `# script_char_set=UTF-8`行解除注释，防止出现乱码
* 修改 `url` `username` `password` 三项，其他配置项按需调整 
      
#### 命令说明
```
Usage: migrate command [parameter] [--path=<directory>] [--env=<environment>] [--template=<path to custom template>]

Commands:
  info               Display build version informations.
                     
  init               Creates (if necessary) and initializes a migration path.
                     初始化一个迁移版本库，生成对应的目录等
                     
  bootstrap          Runs the bootstrap SQL script (see scripts/bootstrap.sql for more).
                     基础的启动信息，用在一个已有的数据库上，记录起始的库表信息；
                     
  new <description>  Creates a new migration with the provided description.
                     创建一条新的迁移，description用于简单描述本次迁移的内容
                     
  up [n]             Run unapplied migrations, ALL by default, or 'n' specified.
                     运行未被执行的迁移，默认执行全部，或者通过`n`指定一次执行的条数
  
  down [n]           Undoes migrations applied to the database. ONE by default or 'n' specified.
                     回滚已执行的迁移，默认一次执行一条回滚，或者通过`n`指定一次执行的条数
  
  version <version>  Migrates the database up or down to the specified version.
                     操作数据库迁移或回滚至指定`version`
  
  pending            Force executes pending migrations out of order (not recommended).
                     强制执行所有处于`pending`状态的异常的迁移（不推荐使用）
  
  status             Prints the changelog from the database if the changelog table exists.
                     打印当前数据库的迁移状态
  
  script <v1> <v2>   Generates a delta migration script from version v1 to v2 (undo if v1 > v2).
                     按指定的v1,v2生成增量的迁移脚本（回滚的话，v1 > v2）
```

#### 常用命令（linux为例）

* `./migrate new`: 创建新的迁移sql

* `./migrate status`: 查看当前sql执行情况

* `./migrate up`: 执行sql（默认执行所有未执行过的sql）,可指定 `[n]`参数，选择一次执行`n`条迁移,如 `migrate up 1`执行一条迁移，执行顺序按照迁移脚本版本从前往后执行

* ./migrate down: 执行回滚语句（默认执行一次回滚）,同样可以指定`[n]`参数，选择一次回滚的迁移条数

* `./migrate [cmd] —-env= `: 指定执行环境，默认为`development`，env对应的选项为`./environments`目录下的配置文件名

* 执行后的sql会在change_logs表中留存一条操作记录，可以通过这个表来维护一个sql的版本管理，通过`./migrate status --env=`来查看当前所有迁移脚本在指定env下的执行情况

#### 注意事项

* 本工具不支持同一个sql脚本执行多条命令，所以尽量拼成一条sql来执行，create语句一条命令对应一个文件；

* 所有修改保证在新的脚本中书写，不要在已有脚本上进行修改，否则协作中或者生产中的环境无法得到同步；

* 在某个环境执行`up`命令遇到中间有`pending`的脚本时，可使用`script [from] [to]`命令导出指定sql，然后通过命令行或者工具执行

* 发布生产环境时，可打包`migration`目录到生产环境，解压后执行`status`命令，查看当前脚本库在生产环境上的执行情况，然后使用`script [from] [to]`命令，导出需要执行的sql

* 当前各环境存在历史数据库表结构不一致的问题，执行`status`命令时会出现pending的情况，遇到这种情况，需要手动检查每条pending的sql在目标环境数据库中的执行情况
    * 当目标数据库中未执行时，使用`pending`命令强行执行
    * 当目标数据库中已经执行过时，使用`script [from] [to]`命令导出对应的sql,然后选择性的将版本信息录入到目标库中


## DML操作说明
> 定义：DML中我们定义环境为 ${environment}
>
> 说明: DML按照不同的环境保存在各自环境仓库下的scripts目录下（`./seeds/${environment}/scripts`），保证各个环境数据的隔离
> 开发/测试环境可用可用一个仓库，通过指定`--env=${environment}`参数执行对应环境的数据迁移
#### 命令行工具
> * *nix: `./seed --path=seeds/${env_repo} [--env=${environment}]`
> * windows: `seed --path=seeds/${env_repo} [--env=${environment}]`

#### 环境配置
* 对应目录 `./seeds/${environment}/environments/development.properties`,使用默认的`development.properties`定义，
* 新增环境，拷贝 `./seeds`下任意目录后，修改对应的`environments/delopment.properties`文件
* 修改 `url` `username` `password` 三项，其他配置项按需调整 

#### 注意事项：
* 不同环境的版本号建议保持一致，便于排查；本地开发时，在development目录下`new`出新的迁移脚本，然后拷贝至其他环境对应的目录下；

#### 其余使用请参考 DDL 操作说明


<br/>
## 注意：
> migration和seeds目录下的`${environment.properties}`配置有所不同，新增环境配置时请注意拷贝对应的配置或者目录，请勿串用
