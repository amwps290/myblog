adminpack 提供了大量支持功能，pgAdmin 和其他管理工具可以使用这些功能提供额外功能，例如远程管理服务器日志文件。默认情况下，只有数据库超级用户才能使用所有这些功能，但其他用户也可以使用 GRANT 命令使用这些功能。


我们先来看一下他支持的函数，可以通过 \dx+ adminpack 来进行查看

+ function pg_file_rename(text,text) 重命名文件
+ function pg_file_rename(text,text,text) 重命名文件，如果新文件存在，将将其命名为第三个参数的名字
+ function pg_file_sync(text)       文件刷入磁盘
+ function pg_file_unlink(text)   删除文件
+ function pg_file_write(text,text,boolean) 写文件
+ function pg_logdir_ls()   列出日志目录下的文件


## pg_file_rename(text,text)

用于重命名文件，我们看一下 sql 代码

```sql
CREATE FUNCTION pg_catalog.pg_file_rename(text, text)
RETURNS bool
AS 'SELECT pg_catalog.pg_file_rename($1, $2, NULL::pg_catalog.text);'
LANGUAGE SQL VOLATILE STRICT;
```

这里我们看到两个参数版本的 `pg_file_rename` 直接调用来三参数版本的 `pg_file_rename`, 因此我们直接查看三参数版本的 SQL 代码

```sql
CREATE OR REPLACE FUNCTION pg_catalog.pg_file_rename(text, text, text)
RETURNS bool
AS 'MODULE_PATHNAME', 'pg_file_rename_v1_1'
LANGUAGE C VOLATILE;
```
这个 SQL 代码直接调用来 C 函数 `pg_file_rename_v1_1` 来实现文件重命名。

现在我们来看一下 C 函数 `pg_file_rename_v1_1`

```C
Datum
pg_file_rename_v1_1(PG_FUNCTION_ARGS)
{
	text	   *file1;
	text	   *file2;
	text	   *file3;
	bool		result;

	if (PG_ARGISNULL(0) || PG_ARGISNULL(1))
		PG_RETURN_NULL();

	file1 = PG_GETARG_TEXT_PP(0);
	file2 = PG_GETARG_TEXT_PP(1);

	if (PG_ARGISNULL(2))
		file3 = NULL;
	else
		file3 = PG_GETARG_TEXT_PP(2);

	result = pg_file_rename_internal(file1, file2, file3);

	PG_RETURN_BOOL(result);
}
```

这个代码中仅仅是判断参数是否为空，如果不为空，则获取参数，然后调用  `pg_file_rename_internal` 这个函数

```C
static bool
pg_file_rename_internal(text *file1, text *file2, text *file3)
{
	char	   *fn1,
			   *fn2,
			   *fn3;
	int			rc;

	fn1 = convert_and_check_filename(file1);
	fn2 = convert_and_check_filename(file2);

	if (file3 == NULL)
		fn3 = NULL;
	else
		fn3 = convert_and_check_filename(file3);

	if (access(fn1, W_OK) < 0)
	{
		ereport(WARNING,
				(errcode_for_file_access(),
				 errmsg("file \"%s\" is not accessible: %m", fn1)));

		return false;
	}

	if (fn3 && access(fn2, W_OK) < 0)
	{
		ereport(WARNING,
				(errcode_for_file_access(),
				 errmsg("file \"%s\" is not accessible: %m", fn2)));

		return false;
	}

	rc = access(fn3 ? fn3 : fn2, W_OK);
	if (rc >= 0 || errno != ENOENT)
	{
		ereport(ERROR,
				(errcode(ERRCODE_DUPLICATE_FILE),
				 errmsg("cannot rename to target file \"%s\"",
						fn3 ? fn3 : fn2)));
	}

	if (fn3)
	{
		if (rename(fn2, fn3) != 0)
		{
			ereport(ERROR,
					(errcode_for_file_access(),
					 errmsg("could not rename \"%s\" to \"%s\": %m",
							fn2, fn3)));
		}
		if (rename(fn1, fn2) != 0)
		{
			ereport(WARNING,
					(errcode_for_file_access(),
					 errmsg("could not rename \"%s\" to \"%s\": %m",
							fn1, fn2)));

			if (rename(fn3, fn2) != 0)
			{
				ereport(ERROR,
						(errcode_for_file_access(),
						 errmsg("could not rename \"%s\" back to \"%s\": %m",
								fn3, fn2)));
			}
			else
			{
				ereport(ERROR,
						(errcode(ERRCODE_UNDEFINED_FILE),
						 errmsg("renaming \"%s\" to \"%s\" was reverted",
								fn2, fn3)));
			}
		}
	}
	else if (rename(fn1, fn2) != 0)
	{
		ereport(ERROR,
				(errcode_for_file_access(),
				 errmsg("could not rename \"%s\" to \"%s\": %m", fn1, fn2)));
	}

	return true;
}
```

