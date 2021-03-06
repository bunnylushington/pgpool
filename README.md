
[![Build Status](https://travis-ci.org/ostinelli/pgpool.svg?branch=master)](https://travis-ci.org/ostinelli/pgpool)
[![Hex pm](https://img.shields.io/hexpm/v/pgpool.svg)](https://hex.pm/packages/pgpool)

# PGPool
PGPool is a PosgreSQL client that automatically uses connection pools and handles reconnections in case of errors.  PGPool also optimizes all of your statements, by preparing them and caching them for you under the hood.

It uses:

 * [epgsql](https://github.com/epgsql/epgsql) as the PostgreSQL driver.
 * [poolboy](https://github.com/devinus/poolboy) as pooling library.


## Install

### For Elixir
Add it to your deps:

```elixir
defp deps do
  [{:pgpool, "~> 2.1"}]
end
```

Ensure that `pgpool` is started with your application, for example by adding it to the list of your application's `extra_applications`:

```elixir
def application do
  [
    extra_applications: [:logger, :pgpool]
  ]
end
```

### For Erlang
If you're using [rebar3](https://github.com/erlang/rebar3), add `pgpool` as a dependency in your project's `rebar.config` file:

```erlang
{pgpool, {git, "git://github.com/ostinelli/pgpool.git", {tag, "2.1.0"}}}
```

Or, if you're using [Hex.pm](https://hex.pm/) as package manager (with the [rebar3_hex](https://github.com/hexpm/rebar3_hex) plugin):

```erlang
{pgpool, "2.1.0"}
```

Ensure that `pgpool` is started with your application, for example by adding it in your `.app` file to the list of `applications`:

```erlang
{application, my_app, [
    %% ...
    {applications, [
        kernel,
        stdlib,
        sasl,
        pgpool,
        %% ...
    ]},
    %% ...
]}.
```

## Usage
Since `pgpool` is written in Erlang, the example code here below is in Erlang. Thanks to Elixir interoperability, the equivalent code in Elixir is straightforward.


### Specify Databases
Databases can be set in the environment variable `pgpool`. You're probably best off using an application configuration file (in releases, `sys.config`):

```erlang
{pgpool, [
  {databases, [
    {db1_name, [
      {pool, [
        %% poolboy options <https://github.com/devinus/poolboy>
        %% The `name` and `worker_module` options here will be ignored.
        {size, 10},        %% maximum pool size
        {max_overflow, 5}, %% maximum number of additional workers created if pool is empty
        {strategy, lifo}   %% can be lifo or fifo (default is lifo)
      ]},
      {connection, [
        {host, "localhost"},
        {user, "postgres"},
        {pass, ""},
        {options, [
          %% epgsql connect_options() <https://github.com/epgsql/epgsql>
          {port, 5432},
          {ssl, false},
          {database, "db1"}
        ]}
      ]}
    ]},

    {db2_name, [
      {pool, [
        {size, 10},
        {max_overflow, 20},
        {strategy, lifo}
      ]},
      {connection, [
        {host, "localhost"},
        {user, "postgres"},
        {pass, ""},
        {options, [
          {port, 5432},
          {ssl, false},
          {database, "db2"}
        ]}
      ]}
    ]}
  ]}
]}
```

### Custom types

```erlang
-type pgpool_query_option() :: no_wait.
```

`no_wait` makes the query call return immediately if there are no available connections in the pool. This allows you to improve your application's flow control by rejecting external calls if your database is unable to handle the load, thus preventing a system overflow.

### Queries
Please refer to [epgsql 3.4 README](https://github.com/epgsql/epgsql/blob/3.4.0/README.md) for how to perform queries. Currently, PGPool supports the following.

#### Simple Query

```erlang
pgpool:squery(DatabaseName, Sql) ->
  pgpool:squery(DatabaseName, Sql, []).
```

```erlang
pgpool:squery(DatabaseName, Sql) ->
  Result

Types:
  DatabaseName = atom()
  Sql = string() | iodata()
  Options = [pgpool_query_option()]
  Result = {ok, Count} | {ok, Count, Rows} | {error, no_connection | no_available_connections}
  Count = non_neg_integer()
  Rows = (see epgsql for more details)
```

For example:

```erlang
pgpool:squery(db1_name, "SELECT * FROM users;").
```

> Simple queries cannot be optimized by PGPool since they cannot be prepared. If you want to optimize and cache your queries, consider using `equery/3,4` or `batch/2` instead.

#### Extended Query

```erlang
pgpool:equery(DatabaseName, Statement, Params) ->
  pgpool:equery(DatabaseName, Statement, Params, []).
```

```erlang
pgpool:equery(DatabaseName, Statement, Params, Options) ->
  Result

Types:
  DatabaseName = atom()
  Statement = string()
  Params = list()
  Options = [pgpool_query_option()]
  Result = {ok, Count} | {ok, Count, Rows} | {error, no_connection | no_available_connections}
  Count = non_neg_integer()
  Rows = (see epgsql for more details)
```

For example:

```erlang
pgpool:equery(db1_name, "SELECT * FROM users WHERE id = $1;", [3]).
```

> PGPool will prepare your statements and cache them for you, which results in considerable speed improvements. If you use a lot of different statements, consider memory usage because the statements are not garbage collected.

#### Batch Queries
To execute a batch:

```erlang
pgpool:batch(DatabaseName, StatementsWithParams) ->
  pgpool:batch(DatabaseName, StatementsWithParams, Options).
```

```erlang
pgpool:batch(DatabaseName, StatementsWithParams) ->
  Result

Types:
  DatabaseName = atom()
  StatementsWithParams = [{Statement, Params}]
  Statement = string()
  Params = list()
  Options = [pgpool_query_option()]
  Result = [{ok, Count} | {ok, Count, Rows}] | {error, no_connection | no_available_connections}
  Count = non_neg_integer()
  Rows = (see epgsql for more details)
```

For example:

```erlang
S = "INSERT INTO users (name) VALUES ($1);",

[{ok, 1}, {ok, 1}] = pgpool:batch(db1_name, [
  {S, ["Hedy"]},
  {S, ["Roberto"]}
]).
```

> PGPool will prepare your statements and cache them for you, which results in considerable speed improvements. If you use a lot of different statements, consider memory usage because the statements are not garbage collected.


## Contributing
So you want to contribute? That's great! Please follow the guidelines below. It will make it easier to get merged in.

Before implementing a new feature, please submit a ticket to discuss what you intend to do. Your feature might already be in the works, or an alternative implementation might have already been discussed.

Do not commit to master in your fork. Provide a clean branch without merge commits. Every pull request should have its own topic branch. In this way, every additional adjustments to the original pull request might be done easily, and squashed with `git rebase -i`. The updated branch will be visible in the same pull request, so there will be no need to open new pull requests when there are changes to be applied.

Ensure that proper testing is included. To run PGPool tests, you need to create the database `pgpool_test` for user `postgres` with no password, and then simply run from the project's root directory:

```
$ make test
```
