1. 在表的列的类型选择上，尽量选择对应的类型，如日期就用date，不要用varchar，查询速度会降低。

   图中a表和b表均一列一条数据且数据一致，a类型为varchar，b为date；

![不同类型查询速度比较](E:\developer\这_MySQL调优笔记\不同类型查询速度比较.png)

2. 避免NULL
   1. 在MYSQL里，null是不=null的，需要用is null
   2. null会占用更多存储空间。
   3. null会导致列的索引以一种更为复杂的方式去统计，减慢效率，以及导致复合索引失效。
   4. 尽量避免而已，不必强行为之，具体看业务情况。

3. 整形数据类型选择，MYSQL中整形类型分为tinyint，smallint，mediumint，int，bigint，从小到大分别占用8,16,24,32,64位大小，在选择整形类型时，尽量选择满足需求的最小大小的类型。

   下图c表为tinyint，d为int。

   ![image-20210509161228049](E:\developer\这_MySQL调优笔记\tinyint和int查询速度比较.png)

4. varchar是可变类型，除去本身数据大小所占字节数，在0-255期间会采用额外一个字节用于保存长度，在大于255时使用额外两个字节存储长度。varchar实际上本身一个大小是指一个字符，varchar(20)是指长度为20的字符串，中文也只占一个位置。虽然用varchar(2)和varchar(20)去存一个字符使用的磁盘大小是一样的，但是其实占用的空间会比本身更多一些，就像操作系统的页对齐一样。MYSQL在加载时，不知道实际数据的长度，会根据定义长度分配内存空间，导致浪费内存。

   varchar一般存放变化较大的数据，以及更新较少的情况，因为每次更新会导致从新计算实际长度以及会使用额外空间保存长度。对于多字节的字符，可以选用varchar。但实际上一般为了省事不出问题，还是会选用varchar。

![image-20210509163719136](E:\developer\这_MySQL调优笔记\使用空间和占用空间.png)

5. char的查询效率确实大于varchar。

   ![image-20210509165327452](E:\developer\这_MySQL调优笔记\char和varchar效率比较.png)

   char会自动去掉数据末尾的空格。

   char最大255长度。

   char固长，空间浪费，由于换取了效率。

   char一般用于固定长度的数据，如标记数据，加密数据如MD5。经常更新的字符串也可以用char以提高效率。	

6. BLOB（二进制存储）和TEXT（字符存储）一般不使用，大文件如果直接存在数据库里存储，查询匹配时间过长，一般会采取将数据量大的文件存入其他机器如FTP服务器，数据库只存储其地址即可，所以这两个基本不会用到。

7. datetime占8个字节，可以精确到毫秒，无法进行时区配置，保存时间年份范围1000-9999。

   timestamp占4个字节，可以精确到秒，可以进行时区配置，保存时间年份1970-2038，自动更新。

   ​	时区配置的优势在于，当项目分布广阔到了多个国家与地区，在欧美地区和中国地区同时操作数据时，电脑	的时间会因为时区而不一样，timestamp可以解决这个问题。

   ​	而自动更新在于创建表时或列时，可以利用timestamp进行自动修改该列时间数据设置。

   ```mysql
   CREATE TABLE `timestampTest` (
     `id` int(11) NOT NULL AUTO_INCREMENT,
     `name` varchar(20) DEFAULT NULL,
     `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
       
       --on update 意为在修改表其他数据时，该列数据自动按时间更新
     `last_modify_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
       
       
     PRIMARY KEY (`id`)
   ) 
   ```

   date仅占3个字节，而且MYSQL,ORACLE均对date有丰富的自带的计算函数，date只能精确到日，年份范围为1000-9999；

8. MYSQL也是有枚举的

```mysql
create table my_enum(

　　sex enum('男','女','未知')

)
```

​	MYSQL的枚举更像JAVA的String常量池，都是常量存在文件中，并每个枚举数据都会自动标记序号，序号规则为定义枚举数据时的顺序，如上图为 【男1 女2 未知3】，数字1开始然后升序。

​	插入枚举数据。

```mysql
INSERT INTO myenum VALUES ('未知'),('未知'),('女'),('女'),('男'),('男')
```

​	查询枚举数据

```mysql
SELECT * FROM myenum WHERE sex =1;
SELECT * FROM myenum WHERE sex =2;
SELECT * FROM myenum WHERE sex =3;
SELECT * FROM myenum WHERE sex ='男';
```

sex=1和sex='男' 是等价的。