这个函数的整体逻辑为，先将 text* 类型的数据转换为 char* 类型的数据，会在这个转换的过程中处理一些路径相关和权限验证的问题。

然后先判断一下 fn1 是不是存在，如果不存在那肯定是没法重命名的，然后判断一下 fn3 是不是为空，并且 fn2 是不是存在，如果不存在，那么将 fn2 重命名为 fn3 也会失败。

然后判断一下 fn3 是不是存在，如果存在，那么说明文件已经存在，肯定不能重命名，也会报一个 DUPLICATE 的错误。如果 fn3 为空，那么就判断 fn2 文件是不是存在，如果存在那也是不能重命名的。


接下来，我们就可以将 fn2 重命名成 fn3, 然后将 fn1 重命名为 fn2,

如果 fn1 重命名为 fn2 出错，则将 fn3 重名名为 fn2 ,即撤消之前的修改操作。

那如果 fn3 为空，直接将 fn1 重命名为 fn2 就可以了，


## pg_file_sync

我们先来看一下 SQL 代码：

```SQL
CREATE OR REPLACE FUNCTION pg_catalog.pg_file_sync(text)
RETURNS void
AS 'MODULE_PATHNAME', 'pg_file_sync'
LANGUAGE C VOLATILE STRICT;
```

可以看到他调用的是 C 函数 `pg_file_sync` ,我们来看一下这个 C 代码：

```C
Datum
pg_file_sync(PG_FUNCTION_ARGS)
{
	char	   *filename;
	struct stat fst;

	filename = convert_and_check_filename(PG_GETARG_TEXT_PP(0));

	if (stat(filename, &fst) < 0)
		ereport(ERROR,
				(errcode_for_file_access(),
				 errmsg("could not stat file \"%s\": %m", filename)));

	fsync_fname_ext(filename, S_ISDIR(fst.st_mode), false, ERROR);

	PG_RETURN_VOID();
}
```


可以看到这个仅仅是将 text 类型的数据转换为 char * 类型的数据。然后调用 storage 的函数实现的功能。(src/backend/storage/file/fd.c)

## pg_file_unlink

我们先来看一下 SQL 代码：

```SQL
CREATE OR REPLACE FUNCTION pg_catalog.pg_file_unlink(text)
RETURNS bool
AS 'MODULE_PATHNAME', 'pg_file_unlink_v1_1'
LANGUAGE C VOLATILE STRICT;
```
发现它调用了 C 函数 `pg_file_unlink_v1_1`. 我们来看一下这个函数：

