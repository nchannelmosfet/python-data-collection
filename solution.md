# Solution to the Problem:

## Two Tables

At the first glance, I tried to understand the relationship between these two tables.

> Typically, a table may look like this example:
>
> | Year              | Crop Type                                                     | Tillage Depth                                                        | Comments |
> | ----------------- | ------------------------------------------------------------- | -------------------------------------------------------------------- | -------- |
> | Four digit number | Constrained Picklist (might be [corn, wheat, barley, hops...] | Constrained Float ie must be `0 <= x < 10`, can be optionally filled | String   |
>
> However, due to the flexibility of our offering, this is not the only data we want to collect. We may want to collect something like:
>
> | Year              | Tilled? | External Account ID    | Tillage Depth                    |
> | ----------------- | ------- | ---------------------- | -------------------------------- |
> | Four digit number | Bool    | Regex Validated String | Slider Control mapped to a float |

They seemed to be similar and there is to be no obviously reason to split this crop "entity" into two tables.
There is no foreign key that would allow the joining of these two tables.
I will make an assumption that they can be combined into 1 table like this. Let's call it the `Crop` table.

| External Account ID    | Year              | Crop Type                                                     | Tilled? | Tillage Depth                                                        | Comments |
| ---------------------- | ----------------- | ------------------------------------------------------------- | ------- | -------------------------------------------------------------------- | -------- |
| Regex Validated String | Four digit number | Constrained Picklist (might be [corn, wheat, barley, hops...] | Bool    | Constrained Float ie must be `0 <= x < 10`, can be optionally filled | String   |

In this combined table, I will assume that the `External Account ID` is a foreign key of `User` id in code.
A user / farmer may plant multiple types of crops across many years. So, the relationship is 1-to-many between `User` and `Crop`.

There are cases to be made to whether or not enforce referential integrity.
If enforced the a crop entity cannot exist without being assigned to a user.
This improves data integrity if this behavior is desired, but it could be a performance hit to database.
Without clear requirement, I will elect not to enforce.

## Get only the data you need

### GraphQL Approach

We can implement a solution for this requirement by using GraphQL. With GraphQL, the client specifies what fields are needed in a query then the server will return data for the specifies fields, nothing less and nothing more.

> By the same token, we may want some other, unique set of columns that are picked from both the examples. You need to design and implement a solution that would let you represent both examples and any combination/permuation of their constituent columns.

For example, a graphql query could look like this:

```graphql
{
    query ($cropId: Int!) {
        getCrop(cropId: $cropId){
            cropId
            userId
            year
            cropType
            tilled
            tillageDepth
            comments
        }
    }
}
```

With graphql variable like:

```json
{ "cropId": 12345 }
```

The client is free to omit 1 or multiple fields in the graphql query and get back only the data of interest.
The output of this query will be an array of crops with specified fields pertaining to a certain user.

### REST Approach

REST API typically returns a fixed number of fields as define by an API contract, i.e. OpenAPI / Swagger docs.
It is possible to enable field selection in a REST API approach by accepting a query parameter called "fields".
The value for this "fields" parameter could be a comma-separated list.

The endpoint could look like this:

```
/crops/{crop_id}?fields=user_id,year,crop_type
```

In the view/route/endpoint function, we can manipulate the SQL result to include only the selected fields.
The downside is that it breaks a consistent API contract and creates friction for system integration.
It is better for the REST API to return the full payload with all the fields to maintain a consistent API contract.
The downstream client who received the full payload can decide what fields to use and what to not use.

### Pagination

This is a topic of pagination. When an array is potentially unbounded from a database query, we should implement pagination to ensure the operation is not too taxing on the database, app server and also the front-end.

> Additionally, as per the screenshot, some configuration must be present to tell the UI how many years of data we wish to collect (hence how many rows to render).

There are several ways to implement pagination. For example with page_size and page_num:

```
/crops?page_size=10&page_num=3
```

page_size determines the number of items in a page while page_num can be used to iterate or traverse all the records.

Another option is to use limit and offset:

```
/crops?limit=10&offset=50
```

On the database level, it is typically limit and offset. It can be implemented as page_size and page_num for convenience to the user.

### SQL Migration

This is a topic of SQL migration.

> In the future we may want to collect other information too; these columns are not fixed.

Typically, adding new columns to a table or adding new fields to an API response is not a breaking change. Deleting columns, changing the type or nullability of the column could be a breaking changing.

It is important to utilize migration instead of create all tables in production.
With migration, each mutation to the database schema is recorded and we can upgrade migration in case of adding new features or downgrade migration in case of critical failure.

Alembic is a popular SQL migration tool for FastAPI and SQLAlchemy which are used in this project.

### Implemented

Please see code for the implementation of the following tasks.

> It is your task to:
>
> - Define and implement a database schema that can store this configuration.
> - Expose a REST API to create/delete these entities.
> - Handle some validation of inputs where sensible.

- Implemented `Crop` model that represents the database schema
- Exposed create and delete endpoints for `Crop`
  - Included get endpoints for single or array of crops with pagination for validation
  - Included all possible response status in Open API documentation
- Handle validation with `sqlmodel` `Field` and `FastAPI` `Query`

### Authentication and Security

This is an important consideration but not implemented in the assignment.
For a user facing application like this one, a modern solution is to use OpenID Connect protocol with an identity provider such as Okta or Azure Active Directory.

We can also stored user passwords as hash in the database, but this not the most modern and secure option.

If this were implemented, then the user can only view crops or other data belong to that user and other user's data.
This is an import security safeguard.
