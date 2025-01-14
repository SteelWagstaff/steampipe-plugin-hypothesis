# Table: hypothesis_profile

If you [authenticate](../index.md), this table reports your Hypothesis username, display name, authority, and groups.

## Examples

### Get your username, display name, and authority

```
select
  "user",
  display_name,
  authority
from
  hypothesis_profile
```

### Get the names and ids of groups you belong to

```
select
  jsonb_array_elements(groups)
from
  hypothesis_profile
```

### Among the most recent 500 notes, find those in your private groups (method 1)

```
with groups as (
  select
    jsonb_array_elements(groups) as group_info
  from
    hypothesis_profile
),
annos as (
  select
    *
  from
    hypothesis_search
  where 
    query = 'limit=500'
  order by
    created desc
)
select
  'https://hypothes.is/a/' || a.id as link,
  a."group",
  g.group_info ->> 'name' as name,
  a."user",
  a.created,
  a.title,
  a.uri
from 
  groups g
join
  annos a
on 
  g.group_info ->> 'id' = a."group"
where
  g.group_info ->> 'public' != 'true'
```    

### Among the most recent 500 notes, find those in your private groups (method 2)

**NOTE** It can be helpful to turn chunks of SQL code into Postgres functions. Here we define, and then use, `hypothesis_is_private_group`, a function that checks if a `groupid` is private. This function makes method 2 simpler than method 1. And you can use the function anywhere a `groupid` appears. See [Postgres functional style](https://blog.jonudell.net/2021/08/21/postgres-functional-style/) for details.

#### Create the function `hypothesis_is_private_group`

**NOTE** Steampipe plugins put their tables into Postgres schemas (namespaces) that match the names of the plugins. So tables in this plugin are actually `hypothesis.hypothesis_search` and `hypothesis.hypothesis_profile`. The examples here don't qualify table names with schemas because if there is no confict with another schema it's unnecessary. When you create functions, though, they live in Postgres' global namespace. Nothing requires you to prepend a schema-like prefix to function names, but it's probably a good idea to do that in order to clarify which plugins they're intended to work with.

```
create function hypothesis_is_private_group (groupid text) returns boolean as $$
  declare is_private boolean;
  begin
    with groups as (
    select
        jsonb_array_elements(groups) as group_info
    from
        hypothesis_profile
    )
    select
      g.group_info ->> 'public' != 'true'
    from
      groups g
    where 
      g.group_info ->> 'id' = groupid
    into
      is_private;
    return is_private;
  end;
$$ language plpgsql;
```

#### Use `hypothesis_is_private_group` 

```
select
  'https://hypothes.is/a/' || id as link,
  "group",
  "user",
  created,
  title,
  uri
from 
  hypothesis_search
where 
  query = 'limit=500'
  and hypothesis_is_private_group("group")
```    




