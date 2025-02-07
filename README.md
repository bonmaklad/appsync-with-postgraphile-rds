# Serverless auto-generated GraphQL API with AWS AppSync and PostGraphile



KEY THING: npm i --legacy-peer-deps then npm run update by triggering this lambda function: AppSyncWithPostgraphileStack-providerFnAF7CD54C-30gAFLby1xIJ




![A diagram of the architecture solution Overview](./overview.png "Solution Overview")

This repo provides a CDK-based solution that allows you to create an [AWS AppSync](https://aws.amazon.com/appsync/) API from a defined Postgres database in AWS RDS.

See this blog [post](https://aws.amazon.com/blogs/mobile/creating-serverless-graphql-apis-from-rds-databases-with-aws-appsync-and-postgraphile/) for additional information.

This solution  leverages [PostGraphile](https://www.graphile.org/postgraphile/) to automatically generate an AppSync compliant schema from PostgreSQL tables, and uses Lambda functions to resolve GraphQL queries against a PostgreSQL database in [Amazon RDS](https://aws.amazon.com/rds/). The solution is serverless, and can be deployed in a few clicks. It uses the [AWS CDK](https://aws.amazon.com/cdk/), does not require writing any code, supports subscriptions, and works with any PostgreSQL database like [Amazon RDS for PostgreSQL](https://aws.amazon.com/rds/postgresql/) and [Amazon Aurora PostgreSQL](https://aws.amazon.com/rds/aurora/features/).

- [Serverless auto-generated GraphQL API with AWS AppSync and PostGraphile](#serverless-auto-generated-graphql-api-with-aws-appsync-and-postgraphile)
  - [Solution Overview](#solution-overview)
  - [Requirements](#requirements)
  - [Deploying a VPC with RDS (optional)](#deploying-a-vpc-with-rds-optional)
    - [Deploy the VPC](#deploy-the-vpc)
    - [Loading the database (optional)](#loading-the-database-optional)
  - [The solution](#the-solution)
    - [Solution Requirements](#solution-requirements)
    - [Deploy the solution](#deploy-the-solution)
    - [Configuration](#configuration)
      - [Passing settings](#passing-settings)
      - [Updating after a database schema change](#updating-after-a-database-schema-change)
  - [Cleaning up up the solution](#cleaning-up-up-the-solution)

## Solution Overview

0. Start by deploying the CDK-based solution. The solution creates an AppSync API with a datasource that uses the resolver Lambda function, and an AppSync function that uses that datasource.
1. Once the solution is deployed, a user runs the `provider` function to analyze the RDS PostgreSQL database and generate the GraphQL schema.
2. The `provider` function retrieves schema information from RDS database using Postgaphile
3. The `provider` updates the Layer function attached to the `resolver` Lambda function and updates the AppSync API. It updates the schema, and properly sets up the queries, mutations, and subscriptions. Note that a user can repeat step 1 at any time (e.g.: after a database schema change) to update the AppSync API definition.
4. The AppSync API is now ready to process requests. A GraphQL request is made.
5. AppSync authorizes the request using the configured Authorization Mode (API KEY, Cognito User Pool, etc...)
6. AppSync resolves the request by calling the attached Direct Lambda Resolver. The identity of the user is included in the request to the `resolver` Lambda function
7. The Lambda function resolves the query using the PostGraphile schema and RDS database

For more information about the solution and a detailed walk-through, please see the related [blog](http://todo).

## Requirements

- [Node.js ≥ 14.15.0](https://nodejs.org/download/release/latest-v14.x/)
- [Git](https://git-scm.com/downloads)

Clone this repository and install dependencies:

```sh
git clone https://github.com/aws-samples/appsync-with-postgraphile-rds.git
cd appsync-with-postgraphile-rds
npm install
```

## Deploying a VPC with RDS (optional)

If you do not have an existing PostgreSQL RDS, the **vpc-with-pg** CDK app will deploy a VPC with public and private subnets with NAT Gateway and provision an RDS instance into the private subnet.

### Deploy the VPC

```sh
## from the top of `appsync-with-postgraphile-rds` directory
cd ./vpc-with-pg
npm run cdk bootstrap # make sur CDK has been bootstrap (optional)
npm run deploy
```

### Loading the database (optional)

If you do not have an existing database schema and data, you can leverage the provided [schema](vpc-with-pg/lib/layers/pg-dbschema-layer/lib/dbschema.sql) to get started. You can load the schema and some data by using the [`dbschema.ts`](vpc-with-pg/lib/functions/dbschema.ts) lambda function that was deployed in the previous step.

**Note**: this lambda function also takes care of defining a database user `lambda_runner` (a user with restricted privileges) that will be used to execute all of our queries against the database.

The schema defines a `Person` and `Post` table inside a database called `forum_demo_with_appsync`

```sh
# in the `vpc-with-pg` directory
npm run load
```

## The solution

Deploy the solution into an existing vpc with RDS, or after deploying **vpc-with-pg**.

### Solution Requirements

To get started, you need the following to enable connections to our database:

- an RDS Postgres database
- an RDS Proxy associated with our RDS Postgres database
  - we use [AWS Identity and Access Management (IAM) authentication for databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy.html)
  - and securely store credentials in AWS Secrets Manager.
- at least one private subnet with a NAT that your AWS Lambda function ENIs will be deployed into, and a [VPC Security Group](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)
  - this security group must be an allowed source of traffic for your RDS Proxy security group

You will also need to know the following information about our Postgres database:

- database to connect to
- schema(s) of interest (containing our tables and functions)
- username/role to use to execute queries. This role should have the scoped-down privileges required to access the schema(s). See this [AWS post](https://aws.amazon.com/blogs/database/overview-of-security-best-practices-for-amazon-rds-for-postgresql-and-amazon-aurora-postgresql-compatible-edition/) for more details on security best practices for Amazon RDS for PostgreSQL. The `provider` uses the `postgres` role for configuration. The `resolver` uses your provided username/role to run queries.

### Deploy the solution

Use the `deploy` script to deploy the CDK solution. The script uses values from **vpc-with-pg**'s `output.json` to configure the stacks context environment and variables.

```sh
## from the top of `appsync-with-postgraphile-rds` directory
cd ./appsync-with-postgraphile

REGION="<region>" # the region your resources are running in, with at least one private subnet with NAT
RDSPROXYNAME="<rds-proxy-name>" # the rds proxy name
SECURITYGROUPID="<security-group-id>" # the security group to assign your Lambda Function ENI
USERNAME="<username>" # username/role that will be used to execute SQL queries on database
DATABASE="<database>" # database to connect to
SCHEMAS="<schemas>" # schemas to access. e.g: schema1,schema2,schema3

npm run deploy -- --region $REGION --proxy $RDSPROXYNAME --sg $SECURITYGROUPID --username $USERNAME --database $DATABASE --schemas $SCHEMAS
```

Note: If you deployed **vpc-with-pg** along with the demo data provided, you can simply run the `deploy-demo` npm command (which will use the values from the demo vpc stack):

```bash
## from the top of `appsync-with-postgraphile-rds` directory
cd ./appsync-with-postgraphile
npm run deploy-demo
```

Make note of the outputs.

After deployment, run the `update` script to update your API and create your schema cache layer

```bash
# in the `appsync-with-postgraphile` directory
npm run update
```

You can visit your API query editor in the AppSync console by following the link specified by the output value `AppSyncWithPostgraphileStack.QueryEditorURL`.

Done.

### Configuration

#### Passing settings

By default, the solution passes the caller's [identity](https://docs.aws.amazon.com/en_us/appsync/latest/devguide/resolver-context-reference.html#aws-appsync-resolver-context-reference-identity) to [`pgSettings`](https://www.graphile.org/postgraphile/usage-library/#pgsettings-function). You can pass additional data by setting your own `pgSettings` values in your AppSync pipeline resolver. To do this, attach your own Appsync function to your resolver and add your `pgSettings` object to the stash. Note that the `PG_CONNECTOR_FN` function must be the last function executed. Here's an example.

Mapping template:

```vtl
$util.quiet($ctx.stash.put("pgSettings", {"some": "value", "domainName": $ctx.request.domainName, "nested": {"sub" : "lower level setting"}}))
{
  "payload": {}
}
```

Response template:

```vtl
{}
```

The solution automatically flattens objects and adds the prefix `appsync.` to all your object keys. The example above produces the following settings:

```json
{
  "appsync.domainName": "my.domain.com",
  "appsync.some": "value",
  "appsync.nested_sub": "lower level setting"
}
```

#### Updating after a database schema change

You can update you GraphQL schema at any time by calling `npm run update`. This will update your schema and set up any new resolvers. Existing resolvers will not be modified.

## Cleaning up up the solution

When you are done with the solution, you can delete your resources.

```sh
# in the `appsync-with-postgraphile` directory
npm run destroy
```

If needed, clean up the demo resources as well

```sh
# in the `vpc-with-pg` directory
npm run destroy
```
