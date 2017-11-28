---
layout: post
title: sqlalchemy简介
date: 2016-12-22
category: "python"
---

曾几何时，程序员因为惧怕SQL而在开发的时候小心翼翼的写着sql，心中总是少不了恐慌，万一不小心sql语句出错，搞坏了数据库怎么办？又或者为了获取一些数据，什么内外左右连接，函数存储过程等等。毫无疑问，不搞懂这些，怎么都觉得变扭，说不定某天就跳进了坑里，叫天天不应，喊地地不答。

ORM 的出现，让畏惧SQL的开发者，在坑里看见了爬出去的绳索，仿佛天空并不是那么黑暗，至少再暗，我们也有了眼睛。顾名思义，ORM 对象关系映射，简而言之，就是把数据库的一个个table(表)，映射为编程语言的class(类)。

python中比较著名的ORM框架有很多，大名顶顶的 SQLAlchemy 是python世界里当仁不让的ORM框架。江湖中peewee，strom， pyorm，SQLObject 各领风骚，可是最终还是SQLAlchemy 傲视群雄。

## SQLAlchemy 简介

`SQLAlchemy` 分为两个部分，一个用于 `ORM`的对象映射，另外一个是核心的 `SQL expression` 。第一个很好理解，纯粹的ORM，后面这个不是 ORM，而是`DBAPI`的封装，当然也提供了很多方法，避免了直接写sql，而是通过一些sql表达式。使用SQLAlchemy则可以分为三种方式。

* 使用 raw sql， 直接书写 sql
* 使用 sql expression ，通过 SQLAlchemy 的方法写sql表达式，简洁的写sql
* 使用 ORM 避开直接书写 sql

为什么要学习 `sql expresstion` ，而不直接上 `ORM`？因为前面两个是 orm 的基础。并且，即使不使用orm，这两个也能很好的完成工作，并且代码的可读性更好。纯粹把`SQLAlchemy`当成dbapi使用。首先`SQLAlchemy` 内建数据库连接池，解决了连接操作相关繁琐的处理。其次，提供方便的强大的log功能，最后，复杂的查询语句，依靠单纯的`ORM`比较难实现。

## 使用raw sql方式

首先需要导入 sqlalchemy 库，然后建立数据库连接，这里使用 `mysql`。通过`create_engine`方法进行

```python
from sqlalchemy import create_engine

engine = create_engine("mysql://root:@localhost:3306/webpy?charset=utf8",encoding="utf-8", echo=True)
```

`create_engine` 方法进行数据库连接，返回一个 db 对象。里面的参数表示

> 数据库类型://用户名:密码（没有密码则为空，不填）@数据库主机地址/数据库名?编码
>
> echo = True 是为了方便 控制台 logging 输出一些sql信息，默认是False

通过这个engine对象可以直接execute 进行查询，例如 `engine.execute("SELECT * FROM user")` 。

也可以通过 engine 获取连接在查询，例如 `conn = engine.connect()` 通过 `conn.execute()`方法进行查询。

两者有什么差别呢？

- 直接使用engine的execute执行sql的方式, 叫做connnectionless执行,
- 借助 engine.connect()获取conn, 然后通过conn执行sql, 叫做connection执行。
- 主要差别在于是否使用transaction模式, 如果不涉及transaction, 两种方法效果是一样的. 官网推荐使用后者。

然后通过直接书写sql语句的方式来访问数据库。

```python
#插入
engine.execute(
    "INSERT INTO ts_test (a, b) VALUES ('2', 'v1')"
)
 
engine.execute(
     "INSERT INTO ts_test (a, b) VALUES (%s, %s)",
    ((555, "v1"),(666, "v1"),)
)
engine.execute(
    "INSERT INTO ts_test (a, b) VALUES (%(id)s, %(name)s)",
    id=999, name="v1"
)
 
#查询
result = engine.execute('select * from ts_test')
result.fetchall()
```

## 使用sql expresstion方式

### 定义表

定义数据表，才能进行sql表达式的操作。如果数据库已经存在了数据表还需要定义么？这里其实是一个映射关系，如果不指定，查询表达式就不知道是附加在哪个表的操作。

定义的时候，注意表名和字段名，代码和数据的必须保持一致。

定义好之后,就能创建数据表，一旦创建了，再次运行创建的代码，数据库是不会创建的。

