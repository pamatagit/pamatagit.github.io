# Openstack数据库访问

## OpenStack中的关系型数据库应用

OpenStack中的数据库应用主要是关系型数据库，主要使用的是MySQL数据库。当然也有一些NoSQL的应用，比如Ceilometer项目。就SQL数据库本身的应用而言，OpenStack的项目和其他项目并没有什么区别，也是采用ORM技术对数据进行增删改查而已。

本文的重点是讲解OpenStack项目中对关系型数据库的应用的基础知识，更多的是涉及ORM库的使用。对于数据库的安装和配置，需要读者自己查找一下MySQL的教程，如果只是为了验证ORM的相关知识，也可以使用sqlite数据库。

### 数据库的选择

OpenStack官方推荐的保存生产数据的是MySQL数据库，在devstack项目（这个项目用于快速搭建OpenStack开发环境）中也是安装了MySQL数据库。不过，因为OpenStack的项目中没有使用特定的只有在MySQL上才能用的功能，而且所采用的ORM库SQLAlchemy也支持多种数据库，所以理论上选择PostgreSQL之类的数据库来替代MySQL也是可行的。

另外，OpenStack项目在单元测试中使用的是sqlite的内存数据库，这样开发者运行单元测试的时候不需要安装和配置复杂的MySQL数据库，只要安装好sqlite3就可以了。而且，数据库是保存在内存中的，会提高单元测试的速度。

### ORM的选择

#### 什么是ORM

ORM的全称是**Object-Relational Mapping**，即对象关系映射，是一种利用编程语言的对象来表示关系数据库中的数据的技术。简单的说，ORM就是把数据库的一张表和编程语言中的一个类对象对应起来，这样我们在编程语言中操作一个对象的时候，实际上就是在操作这张表，ORM（一般是一个库）负责把我们对一个对象的操作转换成对数据库的操作。

#### Python中的ORM实现

一般来说，各种主流语言都有自己的ORM实现，一般来说也不只一种，比较出名的有Java的Hibernate，Ruby on Rails的ORM，C++的ODB等。**在Python中也存在多种ORM的实现**，最著名的两种是**Django的Model层的ORM实现**，以及**SQLAlchemy库**。这两种ORM实现基本上是Python中ORM的事实上的标准，如果你写Django应用，那么你就用Django自带的实现；不然，你就可以选择SQLAlchemy库。

OpenStack基本上都是Python项目，所以在OpenStack中，ORM主要是使用了SQLAlchemy库（Keystone、Nova、Neutron等）；不过使用了Django的Horizon项目（面板）还是使用了Django自带的ORM实现。本文主要是讲解OpenStack中如何使用SQLAlchemy库，这个也是开发OpenStack项目的最基本知识。

## SQLAlchemy

### SQLAlchemy简介

SQLAlchemy项目是Python中最著名的ORM实现，不仅在Python项目中得到了广泛的应用，而且对其他语言的ORM有很大的影响。OpenStack一开始选择这个库，也是看中了它足够稳定、足够强大的特点。

### SQLAlchemy的架构

先让我们来看一下SQLAlchemy这个库的总体架构，如下图（图来自官网）所示：

