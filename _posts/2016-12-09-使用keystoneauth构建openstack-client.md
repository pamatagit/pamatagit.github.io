---
layout: post
title:使用keystoneauth构建openstack client
date: 2016-12-09
category: "keystone"
---

keystoneauth是从keystoneclient中分离出来的模块，主要包含auth plugin和session两部分。

## 使用session

session封装的是python的requests模块，负责发送请求。

```python
>>> from keystoneauth1.identity import v3
>>> from keystoneauth1 import session
>>> from keystoneclient.v3 import client

>>> auth = v3.Password(auth_url='https://my.keystone.com:5000/v3',
...                    username='myuser',
...                    password='mypassword',
...                    project_name='proj',
...                    user_domain_id='default',
...                    project_domain_id='default')
>>> sess = session.Session(auth=auth)
>>> ks = client.Client(session=sess)
>>> users = ks.users.list()
```

这是通过session和auth构建了一个keystoneclient。另外，也可以将auth plugin传入keystoneclient，而不是传给session。

```python
>>> from keystoneauth1.identity import v3
>>> from keystoneauth1 import session
>>> from keystoneclient.v3 import client

>>> auth = v3.Password(auth_url='https://my.keystone.com:5000/v3',
...                    username='myuser',
...                    password='mypassword',
...                    project_name='proj',
...                    user_domain_id='default',
...                    project_domain_id='default')
>>> sess = session.Session()
>>> ks = client.Client(session=sess, auth=auth)
>>> users = ks.users.list()
```

也可以直接用session的get方法发送请求。

```python
>>> sess = session.Session(auth=auth)
>>> resp = sess.get('https://my.keystone.com:5000/v3/users')
```

或：

```python
>>> sess = session.Session()
>>> sess.get('https://my.keystone.com:5000/v3/uses',auth=auth1)
>>> sess.get('https://my.keystone.com:5000/v3/projects',auth=auth2)
```

### service discovery

keystoneclient不需要知道完整的URL，只需要知道相对的URL就可以了，详细的URL信息可以从keystone的endpoint中获取。

session中提供了endpoint_filter参数来查找相应的URL，endpoint_filter有3个选项。

service_type表示服务类型，keystone服务是identity，nova服务是compute，cinder服务是volume，这个跟keystone中的endpoint信息是对应的。

interface表示哪种类型的endpoint，一般有internal、public、admin3种。

region_name表示region。

```python
>>> resp = session.get('/users',
                       endpoint_filter={'service_type': 'identity',
                                        'interface': 'admin',
                                        'region_name': 'myregion'})
```

## 通过config文件load配置项创建client

```python
from cinderclient import client as cinder_client
from keystoneauth1 import loading as ks_loading
from oslo_config import cfg

CONF = cfg.CONF
CONF(args=sys.argv[1:])
CINDER_OPT_GROUP = 'cinder'

ks_loading.register_session_conf_options(CONF,CINDER_OPT_GROUP)
ks_loading.register_auth_conf_options(CONF, CINDER_OPT_GROUP)

sess = ks_loading.load_session_from_conf_options(CONF,CINDER_OPT_GROUP)
auth_plugin = ks_loading.load_auth_from_conf_options(CONF,CINDER_OPT_GROUP)

cinder_client.Client(session=_SESSION, auth=auth_plugin)
```

可以看到基本步骤如下：

* 注册session配置项
* 注册auth配置项
* 加载session配置项生成session对象
* 加载auth配置项生成auth_plugin
* 利用session和auth_plugin生成client

### 注册session配置项

```python
keystoneauth1.loading.session.py

def register_conf_options(self, conf, group, deprecated_opts=None):
    opts = self.get_conf_options(deprecated_opts=deprecated_opts)
    conf.register_group(cfg.OptGroup(group))
    conf.register_opts(opts, group=group)
    return opts

def get_conf_options(self, deprecated_opts=None):
        if not cfg:
            raise ImportError("oslo.config is not an automatic dependency of "
                              "keystoneauth. If you wish to use oslo.config "
                              "you need to import it into your application's "
                              "requirements file. ")

        if deprecated_opts is None:
            deprecated_opts = {}

        return [cfg.StrOpt('cafile',
                           deprecated_opts=deprecated_opts.get('cafile'),
                           help='PEM encoded Certificate Authority to use '
                                'when verifying HTTPs connections.'),
                cfg.StrOpt('certfile',
                           deprecated_opts=deprecated_opts.get('certfile'),
                           help='PEM encoded client certificate cert file'),
                cfg.StrOpt('keyfile',
                           deprecated_opts=deprecated_opts.get('keyfile'),
                           help='PEM encoded client certificate key file'),
                cfg.BoolOpt('insecure',
                            default=False,
                            deprecated_opts=deprecated_opts.get('insecure'),
                            help='Verify HTTPS connections.'),
                cfg.IntOpt('timeout',
                           deprecated_opts=deprecated_opts.get('timeout'),
                           help='Timeout value for http requests'),
                ]
```

### 注册auth配置项