```python
#-- coding: utf-8 --
from sqlalchemy import create_engine, Table, Column, Integer, String, MetaData, ForeignKey

# 连接数据库 
engine = create_engine("mysql://root:@localhost:3306/webpy?charset=utf8",encoding="utf-8", echo=True)

# 获取元数据
metadata = MetaData()

# 定义表
user = Table('user', metadata,
        Column('id', Integer, primary_key=True),
        Column('name', String(20)),
        Column('fullname', String(40)),
    )

address = Table('address', metadata,
        Column('id', Integer, primary_key=True),
        Column('user_id', None, ForeignKey('user.id')),
        Column('email', String(60), nullable=False)
    )

# 创建数据表，如果数据表存在，则忽视
metadata.create_all(engine)

# 获取数据库连接
conn = engine.connect()
```

### 插入insert

有了数据表和连接对象，对应数据库操作就简单了。

```python
>>> i = user.insert()   # 使用插入
>>> i  
<sqlalchemy.sql.dml.Insert object at 0x0000000002637748>
>>> print i  # 内部构件的sql语句
INSERT INTO "user" (id, name, fullname) VALUES (:id, :name, :fullname)
>>> u = dict(name='jack', fullname='jack Jone')
>>> r = conn.execute(i, **u)  # 执行查询，第一个为查询对象，第二个参数为一个插入数据字典，如果插入的是多个对象，就把对象字典放在列表里面
>>> r
<sqlalchemy.engine.result.ResultProxy object at 0x0000000002EF9390>
>>> r.inserted_primary_key  # 返回插入行 主键 id
[4L]
>>> addresses
[{'user_id': 1, 'email': 'jack@yahoo.com'}, {'user_id': 1, 'email': 'jack@msn.com'}, {'user_id': 2, 'email': 'www@www.org'}, {'user_id': 2, 'email': 'wendy@aol.com'}]
>>> i = address.insert()
>>> r = conn.execute(i, addresses)   # 插入多条记录
>>> r
<sqlalchemy.engine.result.ResultProxy object at 0x0000000002EB5080>
>>> r.rowcount   #返回影响的行数
4L

>>> i = user.insert().values(name='tom', fullname='tom Jim')
>>> i.compile()
<sqlalchemy.sql.compiler.SQLCompiler object at 0x0000000002F6F390>
>>> print i.compile()
INSERT INTO "user" (name, fullname) VALUES (:name, :fullname)
>>> print i.compile().params
{'fullname': 'tom Jim', 'name': 'tom'}
>>> r = conn.execute(i)
>>> r.rowcount
1L
```

### 查询select

查询方式很灵活，多数时候使用 sqlalchemy.sql 下面的 `select`方法。

```python
>>> s = select([user])  # 查询 user表
>>> s
<sqlalchemy.sql.selectable.Select at 0x25a7748; Select object>
>>> print s
SELECT "user".id, "user".name, "user".fullname 
FROM "user"
```

如果需要查询自定义的字段，可是使用 user 的`cloumn` 对象，例如

```python
>>> user.c  # 表 user 的字段column对象
<sqlalchemy.sql.base.ImmutableColumnCollection object at 0x0000000002E804A8>
>>> print user.c
['user.id', 'user.name', 'user.fullname']
>>> s = select([user.c.name,user.c.fullname])
>>> r = conn.execute(s)
>>> r
<sqlalchemy.engine.result.ResultProxy object at 0x00000000025A7748>
>>> r.rowcount  # 影响的行数
5L
>>> ru = r.fetchall()  
>>> ru
[(u'hello', u'hello world'), (u'Jack', u'Jack Jone'), (u'Jack', u'Jack Jone'), (u'jack', u'jack Jone'), (u'tom', u'tom Jim')]
>>> r  
<sqlalchemy.engine.result.ResultProxy object at 0x00000000025A7748>
>>> r.closed  # 只要 r.fetchall() 之后，就会自动关闭 ResultProxy 对象
True
```

同时查询两个表

```python
>>> s = select([user.c.name, address.c.user_id]).where(user.c.id==address.c.user_id)   # 使用了字段和字段比较的条件
>>> s
<sqlalchemy.sql.selectable.Select at 0x2f03390; Select object>
>>> print s
SELECT "user".name, address.user_id 
FROM "user", address 
WHERE "user".id = address.user_id
```

### 更新 update

前面都是一些查询，更新和插入的方法很像，都是 表下面的方法，不同的是，`update` 多了一个 `where` 方法 用来选择过滤。

```python
>>> s = user.update()
>>> print s
UPDATE "user" SET id=:id, name=:name, fullname=:fullname
>>> s = user.update().values(fullname=user.c.name)           # values 指定了更新的字段
>>> print s
UPDATE "user" SET fullname="user".name
>>> s = user.update().where(user.c.name == 'jack').values(name='ed')  # where 进行选择过滤
>>> print s 
UPDATE "user" SET name=:name WHERE "user".name = :name_1
>>> r = conn.execute(s)
>>> print r.rowcount         # 影响行数
3
```

