---
Title: Redis breaking changes
linkTitle: Breaking changes
description: Potentially breaking changes in Redis Enterprise, introduced by new versions of open source Redis. 
weight: 90
alwaysopen: false
toc: "false"
categories: ["RS"]
aliases: 
---

When new major versions of open source Redis change existing commands, upgrading your database to a new version can potentially break some functionality. Before you upgrade, make sure to read the provided list of breaking changes that affect Redis Enterprise and update any applications that connect to your database to handle these changes.

To check your Redis database version (`redis_version`), you can use the admin console or run the [`INFO`](https://redis.io/commands/info/) command with [`redis-cli`]({{<relref "/rs/references/cli-utilities/redis-cli">}}):

```sh
$ redis-cli -p <port> INFO
"# Server
redis_version:7.0.8
..."
```

## Redis 7.0 breaking changes

Open source Redis version 7.0 introduces the following potentially breaking changes to Redis Enterprise:

### Programmability

-  Lua scripts no longer have access to the `print()` function ([#10651](https://github.com/redis/redis/pull/10651)) - The `print`  function was removed from Lua because it can potentially cause the Redis processes to get stuck (if no one reads from stdout). Users should use redis.log. An alternative is to override the  `print`  implementation and print the message to the log file.  

- Block [`PFCOUNT`](https://redis.io/commands/pfcount/) and [`PUBLISH`](https://redis.io/commands/publish/) in read-only scripts (*_RO commands,  `no-writes`) ([#10744](https://github.com/redis/redis/pull/10744)) - Consider `PFCOUNT` and `PUBLISH` as write commands in scripts, in addition to `EVAL`; meaning:
  - They can never be used in scripts with shebang (`#!`) and no `no-writes` flag
  - They are blocked in `EVAL_RO` and `_RO` variants, (even in scripts without shebang (`#!`) flags)
  - Allow `no-write` scripts in EVAL (not just in EVAL_RO), even during `CLIENT PAUSE WRITE` 

- Hide the `may_replicate` flag from the [`COMMAND`](https://redis.io/commands/command/) command response  ([#10744](https://github.com/redis/redis/pull/10744)) - As part of the change to treat `may_replicate` commands `PFCOUNT` and `PUBLISH` as write commands in scripts, in addition to `EVAL`, the `may_replicate` flag has been removed from the `COMMAND` response.

### Error handling

- Rephrased some error responses about invalid commands or arguments ([#10612](https://github.com/redis/redis/pull/10612)) - 
  - Error response for unknown command introduced a case change (`Unknown` to `unknown`)
  - Errors for module commands extended to cover subcommands, updated syntax to match Redis Server syntax
  - Arity errors for module commands introduce a case change (`Wrong` to `wrong`); will consider full command name

- Corrected error codes returned from [`EVAL`](https://redis.io/commands/eval/) scripts ([#10218](https://github.com/redis/redis/pull/10218), [#10329](https://github.com/redis/redis/pull/10329)). 

  These examples show changes in behavior:

  ```diff
    1: config set maxmemory 1
    2: +OK
    3: eval "return redis.call('set','x','y')" 0
  - 4: -ERR Error running script (call to 71e6319f97b0fe8bdfa1c5df3ce4489946dda479): @user_script:1: @user_script: 1: -OOM command not allowed when used memory > 'maxmemory'.
  + 4: -ERR Error running script (call to 71e6319f97b0fe8bdfa1c5df3ce4489946dda479): @user_script:1: OOM command not allowed when used memory > 'maxmemory'.
    5: eval "return redis.pcall('set','x','y')" 0
  - 6: -@user_script: 1: -OOM command not allowed when used memory > 'maxmemory'.
  + 6: -OOM command not allowed when used memory > 'maxmemory'.
    7: eval "return redis.call('select',99)" 0
    8: -ERR Error running script (call to 4ad5abfc50bbccb484223905f9a16f09cd043ba8): @user_script:1: ERR DB index is out of range
    9: eval "return redis.pcall('select',99)" 0
   10: -ERR DB index is out of range
   11: eval_ro "return redis.call('set','x','y')" 0
  -12: -ERR Error running script (call to 71e6319f97b0fe8bdfa1c5df3ce4489946dda479): @user_script:1: @user_script: 1: Write commands are not allowed from read-only scripts.
  +12: -ERR Error running script (call to 71e6319f97b0fe8bdfa1c5df3ce4489946dda479): @user_script:1: ERR Write commands are not allowed from read-only scripts.
   13: eval_ro "return redis.pcall('set','x','y')" 0
  -14: -@user_script: 1: Write commands are not allowed from read-only scripts.
  +14: -ERR Write commands are not allowed from read-only scripts.
  ```

- [`ZPOPMIN`](https://redis.io/commands/zpopmin/)/[`ZPOPMAX`](https://redis.io/commands/zpopmax/) used to produce wrong replies when count is 0 with non-zset  [#9711](https://github.com/redis/redis/pull/9711)):
  -  `ZPOPMIN`/`ZPOPMAX` used to produce an `(empty array)` when `key` was not a sorted set and the optional `count` argument was set to `0` and now produces a `WRONGTYPE` error response instead.
  -  The optional `count` argument must be positive. A negative value produces a `value is out of range` error.

  These examples show changes in behavior:

  ```diff
    1: zadd myzset 1 "one"
    2: (integer) 1
    3: zadd myzset 2 "two"
    4: (integer) 1
    5: zadd myzset 3 "three"
    6: (integer) 1
    7: zpopmin myzset -1
  - 8: (empty array)
  + 8: (error) ERR value is out of range, must be positive
    9: 127.0.0.1:6379> set foo bar
   10: OK
   11: zpopmin foo 0
  -12: (empty array)
  +12: (error) WRONGTYPE Operation against a key holding the wrong kind of value
  ```

- [`LPOP`](https://redis.io/commands/lpop/)/[`RPOP`](https://redis.io/commands/rpop/) with count against a nonexistent list returns a null array instead of `(nil)`([#10095](https://github.com/redis/redis/pull/10095)). This change was backported to 6.2.

- [`LPOP`](https://redis.io/commands/lpop/)/[`RPOP`](https://redis.io/commands/rpop/) used to produce `(nil)` when count is 0, now produces a null array ([#9692](https://github.com/redis/redis/pull/9692)). This change was backported to 6.2.

- [`XCLAIM`](https://redis.io/commands/xclaim/)/[`XAUTOCLAIM`](https://redis.io/commands/xautoclaim/) skips deleted entries instead of replying with `nil` and deletes them from the pending entry list ([#10227](https://github.com/redis/redis/pull/10227)) - `XCLAIM`/`XAUTOCLAIM` now behaves in the following way:
  - If you try to claim a deleted entry, it is deleted from the pending entry list (PEL) where it is found (as well as the group PEL). Therefore, such an entry is not claimed, just cleared from PEL (because it doesn't exist in the stream anyway).
  - Because deleted entries are not claimed, `X[AUTO]CLAIM` does not return "nil" instead of an entry.
  - Added an array of all the deleted stream IDs to `XAUTOCLAIM` response.

###  ACLs

- [`ACL GETUSER`](https://redis.io/commands/acl-getuser/) reply now uses ACL syntax for `keys` and `channels` ([#9974](https://github.com/redis/redis/pull/9974)). `ACL GETUSER` now uses the ACL DSL (Domain Specific Language) for keys and channels. 

  These examples show changes in behavior:
  ```diff
    1: acl setuser foo off resetchannels &channel1 -@all +get
    2: OK
    3: acl getuser foo
    4: 1) "flags"
    5: 2) 1) "off"
    6: 3) "passwords"
    7: 4) (empty array)
    8: 5) "commands"
    9: 6) "-@all +get"
   10: 7) "keys"
  -11: 8) (empty array)
  +11: 8) ""
   12: 9)"channels"
  -13 10) 1) "channel1"
  +13 10) "&channel1"
  ```

- [`SORT`](https://redis.io/commands/sort/)/[`SORT_RO`](https://redis.io/commands/sort_ro/) commands reject key access patterns in `GET` and `BY` if the ACL doesn't grant the command full keyspace access ([#10340](https://github.com/redis/redis/pull/10340)) - The `sort` and `sort_ro` commands can access external keys via `GET` and `BY`. In order to make sure the user cannot violate the authorization ACL rules, Redis 7 will reject external keys access patterns unless ACL allows `SORT` full access to all keys.
For backwards compatibility, `SORT` with `GET`/`BY` keeps working, but if ACL has restrictions to certain keys, the use of these features will result in a permission denied error.

  These examples show changes in behavior:

  ```
  USER FOO (+sort ~* ~mylist) 
  #FOO> sort mylist by w* get v*  - is O.K since ~* provides full key access
  ```

  ```
  USER FOO (+sort %R~* ~mylist) 
  #FOO> sort mylist by w* get v*  - is O.K since %R~* provides full key READ access**
  ```

  ```
  USER FOO (+sort %W~* ~mylist)
  #FOO> sort mylist by w* get v*  - will now fail since $W~* only provides full key WRITE access
  ```

  ```
  USER FOO (+sort ~v* ~mylist)
  #FOO> sort mylist by w* get v*  - will now fail since ~v* only provides partial key access
  ```

- Fix ACL category for [`SELECT`](https://redis.io/commands/select/), [`WAIT`](https://redis.io/commands/wait/), [`ROLE`](https://redis.io/commands/role/), [`LASTSAVE`](https://redis.io/commands/lastsave/), [`READONLY`](https://redis.io/commands/readonly/), [`READWRITE`](https://redis.io/commands/readwrite/), [`ASKING`](https://redis.io/commands/asking/) ([#9208](https://github.com/redis/redis/pull/9208)):
  - `SELECT` and `WAIT` have been recategorized from `@keyspace` to `@connection`  
  - `ROLE`, `LASTSAVE` have been categorized as  `@admin`  and  `@dangerous`
  -  `ASKING`, `READONLY`, `READWRITE` have also been assigned the  `@connection`  category and  removed from `@keyspace`
  - Command categories are explained in [ACL documentation](https://redis.io/docs/management/security/acl/#command-categories)

#### Command introspection, stats, and configuration

- [`COMMAND`](https://redis.io/commands/command/) reply drops `random` and `sort-for-scripts` flags, which are now part of [command tips](https://redis.io/docs/reference/command-tips/) ([#10104](https://github.com/redis/redis/pull/10104)) - The `random` flag was replaced with the `nondeterministic_output` tip; the `sort-for-scripts` flag was replaced by the ` nondeterministic_output_order` tip

- [`INFO`](https://redis.io/commands/info/) `commandstats` now shows the stats per sub-command ([#9504](https://github.com/redis/redis/pull/9504))
For example, while previous versions would provide a single entry for all command usage, in Redis 7, each sub command is reported separately:

  - Redis 6.2:
  
    ```
    cmdstat_acl:calls=4,usec=279,usec_per_call=69.75,rejected_calls=0,failed_calls=2
    ```

  - Redis 7:
  
    ```  
    cmdstat_acl|list:calls=1,usec=4994,usec_per_call=4994.00,rejected_calls=0,failed_calls=0
    cmdstat_acl|setuser:calls=2,usec=16409,usec_per_call=8204.50,rejected_calls=0,failed_calls=0
    cmdstat_acl|deluser:calls=1,usec=774,usec_per_call=774.00,rejected_calls=0,failed_calls=0
    cmdstat_acl|getuser:calls=1,usec=6044,usec_per_call=6044.00,rejected_calls=0,failed_calls=0
    ```

- [`CONFIG REWRITE`](https://redis.io/commands/config-rewrite/), [`CONFIG RESETSTAT`](https://redis.io/commands/config-resetstat/), and most [`CONFIG SET`](https://redis.io/commands/config-set/) commands are now allowed during loading ([#9878](https://github.com/redis/redis/pull/9878))