9. MYSQL主键分为代理主键和自然主键，选主键时尽量选用代理主键，代理主键无意义，自然主键为业务数据中不会重复的数据，但由于自然主键在业务层代码出问题生成重复数据的情况下会出问题，所以尽量选择无意义的在插入时再生成的代理主键。

   组合主键，在使用自然主键时，当该列主键可能重复时（如 人名），同时用多个列共同组成主键（如 人名，性别，身份证号三个一起），以防止主键重复的问题。

   ```mysql
   create table test(
       id int(10) not null auto_increment,
       name varchar(20) not null,
       sex int(1) not null,
       primary key (id,name,sex)
   );
   ```

10. MYSQL字符集的选择，MYSQL默认的是latin1 （拉丁1），如果数据库不会存中文或其他拉丁语系外的文字数据，latin1完全可以胜任，但是如果有中文，可以选择UTF8,GBK,UTF8mb4，UTF8是国际标准，当数据库有多国的数据，可以使用UTF8。UTF8占3个字节，GBK占2个字节，当只有中文和拉丁语系数据，又要节省空间时，可以选用GBK。但是当数据库存有emoji的表情时，由于emoji占4个字节，UTF8以及无法正常存储带emoji的数据了，此时应该使用UTF8mb4来存储，mb4意为most byte 4，最大为4个字节，一般如果不是很受限，可以选用UTF8mb4，但UTF8mb4在mysql5.5后才出现。

11. MYSQL存储引擎的选择，MYSQL存储引擎很多，一般是MyISAM和InnoDB，MyISAM是老版本的引擎，InnoDB 5.6以后才出现。

    MyISAM的索引类型为非聚簇索引，支持锁表，支持全文索引，不支持事务，不支持锁行，不支持外键，适合作为大量查询时的引擎。

    InnoDB的索引类型为聚簇索引，支持锁表，支持全文索引，支持事务，支持锁行，支持外键，适合作为大量增删改时的引擎。

    聚簇索引意为数据文件和索引文件是存放在一起的，非聚簇则反之。

    两种引擎是可以互相转换的，但是成本极高。

12. 在ORACLE中，视图有普通的视图和物化视图，普通视图的本质就是一句SQL而已，本身不自带数据，而物化视图是在物理空间中真实存在的一张表，以供多次查询时提供高效率，但其存在更新问题，其有两种更新模式，一是在基表更新时更新，二是在查询视图时再更新。

    而MYSQL只有普通视图。

13. 在设计表时，在查询一个数据总是需要join联查时，由于join后产生的临时表数据很大，IO操作也多，不如直接将需要的数据列加入到另一个表。虽然有一些冗余但是以空间换时间，看情况见仁见智。

    但是在故意冗余数据的同时，需要注意冗余数据列的维护问题，在原始数据发生变化时，也要使冗余数据发生更新，保持一致性，略微增加了维护的成本。

14. 适当拆分数据列，当一个数据列数据很庞大但却并没怎么用到时，可以将其放入另一个表，来保证查询表时不会做很多无用的匹配工作。