### 删除 delete

删除比较容易，调用 `delete`方法即可，不加 `where` 过滤，则删除所有数据，但是不会drop掉表，等于清空了数据表。

```python
>>> r = conn.execute(address.delete()) # 清空表
>>> print r
<sqlalchemy.engine.result.ResultProxy object at 0x0000000002EAF550>
>>> r.rowcount
8L
>>> r = conn.execute(users.delete().where(users.c.name > 'm')) # 删除记录
>>> r.rowcount
3L
```

## 使用ORM方式

### 定义表

```python
sqlalchemy_declarative.py

import os
import sys
from sqlalchemy import Column, ForeignKey, Integer, String
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import relationship
from sqlalchemy import create_engine
 
Base = declarative_base()
 
class Person(Base):
    __tablename__ = 'person'
    # Here we define columns for the table person
    # Notice that each column is also a normal Python instance attribute.
    id = Column(Integer, primary_key=True)
    name = Column(String(250), nullable=False)
 
class Address(Base):
    __tablename__ = 'address'
    # Here we define columns for the table address.
    # Notice that each column is also a normal Python instance attribute.
    id = Column(Integer, primary_key=True)
    street_name = Column(String(250))
    street_number = Column(String(250))
    post_code = Column(String(250), nullable=False)
    person_id = Column(Integer, ForeignKey('person.id'))
    person = relationship(Person)
 
# Create an engine that stores data in the local directory's
# sqlalchemy_example.db file.
engine = create_engine('sqlite:///sqlalchemy_example.db')
 
# Create all tables in the engine. This is equivalent to "Create Table"
# statements in raw SQL.
Base.metadata.create_all(engine)
```

上述代码在数据库中创建了`Person`表和`Address`表。基本步骤如下：

1. 调用`declarative_base()`创建一个`Base`类
2. 从`Base`类派生`Table`类，对应数据库中的表
3. 创建一个链接数据库的`engine`
4. 调用`Base.metadata.create_all`连接数据库，并创建table

### 插入insert

```python
sqlalchemy_insert.py

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
 
from sqlalchemy_declarative import Address, Base, Person
 
engine = create_engine('sqlite:///sqlalchemy_example.db')
# Bind the engine to the metadata of the Base class so that the
# declaratives can be accessed through a DBSession instance
Base.metadata.bind = engine
 
DBSession = sessionmaker(bind=engine)
# A DBSession() instance establishes all conversations with the database
# and represents a "staging zone" for all the objects loaded into the
# database session object. Any change made against the objects in the
# session won't be persisted into the database until you call
# session.commit(). If you're not happy about the changes, you can
# revert all of them back to the last commit by calling
# session.rollback()
session = DBSession()
 
# Insert a Person in the person table
new_person = Person(name='new person')
session.add(new_person)
session.commit()
 
# Insert an Address in the address table
new_address = Address(post_code='00000', person=new_person)
session.add(new_address)
session.commit()
```

步骤如下：

1. 创建一个链接数据库的`engine`
2. 将`engine`绑定到`Base`类
3. 调用`sessionmaker`创建一个`session`
4. 实例化`table`对象，通过`session.add`新增数据
5. 通过`session.commit`执行修改数据库的操作

### 查询select

```python
>>> from sqlalchemy_declarative import Person, Base, Address
>>> from sqlalchemy import create_engine
>>>
>>> engine = create_engine('sqlite:///sqlalchemy_example.db')
>>> Base.metadata.bind = engine
>>>
>>> from sqlalchemy.orm import sessionmaker
>>> DBSession = sessionmaker()
>>> DBSession.bind = engine
>>> session = DBSession()
>>> # Make a query to find all Persons in the database
>>> session.query(Person).all()
[<sqlalchemy_declarative.Person object at 0x2ee3a10>]
>>>
>>> # Return the first Person from all Persons in the database
>>> person = session.query(Person).first()
>>> person.name
u'new person'
>>>
>>> # Find all Address whose person field is pointing to the person object
>>> session.query(Address).filter(Address.person == person).all()
[<sqlalchemy_declarative.Address object at 0x2ee3cd0>]
>>>
>>> # Retrieve one Address whose person field is point to the person object
>>> session.query(Address).filter(Address.person == person).one()
<sqlalchemy_declarative.Address object at 0x2ee3cd0>
>>> address = session.query(Address).filter(Address.person == person).one()
>>> address.post_code
u'00000'
```

步骤如下：

1. 创建一个链接数据库的`engine`
2. 将`engine`绑定到`Base`类
3. 调用`sessionmaker`创建一个`session`
4. 通过`session.query`查询数据库