```C
Datum
pg_file_unlink_v1_1(PG_FUNCTION_ARGS)
{
	char	   *filename;

	filename = convert_and_check_filename(PG_GETARG_TEXT_PP(0));

	if (access(filename, W_OK) < 0)
	{
		if (errno == ENOENT)
			PG_RETURN_BOOL(false);
		else
			ereport(ERROR,
					(errcode_for_file_access(),
					 errmsg("file \"%s\" is not accessible: %m", filename)));
	}

	if (unlink(filename) < 0)
	{
		ereport(WARNING,
				(errcode_for_file_access(),
				 errmsg("could not unlink file \"%s\": %m", filename)));

		PG_RETURN_BOOL(false);
	}
	PG_RETURN_BOOL(true);
}
```
这个函数的整体逻辑是将 text* 类型的数据转换为 char* 类型的数据，并处理路径相关的问题，然后判断一下文件是不是可访问的。然后调用  `unlink` 对文件进行删除。

## pg_file_write

看一下 SQL 代码：


```SQL
CREATE OR REPLACE FUNCTION pg_catalog.pg_file_write(text, text, bool)
RETURNS bigint
AS 'MODULE_PATHNAME', 'pg_file_write_v1_1'
LANGUAGE C VOLATILE STRICT;
```

发现它调用的是 C 函数 `pg_file_write_v1_1`

```C
Datum
pg_file_write_v1_1(PG_FUNCTION_ARGS)
{
	text	   *file = PG_GETARG_TEXT_PP(0);
	text	   *data = PG_GETARG_TEXT_PP(1);
	bool		replace = PG_GETARG_BOOL(2);
	int64		count = 0;

	count = pg_file_write_internal(file, data, replace);

	PG_RETURN_INT64(count);
}

/* ------------------------------------
 * pg_file_write_internal - Workhorse for pg_file_write functions.
 *
 * This handles the actual work for pg_file_write.
 */
static int64
pg_file_write_internal(text *file, text *data, bool replace)
{
	FILE	   *f;
	char	   *filename;
	int64		count = 0;

	filename = convert_and_check_filename(file);

	if (!replace)
	{
		struct stat fst;

		if (stat(filename, &fst) >= 0)
			ereport(ERROR,
					(errcode(ERRCODE_DUPLICATE_FILE),
					 errmsg("file \"%s\" exists", filename)));

		f = AllocateFile(filename, "wb");
	}
	else
		f = AllocateFile(filename, "ab");

	if (!f)
		ereport(ERROR,
				(errcode_for_file_access(),
				 errmsg("could not open file \"%s\" for writing: %m",
						filename)));

	count = fwrite(VARDATA_ANY(data), 1, VARSIZE_ANY_EXHDR(data), f);
	if (count != VARSIZE_ANY_EXHDR(data) || FreeFile(f))
		ereport(ERROR,
				(errcode_for_file_access(),
				 errmsg("could not write file \"%s\": %m", filename)));

	return (count);
}
```
我们可以看到 `pg_file_write_v1_1` 仅仅是获取了参数，然后就调用了 `pg_file_write_internal` 函数。
这个函数的主要逻辑是将 text* 的数据转换为 char* 的数据 。然后判断一下 `replace` 参数，如果为 false,则文件不能存在，然后使用 `AllocateFile` 创建一个文件。
然后使用 `fwrite` 将数据写入文件，`VARDATA_ANY` 宏的作用是获取实际的数据指针，`VARSIZE_ANY_EXHDR` 的作用是获取数据的长度。最后返回写入的长度。


## pg_logdir_ls

我们看一下 SQL 代码：

```SQL
CREATE OR REPLACE FUNCTION pg_catalog.pg_logdir_ls()
RETURNS setof record
AS 'MODULE_PATHNAME', 'pg_logdir_ls_v1_1'
LANGUAGE C VOLATILE STRICT;
```

这里它调用了 C 函数 `pg_logidr_ls_v1_1`,我们看一下这个 C 函数：