15. 使用explain关键字查看MYSQL的执行计划（在SQL前加explain 即可），执行计划可以看到实际上MYSQL执行SQL语句的各个步骤。虽然执行计划增删改查都可以用，但一般用于查询居多。

    ![image-20210510175654943](E:\developer\这_MySQL调优笔记\执行计划初步演示.png)

    id代表执行顺序，id越大的越先执行，id相同的从上往下依次执行，table列可以看到表。

    select_type代表MYSQL认为该次查询步骤的复杂程度。

    possible_keys代表该次可能用到的索引。

    key代表实际用到的索引。

    key_len代表所=索引长度，越短越好，长了IO连接次数变多。

    ref 是指用到的索引的哪一列被使用，一般显示const。

    rows 指该步骤所 查询/影响 的预估行数，不一定准确。

    extra 附加信息，有错报错，也有可能是提示，但没有最好，一般有提示就是有可以优化的地方，use index除外，use index是指索引覆盖，代表没有产生回表。

    > ```mysql
    > --using filesort:说明mysql无法利用索引进行排序，只能利用排序算法进行排序，会消耗额外的位置
    > explain select * from emp order by sal;
    > 
    > --using temporary:建立临时表来保存中间结果，查询完成之后把临时表删除
    > explain select ename,count(*) from emp where deptno = 10 group by ename;
    > 
    > --using index:这个表示当前的查询时覆盖索引的，直接从索引中读取数据，而不用访问数据表。如果同时出现using where 表名索引被用来执行索引键值的查找，如果没有，表面索引被用来读取数据，而不是真的查找
    > explain select deptno,count(*) from emp group by deptno limit 10;
    > 
    > --using where:使用where进行条件过滤
    > explain select * from t_user where id = 1;
    > 
    > --using join buffer:使用连接缓存，情况没有模拟出来
    > 
    > --impossible where：where语句的结果总是false
    > explain select * from emp where empno = 7469;
    > ```

    type列看到的是查询的方式是ALL，意为全表查询，一般有ALL的都是需要优化的。

    ![image-20210510180310159](E:\developer\这_MySQL调优笔记\使用索引查询type变化.png)

    在ggg列加了key索引后，type就变为了index，index意为全局索引查询方式。

    测试的表如下

    ![image-20210510190659780](E:\developer\这_MySQL调优笔记\执行计划测试用表结构.png)

    而执行计划的type按效率高低由低到高可分为：

    1. ALL,全表扫描，最low的，要避免。

    ```mysql
    EXPLAIN SELECT * FROM ggg #all
    ```

    2. INDEX,采用全索引扫描，需要扫描整个索引树，比ALL快一些。

    ```mysql
    EXPLAIN SELECT ggg FROM ggg #index
    ```

    3. RANGE,正式环境中一般可使用的最低标准，索引列+范围查询。(in(), between ,> ,<, >=)

    ```mysql
    EXPLAIN SELECT * FROM ggg WHERE d>1 #range d为primary
    ```

    ​	实际上，只有当条件列的索引为primary和unique时才是range，key和fulltext依旧是all

    ```mysql
    EXPLAIN SELECT * FROM ggg WHERE ggg>1 #all ggg为key
    ```

    ​	所以说啊，避免范围查询不是没道理的，对了一半。（本次测试MYSQL为5.7.33）

    4. index_subquery 没整出来 索引关联子查询
    5. unique_subquery 没整出来 唯一索引关联子查询
    6. index_merge 多种索引组合使用

    ```mysql
    EXPLAIN SELECT * FROM ggg WHERE d=1 OR ggg='1' #index_merge
    ```

    7. ref_or_null ref的有is null的形式，本来ref很快的，加了is null操作就变慢了

    ```mysql
    EXPLAIN SELECT * FROM ggg WHERE ggg='123' OR ggg IS NULL #ref_or_null
    ```

    8. ref 使用索引查询，但是不能是唯一索引

    ```mysql
    EXPLAIN SELECT * FROM ggg WHERE ggg='123' #ref
    ```

    9. eq_ref primary和unique专属，在使用外连接查询时，条件列均有唯一索引时的type

    ```mysql
    EXPLAIN SELECT * FROM ggg LEFT JOIN mytable ON ggg.bbb=mytable.id #ALL+eq_ref
    ```

    ​	其实外连接分为两步执行，第一步时eq_ref，第二步是ALL。因为要左外要查询左边的ggg表的全部数据，

    所以第二步查ggg是ALL，查第一步mytable是eq_ref。

    10. const 直接指定常量去查，和8差不多，但人家是primary和unique，就是比你key和fulltext快

    ```mysql
    EXPLAIN SELECT * FROM ggg WHERE d=1 #const
    ```

    ​	这里有个注意的问题，若果写作where ggg=1 因为ggg是varchar类型，1是int类型，一旦类型不匹配，直接变为ALL，所以选数据类型要匹配好啊。（JYYH全varchar2存数据，查询慢成狗）

    11. system，属于传说级的type，不论是网上还是自己搞，连个样例都整不出来，在座的各位属实Five。
    12. 其实还有个NULL，当MYSQL觉得更根本不需要执行计划的时候，就是NULL了，如：

    ```mysql
    EXPLAIN SELECT 1 FROM DUAL; #null
    ```

    mysql给的信息是：No tables used；

    ```mysql
    EXPLAIN SELECT * FROM ggg WHERE d=10 #null
    ```

    mysql给的信息是：no matching row in const table；没有匹配这个常量的行。

    在实际开发过程中，尽量不要查不存在的ID，会导致全表扫描，数据量大可能会崩，和缓存穿透一个道理。

