auth_delay 让服务器在报告身份验证失败前短暂暂停，以增加对数据库密码进行暴力破解的难度。需要注意的是，这对阻止拒绝服务攻击毫无帮助，甚至可能加剧攻击，因为在报告身份验证失败前等待的进程仍会占用连接。

要使用这个模块必须要在 `postgresql.conf` 中配置参数
```
shared_preload_libraries = 'auth_delay'
auth_delay.milliseconds = '500'
```


这个代码比较简单，一共分为三个部分。

1. hook 函数

在 libpq 中定义了一个 `ClientAuthentication_hook` 函数指针，代码如下：

```C
typedef void (*ClientAuthentication_hook_type) (Port *, int);

/*
 * This hook allows plugins to get control following client authentication,
 * but before the user has been informed about the results.  It could be used
 * to record login events, insert a delay after failed authentication, etc.
 */
ClientAuthentication_hook_type ClientAuthentication_hook = NULL;

```

2. _PG_init 函数

```C
void
_PG_init(void)
{
	/* Define custom GUC variables */
	DefineCustomIntVariable("auth_delay.milliseconds",
							"Milliseconds to delay before reporting authentication failure",
							NULL,
							&auth_delay_milliseconds,
							0,
							0, INT_MAX / 1000,
							PGC_SIGHUP,
							GUC_UNIT_MS,
							NULL,
							NULL,
							NULL);
	/* Install Hooks */
	original_client_auth_hook = ClientAuthentication_hook;
	ClientAuthentication_hook = auth_delay_checks;
}

```

这个 `_PG_init` 函数会在模块调用之前执行，这个函数主要实现的功能是定义了一个 GUC 参数 `auth_delay.milliseconds` 这个参数我们需要在 postgresql.conf 中进行配置，然后它将原来的钩子函数保存在了 `original_client_auth_hook` 变量中，然后将我们自己的 `auth_delay_checks` 函数赋给了 `ClientAuthentication_hook` 变量，这样我们就可以让数据库调用我们的函数了。

3. auth_delay_checks 函数

```C
static void
auth_delay_checks(Port *port, int status)
{
	/*
	 * Any other plugins which use ClientAuthentication_hook.
	 */
	if (original_client_auth_hook)
		original_client_auth_hook(port, status);

	/*
	 * Inject a short delay if authentication failed.
	 */
	if (status != STATUS_OK)
	{
		pg_usleep(1000L * auth_delay_milliseconds);
	}
}
```

这个函数的意思是如果原来存在有钩子函数就先运行原来的钩子函数，然后我们看一下验证的状态 status 是不是正确，如果不正确，就延时 auth_delay_milliseconds 个时间。