```python
keystoneauth1.loading.conf.py

_AUTH_SECTION_OPT = opts.Opt('auth_section', help=_section_help)
_AUTH_TYPE_OPT = opts.Opt('auth_type',
                          deprecated=[opts.Opt('auth_plugin')],
                          help='Authentication type to load')

def register_conf_options(conf, group):
    conf.register_opt(_AUTH_SECTION_OPT._to_oslo_opt(), group=group)
    if conf[group].auth_section:
        group = conf[group].auth_section

    conf.register_opt(_AUTH_TYPE_OPT._to_oslo_opt(), group=group)
```

这里先注册了auth_section和auth_type 2个配置项。

### 加载session配置项

直接读取配置项生成session对象

### 加载auth plugin配置项

```python
keystoneauth1.loading.conf.py

def load_from_conf_options(conf, group, **kwargs):
    if conf[group].auth_section:
        group = conf[group].auth_section

    name = conf[group].auth_type
    if not name:
        return None

    plugin = base.get_plugin_loader(name)
    plugin_opts = plugin.get_options()
    oslo_opts = [o._to_oslo_opt() for o in plugin_opts]

    conf.register_opts(oslo_opts, group=group)

    def _getter(opt):
        return conf[group][opt.dest]

    return plugin.load_from_options_getter(_getter, **kwargs)
```
这里先读取auth_type配置项的值，再根据auth_type注册相应的配置项，最后读取配置文件，生成auth_plugin。

```python
keystoneauth1.loading.base.py

PLUGIN_NAMESPACE = 'keystoneauth1.plugin'

def get_plugin_loader(name):
    try:
        mgr = stevedore.DriverManager(namespace=PLUGIN_NAMESPACE,
                                      invoke_on_load=True,
                                      name=name)
    except RuntimeError:
        raise exceptions.NoMatchingPlugin(name)

    return mgr.driver
```

这里通过stevedore来来管理auth plugin驱动。

```python
[entry_points]
keystoneauth1.plugin =
    password = keystoneauth1.loading._plugins.identity.generic:Password
    token = keystoneauth1.loading._plugins.identity.generic:Token
    admin_token = keystoneauth1.loading._plugins.admin_token:AdminToken
    v2password = keystoneauth1.loading._plugins.identity.v2:Password
    v2token = keystoneauth1.loading._plugins.identity.v2:Token
    v3password = keystoneauth1.loading._plugins.identity.v3:Password
    v3token = keystoneauth1.loading._plugins.identity.v3:Token
    v3oidcpassword = keystoneauth1.loading._plugins.identity.v3:OpenIDConnectPassword
    v3oidcauthcode = keystoneauth1.loading._plugins.identity.v3:OpenIDConnectAuthorizationCode
```

auth_type为password时，驱动为`keystoneauth1.loading._plugins.identity.generic:Password`。

再通过get_options来获取所需配置项。

```python
keystoneauth1.loading._plugins.identity.generic

class Password(loading.BaseGenericLoader):

    @property
    def plugin_class(self):
        return identity.Password

    def get_options(cls):
        options = super(Password, cls).get_options()
        options.extend([
            loading.Opt('user-id', help='User id'),
            loading.Opt('username',
                        help='Username',
                        deprecated=[loading.Opt('user-name')]),
            loading.Opt('user-domain-id', help="User's domain id"),
            loading.Opt('user-domain-name', help="User's domain name"),
            loading.Opt('password', secret=True, help="User's password"),
        ])
        return options
```

```python
keystoneauth1.loading.identity.py

class BaseGenericLoader(BaseIdentityLoader):
    def get_options(self):
        options = super(BaseGenericLoader, self).get_options()

        options.extend([
            opts.Opt('domain-id', help='Domain ID to scope to'),
            opts.Opt('domain-name', help='Domain name to scope to'),
            opts.Opt('project-id', help='Project ID to scope to',
                     deprecated=[opts.Opt('tenant-id')]),
            opts.Opt('project-name', help='Project name to scope to',
                     deprecated=[opts.Opt('tenant-name')]),
            opts.Opt('project-domain-id',
                     help='Domain ID containing project'),
            opts.Opt('project-domain-name',
                     help='Domain name containing project'),
            opts.Opt('trust-id', help='Trust ID'),
            opts.Opt('default-domain-id',
                     help='Optional domain ID to use with v3 and v2 '
                          'parameters. It will be used for both the user '
                          'and project domain in v3 and ignored in '
                          'v2 authentication.'),
            opts.Opt('default-domain-name',
                     help='Optional domain name to use with v3 API and v2 '
                          'parameters. It will be used for both the user '
                          'and project domain in v3 and ignored in '
                          'v2 authentication.'),
        ])

        return options
    
class BaseIdentityLoader(base.BaseLoader):
    def get_options(self):
        options = super(BaseIdentityLoader, self).get_options()

        options.extend([
            opts.Opt('auth-url',
                     required=True,
                     help='Authentication URL'),
        ])

        return options
```

可以看到在加载auth plugin配置项时，会先根据auth_type决定需要加载哪些配置项，然后用这些配置项生成auth plugin。

### 生成client

用session和auth plugin生成client。