16. MYSQL中存储索引的数据结构选择。存储索引的数据结构由选择的引擎决定，innoDB是选择的B+树，Memory选择的是hash，还有两种是R-tree和full-text；

    1. Hash,由于使用hash存储必须将所有数据文件加载到内存中，耗费内存空间，所以只有memery才使用hash，hash在做等值查询时速度很快，但是在范围查询时速度慢。hash使用hash表存储，和hashMap是一样的，一个数组利用散列的方式，取余数的方式存储，相同余数同一数组位置用链表。

    ![image-20210511165129105](E:\developer\这_MySQL调优笔记\hash存储.png)

    2. 二叉树，普通的二叉树会在数据一直大于或小于根节点时造成倾斜问题，甚至变成链表，是MYSQL最早期存储索引的数据结构，但问题严重，会造成深度很大导致查询很慢，因为每遍历一个节点，就是一次IO，IO很费时间，要避免过多。

       ![image-20210511165227895](E:\developer\这_MySQL调优笔记\二叉树存储.png)

    3. 二叉搜索树，因为会对数据排序，所以可以通过二分法查找，是MYSQL的第二个存储索引的数据结构，但还是深度很大，慢。

    4. AVL平衡树，也是二叉树，但是会自旋，以保证最大深度叶子节点不会比最小深度叶子结点的深度大得超过1，即最多比小的深1。在1-9按序插入会变成如下形式。

    ![image-20210511165916328](E:\developer\这_MySQL调优笔记\AVL树存储.png)

    ​		但是，不断的自旋操作还是太浪费时间与空间了。

    5. 红黑树，也是二叉树的一种，也会自旋，但不会自旋得那么的频繁，以保证在增删改操作时，不那么的耗性能，但是由于深度会比AVL要深一些，查询不如AVL。

    ![image-20210511175537167](E:\developer\这_MySQL调优笔记\红黑树存储.png)

    ​		特点是

    ​		1. 根节点和叶子结点必为黑色

    ​		2. 不会出现两个红色节点相邻

    ​		3. 所有路径所经过的黑色节点数量必定相同，上图为3.

    ​		4. 最深叶子深度不会比最浅叶子深度的两倍还大，最多两倍。

    红黑树也是MYSQL用于存索引的版本之一，不过由于还是二叉树，无可避免的有数据量大时深度过深的问题，查询慢。

    6. B树，即B-Tree，所谓的B-树也就是B树，B树不再是二叉树了，是一种船新版本，是范围查询，分而治之的手段，根节点有数据的范围，根据需要查询的数据，将该次查询分配到对应的数据范围的子节点，不断地分配范围，直到查到数据，一般3-7阶，也就是深度一般3-7。

       ![image-20210511183509222](E:\developer\这_MySQL调优笔记\B树存储.png)

       上图为三阶B树，在不同的数据来时，先查根节点范围，再子节点范围，再查数据，叶子结点是不联通的，不可以互相流动。B树的每个节点都是有自带数据的，一般自带数据即判断范围的数据，如根节点自带25，87的数据。

    7. B+树，MYSQL在B树的基础上做了优化改进，由于B树是有着自带数据的，而数据本身如果很大，将会使节点的大小变得很大，不利于存储，所以B+树就直接去除了非叶子节点的自带数据，使节点仅用于存储范围数据，将节点变小，从而提高性能。（倘若按照InnoDB一页16KB的话，算一个节点一页16KB，一个范围数据10B，一个节点可以存1600个范围数据，但如果有自带数据如果很大有10KB，那么只剩下6KB用于存储范围数据了，性能变差）

       ![image-20210511184603552](E:\developer\这_MySQL调优笔记\B+树存储.png)

       B+树的叶子节点可流动互通，增加了缓存命中率，其实MYSQL很多调优其实都为了增加缓存命中率，但十分残念的是MYSQL8以后取消了缓存。

       B+树是目前MYSQL最常使用的存储索引的数据结构。最新了。

    8. 顺便一提B\*树，B\*树就是在B树基础上，除了叶子节点，非叶子节点也可以流动了，但对于B+树来说没意义，毕竟B+树非叶子节点没数据可用。

17. 虽然MyISAM和InnoDB都用的B+树，但由于MyISAM是非聚簇索引，索引文件和数据文件是分开了的，所以最后叶子结点存的不是真实的数据，而是真实数据所在的文件的物理地址，还需要一次IO去找到真实的数据，而InnoDB是聚簇索引可以直接拿到数据，更快。

18. 建立表时，尽量建立主键或者有唯一索引的列，否则MYSQL会自动创建一个rowID做主键，但是此时，会产生回表， 

    因为只有在主键索引和唯一索引的B+树的叶子节点才会存该行全部数据，其他的索引只会存该列和有主键和唯一索引列的数据，所以当使用一个非主键非唯一的索引列查select * from xxx where 非主键>xxx时，会变成全表查询ALL的道理就在这，因为该列的索引树没其他无索引列的数据，而使用主键select * from xxx where 主键 > xxx会是range,而select  非主键 from xxx where 非主键>xxx是index。

    要避免回表，查询时尽量用主键做条件列，否则产生回表将会查两次。

    回表：查询的列未在条件列的索引树中有数据，改为查询主键索引树。

19. 索引的作用
    1. 减少MYSQL数据库需要检索的数据量
    2. 避免了数据库在查询后排序和建立临时表
    3. 是查询时的随机IO变为有序的IO