```C
Datum
pg_logdir_ls_v1_1(PG_FUNCTION_ARGS)
{
	return (pg_logdir_ls_internal(fcinfo));
}

static Datum
pg_logdir_ls_internal(FunctionCallInfo fcinfo)
{
	ReturnSetInfo *rsinfo = (ReturnSetInfo *) fcinfo->resultinfo;
	bool		randomAccess;
	TupleDesc	tupdesc;
	Tuplestorestate *tupstore;
	AttInMetadata *attinmeta;
	DIR		   *dirdesc;
	struct dirent *de;
	MemoryContext oldcontext;

	if (strcmp(Log_filename, "postgresql-%Y-%m-%d_%H%M%S.log") != 0)
		ereport(ERROR,
				(errcode(ERRCODE_INVALID_PARAMETER_VALUE),
				 errmsg("the log_filename parameter must equal 'postgresql-%%Y-%%m-%%d_%%H%%M%%S.log'")));

	/* check to see if caller supports us returning a tuplestore */
	if (rsinfo == NULL || !IsA(rsinfo, ReturnSetInfo))
		ereport(ERROR,
				(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
				 errmsg("set-valued function called in context that cannot accept a set")));
	if (!(rsinfo->allowedModes & SFRM_Materialize))
		ereport(ERROR,
				(errcode(ERRCODE_SYNTAX_ERROR),
				 errmsg("materialize mode required, but it is not allowed in this context")));

	/* The tupdesc and tuplestore must be created in ecxt_per_query_memory */
	oldcontext = MemoryContextSwitchTo(rsinfo->econtext->ecxt_per_query_memory);

	tupdesc = CreateTemplateTupleDesc(2);
	TupleDescInitEntry(tupdesc, (AttrNumber) 1, "starttime",
					   TIMESTAMPOID, -1, 0);
	TupleDescInitEntry(tupdesc, (AttrNumber) 2, "filename",
					   TEXTOID, -1, 0);

	randomAccess = (rsinfo->allowedModes & SFRM_Materialize_Random) != 0;
	tupstore = tuplestore_begin_heap(randomAccess, false, work_mem);
	rsinfo->returnMode = SFRM_Materialize;
	rsinfo->setResult = tupstore;
	rsinfo->setDesc = tupdesc;

	MemoryContextSwitchTo(oldcontext);

	attinmeta = TupleDescGetAttInMetadata(tupdesc);

	dirdesc = AllocateDir(Log_directory);
	while ((de = ReadDir(dirdesc, Log_directory)) != NULL)
	{
		char	   *values[2];
		HeapTuple	tuple;
		char		timestampbuf[32];
		char	   *field[MAXDATEFIELDS];
		char		lowstr[MAXDATELEN + 1];
		int			dtype;
		int			nf,
					ftype[MAXDATEFIELDS];
		fsec_t		fsec;
		int			tz = 0;
		struct pg_tm date;

		/*
		 * Default format: postgresql-YYYY-MM-DD_HHMMSS.log
		 */
		if (strlen(de->d_name) != 32
			|| strncmp(de->d_name, "postgresql-", 11) != 0
			|| de->d_name[21] != '_'
			|| strcmp(de->d_name + 28, ".log") != 0)
			continue;

		/* extract timestamp portion of filename */
		strcpy(timestampbuf, de->d_name + 11);
		timestampbuf[17] = '\0';

		/* parse and decode expected timestamp to verify it's OK format */
		if (ParseDateTime(timestampbuf, lowstr, MAXDATELEN, field, ftype, MAXDATEFIELDS, &nf))
			continue;

		if (DecodeDateTime(field, ftype, nf, &dtype, &date, &fsec, &tz))
			continue;

		/* Seems the timestamp is OK; prepare and return tuple */

		values[0] = timestampbuf;
		values[1] = psprintf("%s/%s", Log_directory, de->d_name);

		tuple = BuildTupleFromCStrings(attinmeta, values);

		tuplestore_puttuple(tupstore, tuple);
	}

	FreeDir(dirdesc);
	return (Datum) 0;
}
```

这里它直接调用了 `pg_logdir_ls_internal` 函数，其中的 `fcinfo` 是这个宏展开的：

`#define PG_FUNCTION_ARGS	FunctionCallInfo fcinfo`

