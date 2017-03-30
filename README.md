# Lower Query Language

LQL is a simplified language for selecting records in a relational database. Valid LQL will generate valid SQL which you can safely execute without any risks of modifying or freezing your database.

## Purpose

LQL is intended to be used by non-technical users of applications with complex databases. For example, maybe you have a CRM with 100+ tables and you want to provide an advanced search where users can compose/run/save LQL queries.

## Status

LQL is currently in **concept phase**.

## How it works

An LQL processor does the following

1. Takes input
    * LQL code
    * Database schema (so that it knows about foreign keys)
    * *(optionally) Additional schema hints (like pseudo foreign keys)*
    * *(optionally) Other global settings*
1. Produces output
    * SQL

## Intentional limitations

* It only produces SELECT queries.
* With some exceptions for relative dates, it does not allow expressions to be used as values.
* Results are always grouped by the primary key of one and only one table.

## Example


```text
                                  /* These are comments */
                                  /* Most white space doesn't matter */
                                  /* First word is the table to select FROM */
person {                          /* Curly braces enclose "AND" conditions */

  [                               /* Square braces enclose "OR" conditions */
    first_name = 'Liz'
    nick_name = 'Liz'
  ]

  favorite_color.name = 'blue'    /* Many-to-one join through FK
                                     ("favorite_color" is another table) */

  +address {                      /* One-to-many "inclusion" join
                                   * (At least one associated entity must exist
                                   * which matches the specified criteria) */

    state.code = ['MA' 'NY']      /* IN operator */
    tz.utc_offset >< (-6 -4)      /* BETWEEN operator */
  }

  -phone {                        /* One-to-many "exclusion" join through FK
                                   * (There must be zero associated entities
                                   * which matches the specified criteria) */

    number ~ '617%'               /* LIKE operator */
  }

  is_deleted != 1                 /* not equal to */

  birth_date <= @now - 18 year    /* relative dates */

}
                                  /* Here are the columns to return */
id 
first_name 
last_name 
phone.number
supervisor.first_name 
supervisor.last_name
```

The above LQL *(along with a supplied DB schema that you are expected to infer from this example)* will produce the following SQL:

```sql
SELECT
  person.id,
  person.first_name,
  person.last_name,
  group_concat(distinct phone.number separator ", "),
  supervisor.first_name,
  supervisor.last_name
FROM person
JOIN favorite_color ON 
  favorite_color.id = person.favorite_color_id AND
  favorite_color.name = "blue"
JOIN address ON
  address.person_id = person.id
JOIN state ON
  state.id = address.state_id AND
  state.code in ("MA", "NY")
JOIN tz ON
  tz.id = address.tz_id AND
  tz.utc_offset BETWEEN -6 AND -4
JOIN person supervisor ON
  supervisor.id = person.supervisor_id
LEFT JOIN phone ON
  phone.person_id = person.id
LEFT JOIN phone phone_exclusion ON
  phone_exclusion.person_id = person.id AND
  phone_exclusion.number LIKE "617%"
WHERE
  ( 
    person.first_name = "Liz" OR
    person.nick_name = "Liz"
  ) AND
  person.is_deleted IS NOT NULL AND
  person.birth_date <= now() - INTERVAL 18 YEAR AND
  phone_exclusion.id IS NULL
GROUP BY person.id;
```


