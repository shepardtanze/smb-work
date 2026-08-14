# 引用计数泄露修复分析

## 1. 两个 session 索引

| 索引 | 加入位置 | 释放位置 |
|---|---|---|
| `sessions_table` | `__session_create()` 调用 `hash_add()` | `ksmbd_expire_session()`、`ksmbd_sessions_deregister()` |
| `conn->sessions` | `ksmbd_session_register()` 调用 `xa_store()` | `ksmbd_expire_session()` 或 `ksmbd_sessions_deregister()` 调用 `xa_erase()` |



## 2. session 如何加入两个索引

```text
smb2_sess_setup()
  -- ksmbd_smb2_session_create()
 |  -- __session_create()
 |    -- hash_add(sessions_table, &sess->hlist, sess->id) // 加入全局链表
 |
  -- ksmbd_session_register(conn, sess)
        -- ksmbd_expire_session(conn)
        -- xa_store(&conn->sessions, sess->id, sess) //添加到struct xarray sessions;

```

加入顺序是：

```text
先由 __session_create() 加入 sessions_table
        ↓
再由 ksmbd_session_register() 加入 conn->sessions
```



如果 `xa_store()` 成功，两个索引中都有 session；

如果 XArray 内部节点分配失败，`xa_store()` 返回 `-ENOMEM`，`conn->sessions`就不会变更。

## 3. 正常情况下如何删除两个索引

### 3.1 过期清理

```text
smb2_sess_setup()
  -- ksmbd_session_register()
        -- ksmbd_expire_session()
              -- xa_for_each(&conn->sessions, ...)
              -- xa_erase(&conn->sessions, sess->id)
              -- hash_del(&sess->hlist)
              -- ksmbd_session_destroy(sess)
```

`ksmbd_expire_session()` 先从 `conn->sessions` 找到过期 session，然后同时删除
连接 XArray 和全局哈希表中的索引。

### 3.2 连接注销清理

```text
ksmbd_conn_handler_loop()
  -- default_conn_ops.terminate_fn(conn)
        -- ksmbd_server_terminate_conn()
              -- ksmbd_sessions_deregister()
                    -- xa_erase(&conn->sessions, sess->id)
                    -- hash_del(&sess->hlist)
```

`ksmbd_sessions_deregister()` 负责连接退出时的 session 清理。

## 4. 补丁 修复前为什么泄漏

失败调用关系如下：

```text
smb2_sess_setup()
  |
   -- ksmbd_smb2_session_create()
  |  -- __session_create()
  |      -- hash_add(sessions_table, sess)
  |
   -- ksmbd_session_register()
  |  -- ksmbd_expire_session(conn)
  |    -- xa_store(conn->sessions, sess) //如果失败，sess 没有加入 conn->sessions
  |
   -- out_err
     -- sess->state = SMB2_SESSION_EXPIRED
       -- ksmbd_user_session_put(sess)：//引用计数refcnt 2 -> 1
```

最终状态：

```text
sessions_table：有 sess
conn->sessions：没有 sess
refcnt：1
```

`ksmbd_expire_session()` 只遍历 `conn->sessions`。但是sess 没有加入 conn->sessions，所以之后无法被过期清理找到。
仅设置`SMB2_SESSION_EXPIRED` 不会自动删除全局哈希项，因此 `sessions_table` 和最后
一个引用一直保留。

## 5. 问题提交 如何引入问题

问题提交f5c779b7ddbd 之前的错误清理关系是：

```text
smb2_sess_setup() 错误路径
  -- xa_erase(&conn->sessions, sess->id)
  -- ksmbd_session_destroy(sess)
    -- hash_del(&sess->hlist)   
  -- work->sess = NULL
```

如果 `xa_store()` 已经失败，`xa_erase()` 实际没有 entry 可以删除；但当时的
`ksmbd_session_destroy()` 仍会执行 `hash_del()`，所以 `sessions_table` 能被
清理。

f5c779b7ddbd 将上述清理替换为：

```c
sess->state = SMB2_SESSION_EXPIRED;
```

这样就不再调用 `ksmbd_session_destroy()`，也不再删除 `sessions_table`。而注册
失败的 session 不在 `conn->sessions` 中，后续过期清理又找不到它。因此 f5
引入了这个泄漏问题。

## 6. 修复补丁 如何修复

5d 在 `ksmbd_session_register()` 的失败分支中增加：

```c
if (ret) {
	down_write(&sessions_table_lock);
	hash_del(&sess->hlist);
	up_write(&sessions_table_lock);
	ksmbd_user_session_put(sess);
}
```

修复后的调用关系：

```text
smb2_sess_setup()
   -- ksmbd_session_register()
  |  -- xa_store() 失败
  |    -- hash_del(&sess->hlist) //删除 sessions_table 中的索引
  |    -- ksmbd_user_session_put(sess) //refcnt 2 -> 1
  |           
  |
   -- out_err
     -- ksmbd_user_session_put(sess) //refcnt 1 -> 0
       -- ksmbd_session_destroy(sess)
```