我们来看一下 `pg_logdir_ls_internal` 函数：

首先我们看到了它从 `fcinfo` 中获取了一个 `resultinfo` 的字段，然后将其转换为 `ReturnSetInfo` 这个结构体指针。我们来看一下这个结构体

```C
/*
 * When calling a function that might return a set (multiple rows),
 * a node of this type is passed as fcinfo->resultinfo to allow
 * return status to be passed back.  A function returning set should
 * raise an error if no such resultinfo is provided.
 */
typedef struct ReturnSetInfo
{
	NodeTag		type;
	/* values set by caller: */
	ExprContext *econtext;		/* context function is being called in */
	TupleDesc	expectedDesc;	/* tuple descriptor expected by caller */
	int			allowedModes;	/* bitmask: return modes caller can handle */
	/* result status from function (but pre-initialized by caller): */
	SetFunctionReturnMode returnMode;	/* actual return mode */
	ExprDoneCond isDone;		/* status for ValuePerCall mode */
	/* fields filled by function in Materialize return mode: */
	Tuplestorestate *setResult; /* holds the complete returned tuple set */
	TupleDesc	setDesc;		/* actual descriptor for returned tuples */
} ReturnSetInfo;
```

从这个注释中我们可以看到，当我们调用的函数需要返回一个集合，即多行数据的时候，我们就可以使用这个结构体，而 `resultinfo` 的实际类型是一个 `fmNodePtr` ,实际上就是一个 Node 节点。而 `ReturnSetInfo` 也是一个 Node 节点。

接下来我们定义来一些用于返回结果需要的辅助数据结构。

第一个判断我们的 `Log_filename` 的格式是不是符合 `postgresql-%Y-%m-%d_%H%M%S.log`  这个规则，不符合这个规则是无法使用这个函数的。`Log_filename` 是一个 GUC 参数，你可以在 `postgresql.conf` 中修改，也可以使用 `show log_filename` 命令来查看当前的值。

然后我们检查一下 `rsinfo` 变量的类型是不是一个 `ReturnSetInfo`,实际上就是通过  NodeTag 进行比较的。

接下来我们检查一下 `rsinfo` 中的 `allowedMode` 返回模式是否是 `SFRM_Materialize` 这个模式可以让我们将要返回的数据都存储在 `Tuplestore` 中。

接下来我们需要将内存上下文切换到 `rsinfo->econtext->ecxt_per_query_memory` ，在这个内存上下文中我们保存要返回的结果，即这里的 tuple.

然后我们使用 `CreateTemplateTupleDesc` 来创建一个 tuple 的描述，其实我感觉可以理解为 DDL ,即这个表各个字段是什么样的数据。这里我们添加了两个字段，其数据类型分别为 `TIMESTAMP` 和 `TEXT`.


然后我们使用 `tuplestore_begin_heap` 函数创建一个 `tuplestore` ，我们最终的结果也将存放在这个数据结构中。

`attinmeta = TupleDescGetAttInMetadata(tupdesc);` 这个语句把 tuple 的属性信息存储到 `attinmeta` 这个会在我们后续构造 tuple 的时候要用到。

接下来，我们打开目录，这个 `Log_directory` 也是一个 GUC 参数可以设置和查看。
这个目录是相对于 data 目录来说的。

接下来，我们遍历目录中的文件，查看文件的名字是否满足 `postgresql-` 这种形式，然后将文件名中包含的日期信息提取出来，然后将其变成好看的日期格式，最后我们将日期和文件名写入到一个 tuple 里

```C
values[0] = timestampbuf;
values[1] = psprintf("%s/%s", Log_directory, de->d_name);
tuple = BuildTupleFromCStrings(attinmeta, values);
```
其中的 `attinmeta` 就是我们之前准备好的 tuple 描述信息，构造好一个 tuple 以后，我们就可以把它放在 tuplestore 里边了。

`tuplestore_puttuple(tupstore, tuple);`

最后关闭目录。




