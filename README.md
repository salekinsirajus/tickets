# Ticket Interface
A RESTful API backend that stores JIRA tickets. Implemented with the 
[serverless] (https://serverless.com) framework. 

## Tools Used
- AWS Lambda 
- DynamoDB 
- Python3.6
- AWS API Gateway (with Lambda-Proxy integration)
- Serverless Framework

## How to use
The easiest way to communicate with this API is to use `curl`.
1. List the tickets in the table:
```
curl -X GET https://qoofdupzk8.execute-api.us-west-2.amazonaws.com/dev/tickets
```
2. Get a spcific ticket:
```
https://qoofdupzk8.execute-api.us-west-2.amazonaws.com/dev/tickets/{id}

example:
curl -X GET https://qoofdupzk8.execute-api.us-west-2.amazonaws.com/dev/tickets/1544473087987

# outputs:
#{
#    "completed": "2018-10-10T20:15:07.958Z", 
#    "created": "2018-12-10T20:18:07.987Z",
#    "priority": "major", 
#    "summary": "Does this go to db", 
#    "id": "1544473087987",
#    "duration": 61
#}
```

3. Create a ticket 
```
curl -X POST https://qoofdupzk8.execute-api.us-west-2.amazonaws.com/dev/tickets
-H "Content-Type: application/json" -d '{"completed":"2018-10-19T20:15:07.958Z",
"priority":"major", "summary":"bug or a feature",
"created":"2018-11-04T07:18:03.764Z"}' 

```

#### Caveats: 
* You can just paste the url on a broswer for the get methods (1 & 2), and it
will return the same thing. Howeverm for creating (3), it is a good idea to use
`curl`
* Make sure to pass all the fields in the json object (when calling the create
method), even though they might contain an empty string. DynamoDB cannot accept
a null value, and an empty field won't be recorded (consistent with DynamoDB
being a schemaless database).

---
## Design Plan
At the heart, the problem is of storing and reading data. So we are going to
figure out how to store the simplest data point using AWS services. After that,
we will add more features, convenient helpers, etc.

For this project, we will implement a REST API service for a uniform interface.
In respect to the serverless framework (and architecture):

Whenever a user call the using the API endpoint a POST request, the lambda
`create` function should store (validate either by the gateway or the lambda) the
data served through that request in DynamoDB. Similarly, when a user invokes
with a GET request, the `read` function will execute and send a response.

- Functions: 
    * create
    * read
    * scan (for browsing)
    * update (mark finished)
    * delete (extra feature)

- Event: An AWS API Gateway HTTP endpoint request (a GET/POST request)
- Resource: An AWS DyanomDB table

Database schema:
* DynamoDB needs a primary key. It could be a hash, or a string. 
Three options are available: 
    - use the partition key as primary key
    - use the partition and sort key as composite primary key
    - create a ticket_id. (we need to scan in that case for reading)
If the I choose to go with the latter, sort key = priority. 

* Other fields it should have: 
    - summary: string
    - created_at: timestamp/string (how to use
      [here](https://stackoverflow.com/questions/40561484/what-data-type-should-be-use-for-timestamp-in-dynamodb)
    - description: string
    - priority: string
    - completion_time: timestamp/string/float

I need to figure out a way to retrieve an item. Possible options:
1. Use the created_at timestamp (obvious, *can* be tedius)
2. Use the sort key to get a range of items (is it possible?)
3. Look at LSI

### Design Decisions and Changes:
1. I finally decided to derive a partition (not composite) key for DynamoDB
using the converted epoch values of the `created` attribute of the ticket. The
`id`' numbers have high cardinality, so in case the backend is scaled up, we are
not going to have uneven traffic on one partition.

Reasons for not using the other alternatives:
    - A datetimp string: it contains unsafe characters for the browser, although it
    could have been useful for the case of retrieving a single item from the data 
    off the ticket.
    - Using a uuid generated key: if it's going to be a large number, let it be
    meaningful to some degree.


### To Do
- [x] Add functions in `serverless.yml`
- [x] Provide appropriate permissions for DynamoDB
- [x] Set up the DynamoDB table(resources)
- [x] Provide appropriate IAM access for functions. Come up with a convenient
way to call the create method
- [x] Validate data before putting into DynamoDB
- [ ] `create` function: check whether the item is duplicate before replacing
- [ ] `get` function: solve the KeyError issue
- [ ] Write unit tests and integration tests

### Progres
- [x] Deployed the hello world app
- [x] Setting up the DynamoDB table (wip)
- [x] Deployed a test GET api method
- [x] Implemented a hardcoded create function
- [x] Implemented a scan function


### Learning from this exercise
- Serverless architecture can be a great model for reducing operationals cost,
and depending on the organization, the server management could be completely
gone.
- DynamoDB's strength lies in its distributive nature. So it is crucial that even
distributions across the partitions are ensured by choosing a partition key
with high cardinality.
- The serverless framework  reduces a lot of pain points
that comes with deploying a non-trivial app in a serverless model. For AWS,
this framework produces a CloudFormation template and deploy resources in a
systematic way, which I found to be very helpful during the development.
- AWS Lambda is a great choice for orgs that uses functional programming as
their primary paradigm of developemnt. AWS encourage Lambda functions 
to be programmed in a stateless manner.
- There is an odd error with CloudFormation that is if you change some
attributename in DynamoDB, you need to renamce the table name, and the rename
back, for the case when you just want to replace the name of the column. I have
a noticed a lot of such workarounds (and used them) during the development. It
will take some time before some of this significant issues are taken care of.
