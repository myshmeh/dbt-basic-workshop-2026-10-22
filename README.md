Welcome to your new dbt project!

### Using the starter project

Try running the following commands:
- dbt run
- dbt test


### Resetting your development schema

To start over, drop and recreate your target schema (in the Studio IDE, your own `dbt_<name>` schema):

```
dbt run-operation drop_target_schema
```

This deletes every table and view in that schema. The macro is in `macros/drop_target_schema.sql`.


### Resources:
- Learn more about dbt [in the docs](https://docs.getdbt.com/docs/introduction)
- Check out [Discourse](https://discourse.getdbt.com/) for commonly asked questions and answers
- Join the [dbt community](https://getdbt.com/community) to learn from other analytics engineers
- Find [dbt events](https://events.getdbt.com) near you
- Check out [the blog](https://blog.getdbt.com/) for the latest news on dbt's development and best practices