20. 索引覆盖，其实和回表相对应，回表是索引树没该列数据，而索引覆盖是索引树有该列数据，通过一个索引树就查询到所有数据，没发生回表，就叫索引覆盖。
21. 最左匹配原则，针对于组合索引的原则，当定义组合索引时

```mysql
CREATE TABLE `user2` (
  `userid` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(20) NOT NULL DEFAULT '',
  `password` varchar(20) NOT NULL DEFAULT '',
  `usertype` varchar(20) NOT NULL DEFAULT '',
  PRIMARY KEY (`userid`),
    #建立组合索引
  KEY `a_b_c_index` (`username`,`password`,`usertype`)
    
)
```

此时username在最左边。

如果使用条件查询

```mysql
EXPLAIN SELECT * FROM user2 WHERE username='1';
```

由于username在最左边，所以此次查询会走索引

![image-20210512171249575](E:\developer\这_MySQL调优笔记\最左匹配原则1.png)

在此使用条件查询

```mysql
EXPLAIN SELECT * FROM user2 WHERE PASSWORD='1';
```

此时最左为password，不会走组合索引。

![image-20210512171436660](E:\developer\这_MySQL调优笔记\最左匹配原则2.png)

但是不是ALL而是INDEX，为啥呢，略显奇怪,但不管怎么说，速度变慢了许多。

但是其实只要最左的条件列(username)出现了，就行了，不论在不在最左边，因为优化器在SQL执行时会自动优化。

```mysql
EXPLAIN SELECT * FROM user2 WHERE username='1' AND PASSWORD='1';

EXPLAIN SELECT * FROM user2 WHERE PASSWORD='1' AND username='1';
```

![image-20210512171649227](E:\developer\这_MySQL调优笔记\最左匹配原则3.png)

组合索引在遇到范围查询时将停止后面的条件列使用索引，而且如果是A,B,C如果WHERE A C ，A列将使用索引而C不会，因为没B。

22. 索引下推。在一般的查询过程中，是先根据索引把全部数据拿到，再根据where后的条件去筛选数据。然而如果使用索引下推优化，就不用一开始就去取全部的数据那么浪费时间空间了，一开始先直接让条件列的条件在该列索引中去筛选，不先去取数据，筛选完了再去取对应行数据。

    如where number >123 先在number 列的索引树里把不打于123的去掉，此时数据变少，再取数据，节约时间空间。	

23. 索引并不是建立得越多越好，因为索引也需要维护的，所谓的维护就是在增删改操作时，不仅数据需要改变，其对应的索引也需要改变，所以说如果索引多了，会影响增删改的性能，一般INDEX不要超过5个（网上建议）。

24. 索引会对列自动排序，不再需要orderby，因为orderby是全排序，更慢。

    测试：

    ![image-20210512175230008](E:\developer\这_MySQL调优笔记\索引排序前.png)

    此时对data列加索引

    ![image-20210512175456144](E:\developer\这_MySQL调优笔记\索引排序加索引.png)

    然后结果是

    ![image-20210512175529689](E:\developer\这_MySQL调优笔记\索引排序后.png)

    但是再加一列无索引列后仍然select * ，排序将会失效，只有select有索引列时才会自动排序。

    ![image-20210512175907960](E:\developer\这_MySQL调优笔记\查询额外的无索引列时排序失效.png)

25. 索引在匹配时，可以匹配列的前缀，也就是可以完全匹配或者只匹配前几行，所以就导致了一个现象，就是aaa like '213%'会走aaa的索引，而aaa like '%213'不会走aaa的索引，所以like使用时尽量不要再前面写%如果可以的话。

    ![image-20210512185453241](E:\developer\这_MySQL调优笔记\like时%写后面.png)

    ![image-20210512185608310](E:\developer\这_MySQL调优笔记\like时%写前面.png)

26. 哈希冲突，在对数组存值时，当使用的哈希算法运算多个值得出相同hash值时，就产生了哈希冲突。

    解决：

    1. 开放地址方法，当数组中已经存在此时得出的hash值数据，对当前新数据的hash值进行运算（加1，平方，随机生成等），使其直到不冲突为止。感觉很low。
    2. 链式地址方法，也就是HashMap所采用的方式，相同的Hash值得数据采用链表一起存储。
    3. 建立公共溢出区，如其名，将冲突的数据全部放在另一个区域。（真的大丈夫？）
    4. 再哈希法，说得高深，就是叫你换个好点的哈希算法，别用什么取模什么的LowB算法。

27. MYSQL中也有哈希索引，但只有Memory引擎才显示的支持这种索引，这种索引的特点是，一旦有任何的范围查询就失效，也许这就是网上说不要用任何范围查询的根源？明明InnoDB中范围查询是可以使用的，只是要注意一些问题。哈希索引的由于其Hash表的数据结构优势，查询速度很快。但哈希索引无法进行自动排序，同时如果Hash算法太菜，导致哈希冲突过多，将使维护成本提高。