![img](http://cdn3.infoqstatic.com/statics_s1_20170510-0410/resource/articles/learning-openstack-through-demo-part01/zh/resources/3662229628-56890bcfe5df8.png)

SQLAlchemy这个库分为两层：

- 上面这层是ORM层，为用户提供ORM接口，c即通过操作Python对象来实现数据库操作的接口。

- 下面这层是Core层，这层包含了Schema/Types、SQL Expression Language、Engine这三个部分：

  - SQL Expression Language是SQLAlchemy中实现的一套SQL表达系统，主要是实现了对SQL的DML(Data Manipulation Language)的封装。这里实现了对数据库的SELECT、DELETE、UPDATE等语句的封装。SQL Expression Language是实现ORM层的基础。
  - Schema/Types这部分主要是实现了对SQL的DDL(Data Definition Language)的封装。实现了Table类用来表示一个表，Column类用来表示一个列，也是实现了将数据库的数据类型映射到Python的数据类型。上面的SQL Expression Language的操作对象就是这里定义的Table。
  - Engine实现了对各种不同的数据库客户端的封装和调度，是所有SQLAlchemy应用程序的入口点，要使用SQLAlchemy库来操作一个数据库，首先就要有一个Engine对象，后续的所有对数据库的操作都要通过这个Engine对象来进行。下图是官方文档中的Engine位置的描述图：

  ![img](http://cdn3.infoqstatic.com/statics_s1_20170510-0410/resource/articles/learning-openstack-through-demo-part01/zh/resources/3162376620-56891835d3c04.png)

  1. Pool是Engine下面的一个模块，用来管理应用程序到数据库的连接。
  2. Dialect是Engine下的另一个模块，用来对接不同的数据库驱动（即DBMS客户端），这些驱动要实现DBAPI接口。


- 最后，SQLAlchemy还要依赖各个数据库驱动的DBAPI接口来实现对数据库服务的调用。DBAPI是Python定义的数据库API的实现规范，具体见[PEP0249](https://www.python.org/dev/peps/pep-0249/)。

上面简单的总结了SQLAlchemy的架构，希望大家能够大概了解一下SQLAlchemy，在后面介绍一些相关概念时，能够知道这个概念是属于整个架构的哪个部分。

在openstack中，使用的是ORM层

#### Dialect和数据库客户端

上面提到了Dialect是用来对接不同的数据库驱动的，它主要负责将SQLAlchemy最后生成的数据库操作转换成对数据库驱动的调用，其中会处理一些不同数据库和不同DBAPI实现的差别。这个部分一般是SQLAlchemy的开发者关心的内容，如果你只是使用SQLAlchemy来操作数据库，那么可以不用关心这个部分。不过我们还是要来了解一下SQLAlchemy支持的和OpenStack相关的数据库驱动。

**MySQL**

OpenStack项目主要是使用MySQL，之前一直都在使用[**MySQL-Python**](https://pypi.python.org/pypi/MySQL-python/)驱动，因为这个驱动足够成熟和稳定。不过这个情况正在转变，有如下两个原因：

1. MySQL-Python不支持Python3，而OpenStack正在转换到Python3的过程中，所以这个驱动最终是要放弃的。
2. MySQL-Python是用C语言写的，不支持eventlet库的monkey-patch操作，无法被eventlet库转换成异步操作，所以使用了eventlet库的到OpenStack项目在使用MySQL数据库时，都是进行同步的串行操作，有性能损失。

为了解决这个问题，社区发起了一次对新驱动的评估，主要是评估[MySQL-Python](https://pypi.python.org/pypi/MySQL-python/)驱动：[**PyMySQL Evaluation**](https://wiki.openstack.org/wiki/PyMySQL_evaluation)。这个评估还在社区的邮件列表发起了好几次讨论，到目前为止的结果是：**如果使用Python 2.7，那么继续使用MySQL-Python这个驱动**，否则就使用PyMySQL这个驱动。PyMySQL驱动是使用纯Python写的，不仅支持Python3而且可以支持eventlet的异步。

**SQLite3**

OpenStack项目一般会使用SQLite3数据库来运行单元测试。OpenStack在Python2.7下会使用[pysqlite](https://pypi.python.org/pypi/pysqlite/)驱动，不过这个驱动和标准库中的sqlite3模块是一样的，也就是Python内置了SQLite3的驱动，你无需选择其他的驱动。

### SQLAlchemy的基本概念和使用

使用SQLAlchemy大体上分为三个步骤：连接到数据库，定义数据模型，执行数据操作。

#### 连接到数据库

在你的应用可以使用数据库前，你要先定义好数据库的连接，包括数据库在哪里，用什么账号访问等。所有的这些工作都是通过Engine对象来进行的（记得上面提到的Engine了么？）。

**数据库URL**

SQLAlchemy使用URL的方式来指定要访问的数据库，整个URL的具体格式如下：

```
dialect+driver://username:password@host:port/database
```

其中，**dialect**就是指DBMS的名称，一般可选的值有：*postgresql、 mysql、sqlite*等。**driver**就是指驱动的名称，如果不指定，SQLAlchemy会使用默认值。*database*就是指DBMS中的一个数据库，一般是指通过`CREATE DATABASE`语句创建的数据库。其他的参数就不言而喻了。dialect和driver参数有很多选择，具体的可以参考官方文档：[Database URLs](http://docs.sqlalchemy.org/en/rel_1_0/core/engines.html#database-urls)。

**创建Engine对象**

确定了要连接的数据库信息后，就可以通过`create_engine`函数来创建一个Engine对象了。

```
from sqlalchemy import create_engine

engine = create_engine('sqlite://:memory:')
```

`create_engine`函数还支持以下几个参数：

- connect_args：一个字典，用来自定义数据库连接的参数，比如指定客户端使用的字符编码。
- pool_size和max_overflow：指定连接池的大小。
- poolclass：指定连接池的实现
- echo：一个布尔值，用来指定是否打印执行的SQL语句到日志中。

还有很多其他的参数，可以参考官方文档：[Engine Configuration](http://docs.sqlalchemy.org/en/rel_1_0/core/engines.html)。

一般来说，Engine对象会默认启用连接池，会根据不同的dialect来选择不同的默认值。一般来说，你是不用考虑连接池的配置的，默认情况都配置好了。想了解关于连接池的更多内容，请查看官方文档：[Connection Pooling](http://docs.sqlalchemy.org/en/rel_1_0/core/pooling.html)。

**使用Engine对象**

一般来说，应用程序的代码是不直接使用Engine对象的，而是把Engine对象交给ORM去使用，或者创建session对象来使用。不过，我们还是来简单看一下Engine对象能做什么事情。

应用程序可以调用Engine对象的`connect()`方法来获得一个到数据库的连接对象；然后可以在这个连接对象上调用`execute()`来执行SQL语句，调用`begin()`、`commit()`、`rollback()`来执行事务操作；调用`close()`来关闭连接。Engine对象也有一些快捷方法来直接执行上述操作，避免了每次都要调用`connect()`来获取连接这种繁琐的代码，比如`engine.execute()`、`with engine.begin()`等。

#### 定义数据模型

有了数据库连接后，我们就可以来定义数据模型了，也就是定义映射数据库表的Python类。在SQLAlchemy中，这是通过**Declarative**的系统来完成的。

**Declarative系统**

根据官方文档的描述，SQLAlchemy一开始是采用下面这种方式来定义ORM的：

1. 首先定义一个映射类，这个类是数据库表在代码中的对象表示，这类的类属性是很多Column类的实例。
2. 然后定义一个Table对象，这里的Table就是上面提到的在**Schema/Types**模块中的一个类，用来表示一个数据库中的表。
3. 调用`sqlalchemy.orm.mapper`函数把步骤1中定义的类映射到步骤2中定义的Table。

上面这种方式称为*Classical Mappings*，看起来好麻烦啊。所以就有了**Declarative**系统。这个系统就是一次完成这三个步骤，你只需要定义步骤1中的类即可。这也是现在在SQLAlchemy中使用ORM的方式，无需在使用过去这种麻烦的方法。

要使用Declarative系统，你需要为所有映射类创建一个基类，这个基类用来维护所有映射类的元信息。

```
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()
```

**定义映射类**

现在我们可以开始创建映射类了。假设我们在数据库中有一个表Person，这个表有两个列，分别是id和name，那么我们创建的映射类如下：

```
from sqlalchemy import Column, Integer, String

# 这里的基类Base是上面我们通过declarative_base函数生成的
class Person(Base):
    __tablename__ = 'person'

    id = Column(Interger, primary_key=True)
    name = Column(String(250), nullable=False)
```

这样我们就定义了一个映射类**Person**，后续我们可以通过操作这个类的实例来实现对数据库表person的操作。在我们的映射类中，我们使用`__tablename__`属性来指定该映射类所对应的数据库表，通过`Column`类实例的方式来指定数据库的字段。这里，读者可能会问：我如何能知道`Column`都能支持哪些类型呢？这个查看官方文档获得：[Column And Data Types](http://docs.sqlalchemy.org/en/rel_1_0/core/type_basics.html)。

因为我们使用了Declarative系统，所以虽然我们自己没有定义Table对象，但是Declarative系统帮我们做了，并且帮我们调用了`mapper`函数。因此，当我们定义好一个表的映射类后，这个类的`__table__`属性就保存了该映射类所映射的*Table*对象：

```
In [6]: Person.__table__
Out[6]: Table('person', MetaData(bind=None),
    Column('id', Integer(), table=<person>, primary_key=True, nullable=False),
    Column('name', String(length=250), table=<person>, nullable=False), schema=None)
```

定义映射类是我们使用ORM的最主要的功能之一，不仅可以指定单表的映射，还能够指定表之间的关系。由于篇幅限制，我们在本文就不展开讲了。

**Schema和Metadata**

关于Table对象，我们上面也提到了，它属于SQLAlchemy的core层的**Schema/Types**这个部分。SQLAlchemy中的Schema可以理解为和DDL相关的一套体系，它告诉SQLAlchemy的其他部分，数据库中的表是如何定义的。这个相当于我们在MySQL中使用`describe`命令，或者在PostgreSQL中使用`\d`命令。

SQLAlchemy中通过**schema metadata**来实现上面说的Schema。Schema metadata，官方文档中也称为database metadata，简称为metadata，是一个容器，其中包含了和DDL相关的所有信息，包括Table、Column等对象。当SQLAlchemy要根据映射类生成SQL语句时，它会查询metadata中的信息，根据信息来生成SQL语句。

为了要让metadata可以工作，我们需要把DDL的相关信息放到metadata中。如果你注意看上面`Person.__table__`的输出，就会发现`Table`类的第二个参数就是一个Metadata实例，也就是说，我们需要在定义Table的时候就把DDL信息放到metadata中。如果是是用classical mapping的方式，我们需要先创建一个metadata实例，然后每次创建一个Table对象的时候就把metadata传递进去。从写代码的角度来说，这个方式没有什么问题，也不算麻烦；问题是我们在使用ORM的过程中，几乎不会用到metadata，metadata基本上是给SQLAlchemy用的，对于用户来说metadata提供的接口只能用来创建表和删除表，这种操作的频率远低于查询操作。

好在Declarative系统则帮我们把这些都做好了。当我们通过`declarative_base()`生成一个基类Base的时候，这个基类就已经包含了一个metadata实例，后面基于Base定义的映射类都会被自动加入到这个metadata中。我们可以通过`Base.metadata`来访问这个metadata实例。

说了这么多关于metadata的内容，简单总结一下：metadata是schema在SQLAlchemy中的实现，包含了DDL的信息，SQLAlchemy中的其他部分需要依赖于metadata中的信息，一般用户很少使用metadata。

很少用？那说这么多是做啥？主要是让读者可以理解下面这个语句的原理：

```python
Base = declarative_base()

# 基于Base定义映射类
Base.metadata.create_all(engine)
```

最后这行代码是我们最常用到metadata的地方：创建所有的表。我们告诉`create_all`使用哪个engine，它就会生成所有的`CREATE TABLE`语句，并且通过engine发送到数据库上执行。

#### Session

会话(**session**)是我们通过SQLAlchemy来操作数据库的入口。我们前面有介绍过SQLAlchemy的架构，session是属于ORM层的。Session的功能是管理我们的程序和数据库之间的会话，它利用Engine的连接管理功能来实现会话。我们在上文有提到，我们创建了Engine对象，但是一般不直接使用它，而是把它交给ORM去使用。其中，通过session来使用Engine就是一个常用的方式。

通过session来使用Engine的一个好处是session可以处理事务（transaction），只有执行session.commit使，对数据库做的修改或查询才会提交到数据库中。

要是用session，我们需要先通过`sessionmaker`函数创建一个session类，然后通过这个类的实例来使用会话，如下所示：

```
from sqlalchemy.orm import sessionmaker

DBSession = sessionmaker(bind=engine)
session = DBSession()
```

我们通过`sessionmaker`的*bind*参数把Engine对象传递给`DBSession`去管理。然后，`DBSession`实例化的对象`session`就能被我们使用了。

**CRUD**

**CRUD**就是CREATE、READ、UPDATE、DELETE，增删改查。这个也是SQLAlchemy中最常用的功能，而且都是通过上一小节中的`session`对象来使用的。我们这简单的介绍一下这四个操作，后面会给出官方文档的位置。

**Create**

在数据库中插入一条记录，是通过session的`add()`方法来实现的，你需要先创建一个映射类的实例，然后调用`session.add()`方法，然后调用`session.commit()`方法提交你的事务（关于事务，我们下面会专门讲解）：

```
new_person = Person(name='new person')
session.add(new_person)
session.commit()
```

**Delete**

删除操作和创建操作差不多，是把一个映射类实例传递给`session.delete()`方法。

**Update**

更新一条记录需要先使用查询操作获得一条记录对应的对象，然后修改对象的属性，再通过`session.add()`方法来完成更新操作。

**Read**

查询操作，一般称为query，在SQLAlchemy中一般是通过**Query对象**来完成的。我们可以通过session.query()方法来创建一个Query对象，然后调用Query对象的众多方法来完成查询操作。

#### 事务

使用session，就会涉及到事务，我们的应用程序也会有很多事务操作的要求。当你调用一个session的方法，导致session执行一条SQL语句时，它会自动开始一个事务，直到你下次调用`session.commit()`或者`session.rollback()`，它就会结束这个事务。你也可以显示的调用`session.begin()`来开始一个事务，并且`session.begin()`还可以配合Python的with来使用。

## 使用SQLAlchemy

openstack使用SQLAlchemy分为2个阶段，最开始各个项目都是在代码中直接调用SQLAlchemy模块，后来由于各个项目都需要创建engine、session等，就创建了olso.db项目，封装这些方法，供所有openstack项目使用。

### 直接使用SQLAlchemy

最简单的模型下，数据库相关代码一般存放在2个文件中：

```
1. 在**db/models.py**中定义`User`类，对应数据库的user表。
2. 在**db/api.py**中实现一个`Connection`类，这个类封装了所有的数据库操作接口。我们会在这个类中实现对user表的CRUD等操作。

```

#### 定义User数据模型映射类

**db/models.py**中的代码如下：

```python
from sqlalchemy import Column, Integer, String
from sqlalchemy.ext import declarative
from sqlalchemy import Index

Base = declarative.declarative_base()

class User(Base):
    """User table"""

    __tablename__ = 'user'
    __table_args__ = (
        Index('ix_user_user_id', 'user_id'),
    )
    id = Column(Integer, primary_key=True)
    user_id = Column(String(255), nullable=False)
    name = Column(String(64), nullable=False, unique=True)
    email = Column(String(255))
```

这样定义了一个User类，映射到了数据库中的user表。

#### 实现DB API

**DB通用函数**

在**db/api.py**中，我们先定义了一些通用函数，代码如下：

```
from sqlalchemy import create_engine
import sqlalchemy.orm
from sqlalchemy.orm import exc

from webdemo.db import models as db_models

_ENGINE = None
_SESSION_MAKER = None

def get_engine():
    global _ENGINE
    if _ENGINE is not None:
        return _ENGINE

    _ENGINE = create_engine('sqlite://')
    db_models.Base.metadata.create_all(_ENGINE)
    return _ENGINE

def get_session_maker(engine):
    global _SESSION_MAKER
    if _SESSION_MAKER is not None:
        return _SESSION_MAKER

    _SESSION_MAKER = sqlalchemy.orm.sessionmaker(bind=engine)
    return _SESSION_MAKER

def get_session():
    engine = get_engine()
    maker = get_session_maker(engine)
    session = maker()

    return session
```

上面的代码中，我们定义了三个函数：

- `get_engine`：返回全局唯一的engine，不需要重复分配。
- `get_session_maker`：返回全局唯一的session maker，不需要重复分配。
- `get_session`：每次返回一个新的session，因为一个session不能同时被两个数据库客户端使用。

这三个函数是使用SQL Alchemy中经常会封装的，所以OpenStack的**oslo_db**项目就封装了这些函数，供所有的OpenStack项目使用。

这里需要注意一个地方，在`get_engine()`中：

```
    _ENGINE = create_engine('sqlite://')
    db_models.Base.metadata.create_all(_ENGINE)
```

我们使用了sqlite内存数据库，并且立刻创建了所有的表。这么做只是为了演示方便。在实际的项目中，`create_engine()`的数据库URL参数应该是从配置文件中读取的，而且也不能在创建engine后就创建所有的表（这样数据库的数据都丢了）。要解决在数据库中建表的问题，就要先了解数据库版本管理的知识，也就是**database migration**，我们在下文中会说明。

**访问数据库**

`Connection`的实现就简单得多了，直接看代码。这里只实现了`get_user()`和`list_users()`方法。

```python
class Connection(object):

    def __init__(self):
        pass

    def get_user(self, user_id):
        # 调用session.query创建query对象
        query = get_session().query(db_models.User).filter_by(user_id=user_id)
        try:
            user = query.one()
        except exc.NoResultFound:
            # TODO(developer): process this situation
            pass

        return user

    def list_users(self):
        session = get_session()
        query = session.query(db_models.User)
        users = query.all()

        return users

    def update_user(self, user):
        pass

    def delete_user(self, user):
        pass
```