28. 索引覆盖的优势
    1. 由于索引中的列永远小于等于整行的列，所以如果产生了索引覆盖的话，将会极大减少数据的访问量，速度快。
    2. 由于索引是排好序的，而磁盘中数据是随机存储的，在IO密集型的范围查询情况下，访问索引的匹配速度会快于磁盘中的随机查找。
    3. 如果是非聚簇索引，如MyISAM引擎，只会缓存索引在内存中，而数据依旧在电脑硬盘中，需要请求操作系统的资源管理器去间接地进行调用，耗性能。

#### 实际优化技巧

29. 不要在where 后面的表达式做运算，将会导致type等级大大降低。

```mysql
EXPLAIN SELECT * FROM ggg WHERE d = 1;#const
EXPLAIN SELECT d FROM ggg WHERE d + 1= 2;#index
EXPLAIN SELECT * FROM ggg WHERE d + 1= 2;#all
```

30. 查询时尽量使用where 主键 查询，会产生索引覆盖，减少回表的概率。

```mysql
EXPLAIN SELECT * FROM ggg WHERE ccc = 1;#ref
EXPLAIN SELECT * FROM ggg WHERE d = 1;#const
```

31. 在对于blob，text，varchar的大长度  的数据类型建立索引时，如果索引把这些类型的列的数据全部存储，将会导致消耗大量的空间，而且在匹配时也非常耗时，所以要使用 前缀索引以减少索引匹配的数据的字符数。

```mysql
alter table ggg add key(ggg(5));#只把前五个字符作为索引
```

32. 在外连接 join on 时，如果可以关联多个列，尽量关联基数（Cardinality）小的列，基数指数据表中不重复数据的个数，MYSQL中的基数是由HyperLogLog算法得出的。
33. 如果查询的数据需要排序，尽量使用order by主键，其他的列将会使用filesort文件排序。

```mysql
EXPLAIN SELECT * FROM ggg ORDER BY d;#index
EXPLAIN SELECT bbb,d FROM ggg ORDER BY bbb;#index
EXPLAIN SELECT * FROM ggg ORDER BY bbb;#all Using filesort
```

34. 范围查询会使后面的列索引失效，但是测试没测试出来。
35. 能用union all就用all，union多了一个distinct操作，导致慢。

```mysql
EXPLAIN 
SELECT * FROM ggg WHERE d=1 
UNION ALL
SELECT * FROM ggg WHERE d=2 
```

![image-20210516184757944](E:\developer\这_MySQL调优笔记\unionAll计划.png)

此时如果是union ，会多一个ALL级的操作。

```mysql
EXPLAIN 
SELECT * FROM ggg WHERE d=1 
UNION 
SELECT * FROM ggg WHERE d=2  
```

![image-20210516185011848](E:\developer\这_MySQL调优笔记\union计划.png)

36. 对于in 和 or的速度比较问题，当in 和 or对于主键的查询时，区别不大，or要略微的快一些，但差距在10的-5次方级别。但是一旦in和or对于非主键列来说，in的效率大大快于or的效率。

    ![image-20210516190025375](E:\developer\这_MySQL调优笔记\in和or效率比较.png)

    红框中为非主键的比较，其余为主键比较。

37. exists关键字一般用于连表查询时，需要用in，但不需要另一个表的数据，可以把SQL改写为exists的形式，速度非常快，但可读性低。

38. 强制类型转换不会触发索引，所以在where条件时，尽量按照类型匹配去写，不要省麻烦。

```mysql
EXPLAIN SELECT * FROM ggg WHERE ggg = '1';#ref
EXPLAIN SELECT * FROM ggg WHERE ggg = 1;#ALL
```

39. 索引也有不适合建立得情况，当该数据列经常变动时，不适合建立索引，会导致B+树不断地更新维护浪费性能。当该列基数很小的时候，如性别，索引无法有效的过滤数据，因为都是一样的，该遍历那么多还是得那么多。

40. 创建索引的列，尽量不要允许null，因为索引在做判断时，如果有null，将会导致计算方式更加复杂（因为null和null是不相等的），并且可以为null的话会额外占一个字节的存储空间。

41. join操作实际上是循环匹配的过程，如A join B，A称作驱动表，B称作匹配表，循环A表，每次循环以A表的一行的数据去匹配B表的每一行数据，匹配成功则中断该次循环。由此可见，如果B表的匹配列有索引，则可以使匹配过程耗时大大减少，主键更好，连回表也省了。而且可以得出，尽量小表join大表，每次外循环都会去读取一次匹配表，如果驱动表行数过多，将会导致此IO操作过多，从而导致匹配慢。

42. 如果join on的列没有索引，MYSQL会优化算法，使用缓存来存储驱动表需要匹配的列，此时可以进行批量的匹配，减少匹配次数，缓冲区称为join-buffer，N个表join会有N-1个缓存区域，join_buffer默认256KB，有可能放不下，可以改大小，否则不会使用缓存算法而采用原算法。

    ![image-20210518164153361](E:\developer\这_MySQL调优笔记\joinbuffer大小.png)

43. 多表关联查询时，匹配的列如果类型不相同，尽量先转为相同的再联查。

    ```mysql
    EXPLAIN SELECT * FROM ggg,mytable WHERE ggg.ggg = mytable.id;
    ```

    ![image-20210518171209402](E:\developer\这_MySQL调优笔记\不同类型关联.png)

    All级的type,而此时，如果转id为char再关联。

    ```mysql
    EXPLAIN SELECT * FROM ggg,mytable WHERE ggg.ggg = CONVERT(mytable.id,CHAR)
    ```

    ![image-20210518171319076](E:\developer\这_MySQL调优笔记\相同类型关联.png)

    变为ref类型了。

44. limit会使查询速度变快，因为MYSQL查询是先查询出全部数据再筛选，limit能有效减少第一次的全部查询条数，但其实MYSQL会在没有limit的语句后面自动加上limit 0,1000以优化，所以其实这个优化点没啥暖用。

    ![image-20210518173106831](E:\developer\这_MySQL调优笔记\MYSQL自带limit.png)

    但是当起始数据很大时，limit 10000,5和limit 0,5差别是很大的，匹配前10000会消耗大量时间。

    ![image-20210518192059909](E:\developer\这_MySQL调优笔记\limit起始数据大时.png)

45. 在插入MYSQL数据时，循环insert会导致IO过多，应该先关闭自动提交，再插入后一并手动提交或批量的提交，以减少IO。二者时间差约几千到一万倍。前者3.99秒只能插入100条数据而后者插入100000条数据只要2.84秒。如果是Mybatis，可以使用

    ```java
    session = sqlSessionFactory.openSession(ExecutorType.BATCH, false);
    ```

    去get一个批量提交的一个session，然后用一个List装需要插入的众多数据，然后用

    ```java
    session.insert("(namesapce.method)", List);
    ```

    去实现批量的提交，减少IO次数。

46. select * 的又一弊端，因为MYSQL是先全部数据查询再筛选，所以*会查询到很多用不到的数据列导致速度慢，尽量只查询需要的数据列，避免全部查询。

    ![image-20210518193957676](E:\developer\这_MySQL调优笔记\select星查询全部列速度慢.png)

47. 在老的MYSQL版本，order by 的原理其实是，先把排序列的数据查询出来，对其进行排序，然后再根据排序结果去取对应的行数据，这会产生大量随机IO，不利于提升速度，此为两次传输排序。而新的版本里是单次传输排序，即先把所有要查询的数据先查询出来到缓存中，再根据排序列排序，虽然这样省去了一次IO，但是会使用大量的缓存的空间（如果列很多的话，特别是select *），相比老版的order by，新版的是以空间换了时间。当然，MYSQL也有自我保护机制，不会让非常大的数据把缓存撑爆，当要缓存的数据大小大于了max_length_for_sort_data（MYSQL全局参数）的大小时，会选择老版本的两次传输排序。max_length_for_sort_data默认大小为1024B。

48. MYSQL 的count()函数，count(*) count(id) count(123124) 都是一样的，但是如果count一个有NULL值的列，查询结果将不准确，会小于等于真实结果，NULL值不统计。而且在show profiles时，永远都是第一次查询耗时长，因为有缓存，所以如果第一次查用count(\*)然后发现耗时长，不可以得出count(\*)更耗时的结论。他们的

```mysql
show status like '%last_query_cost%'
```

​	的值是一样的。

49. 子查询由于会产生临时表，有一个IO的操作，所以效率不如直接关联查询，所以能关联查询的不要用子查。

50. MYSQL可以自定义变量，使用

    ```mysql
    SET @wocao:=1;
    ```

    来设置，然后就可以查询了。

    ```mysql
    SELECT @wocao;
    ```

    而且MYSQL自带一些变量，如@@autocommit

    ![image-20210521144431580](E:\developer\这_MySQL调优笔记\MYSQL自带变量@@autocommit.png)

    但是自定义的变量实际是存在缓存中的，一旦断开了连接就没有了，而@@的，两个@定义的是系统变量，不会消失。

    其实自定义变量在实际运用中偶尔会有奇效，如一些常量需要定义时。

    ```mysql
    SELECT @i:=@i+1 AS w,ggg.d FROM ggg,(SELECT @i:=1) t
    ```

    ![image-20210521145319839](E:\developer\这_MySQL调优笔记\自定义变量作序列号用.png)

51. 当需要limit很大时，选用直接limit不如换种方式

    ```mysql
    SELECT * FROM mytable LIMIT 100000,5; #1
    SELECT a.* FROM mytable a,(SELECT id FROM mytable LIMIT 100000,5) b WHERE a.id=b.id;#2
    ```

    使用下方的查询速度将更快。

    ![image-20210521152220669](E:\developer\这_MySQL调优笔记\limit优化结果.png)

    而且，当表的列越多时，优化效果越明显。

52. 当MYSQL要存的数据很大时，可以考虑分区，不过分区也分为纵向和横向的，如果业务繁多，导致表很多，关系也错综复杂，则需要将各个业务分别存入各个物理数据库以减少数据库压力，以及方便维护。如果数据量大，则应该同一张表拆分到各个数据库，然后查询时同时向各个数据源查询然后连起来。

53. MYSQL分区表，对于数据量很大的数据表，可以进行分区，还是同一个数据库，但是存储起来是散的，这样查询就可以并发的执行，但实际用起来都是一样的select，没什么不一样，只是创建表时需要

    ```mysql
    CREATE TABLE myFenQuTest1 (
        id INT(10),
        myDate DATE,
        age INT(10)
    )
    PARTITION BY RANGE (YEAR(myDate)) (
        PARTITION p0 VALUES LESS THAN (1030),
        PARTITION p1 VALUES LESS THAN (1060),
        PARTITION p2 VALUES LESS THAN (1090),
        PARTITION p3 VALUES LESS THAN (1120),
        PARTITION p4 VALUES LESS THAN (1150),
        PARTITION p5 VALUES LESS THAN (1180),
        PARTITION p6 VALUES LESS THAN (1210),
        PARTITION p7 VALUES LESS THAN (1240),
        PARTITION pmax VALUES LESS THAN MAXVALUE
    );
    ```

    把myDate列拿来做分区标准，每30年分一个区，方便查询。

    与非分区的表相比，

    ![image-20210524172005452](E:\developer\这_MySQL调优笔记\分区表查询结果.png)

![image-20210524172117882](E:\developer\这_MySQL调优笔记\非分区表查询结果.png)

##### 但是，在没有where mydate列查询的时候，分了区的表反而要更慢一些，而且如果分区列的分段值不合理的话或者分区表数据量很小，也会导致非分区表反而会更快一些。

当然上述只是多种分区方式的一种，range分区，还有范围分区，列分区，hash分区，key分区，子分区，虽然方式多样，但也只是分区的算法不一样，分区的本质还是没变的。

5.7以前最多1024个分区，5.7以后可以有8096个。

54. MYSQL的表锁和行锁，

    1. 表锁，开销小，加锁快，不会出现死锁，锁冲突发生概率高，并发度低。
    2. 行锁，开销大，加锁慢，会出现死锁，并发度高。

    表的写锁采用

    ```mysql
    LOCK TABLE ggg WRITE;
    ```

    读锁采用

    ```mysql
    LOCK TABLE ggg READ;
    ```

    解锁使用

    ```mysql
    UNLOCK TABLES;
    ```

    好像只能全部一起解除的样子。

    写锁一旦加上，其他的线程将无法进行读取和写入操作，直到锁被释放。而读锁加上时，其他线程可以进行读取操作，但是锁住表的线程不可以读取其他的表，因为这可能会导致死锁的出现，所以MYSQL禁止了这种操作。

    MyISAM的读和写都是串行的，但是为了变相地实现并发以及事务，MyISAM有

    ```mysql
    LOCK TABLE ggg READ LOCAL;
    ```

    操作，只锁本次会话，其他的连接还是可以对表进行操作。

    实际上，这些锁操作并不需要人为的去显示地调用，MYSQL会在查询前自动加读锁，增删改前自动加写锁。

    上述锁都是表锁。

    

    for update是行锁，也是排它锁，类似写锁。

    ```mysql
    SELECT * FROM mytable FOR UPDATE;
    ```

    lock in share mode是行锁，也是共享锁，类似读锁。

    ```mysql
    SELECT * FROM mytable WHERE id = 1 LOCK IN SHARE MODE;
    ```

    要实验行锁，需要关闭自动提交或者start transaction；

    当面对死锁时，MYSQL会自动的发现死锁，并且重启事务让锁全部解锁。

![image-20210526170839815](E:\developer\这_MySQL调优笔记\MYSQL自动解除死锁.png)
