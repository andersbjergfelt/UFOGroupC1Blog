#Blog entry
## Abstract

### State the problem
**NoSQL injection attacks potential impacts are greater than traditional SQL injection.**

### Say why it is interesting
**NoSQL injections enable an attacker to inject code into the query that would be executed by the database. It can result in data loss or
corruption.**

### Say what your solution achieves
**However, NoSQL injection is easy to prevent by taking precaution.**

### Say what follows from your solution.
**Addressing this precaution will make your site invulnerable to one of the most dangerous security risks.**  

Choosing a NoSQL database does not mean that your website is invulnerable to injections.
This blog entry briefly discuss and shows how a NoSQL database can be vulnerable to NoSQL injections and how to prevent them. We will focus on MongoDB as [its the most popular NoSQL database at the moment](https://db-engines.com/en/ranking). Concepts described in this blog entry applies to other NoSQL databases too.

## Injections in general

Most of us are befriended with SQL injections. It is a well-known attack and happens when an attacker injects code into the query that would be executed by the database.
It is possible because a potential unsafe string concatenation when creating a query. NoSQL databases does not use SQL language for queries.

## How much damage can an injection cause?
An injection attack is considered number one most critical web application security risk by OWASP.
(*The OWASP Top 10 is a powerful awareness document for web application security. It represents a broad consensus about the most critical security risks to web applications.*)
It allow attackers to spoof identity, tamper with existing data, allow the complete disclosure of all data on the system, destroy the data or make it otherwise unavailable.


## How does MongoDB address injection?

MongoDB represents queries as [BSON](https://docs.mongodb.com/manual/reference/glossary/#term-bson) objects, not as a string.
Most client libraries available for MongoDB would provide a convenient and injection free, process to build these objects.

### How is an injection still possible then?

See, some MongoDB operations allow you to run arbitrary JavaScript expressions directly on the server. The following operations are:

* $where
* mapReduce
* group

In these cases you must take care to prevent users from submitting malicious JavaScript.
MongoDB further suggests that if you need to pass user-supplied values, [you may want to escape these values.](https://docs.mongodb.com/manual/faq/fundamentals/#how-does-mongodb-address-sql-or-query-injection)

### Testing for NoSQL injection vulnerabilities in MongoDB

As stated earlier MongoDB expects BSON objects. It does not prevent that it is possible to query unserialized JSON and JavaScript expressions in alternative query parameters.
The **$where** operator is the most commonly used API call that allows arbitrary JavaScript input.  
The $where operator is commonly used as a filter.
If an attacker were able to manipulate the data passed into the **$where** operator and could include arbitrary JavaScript to be evaluated.

```javascript
$where: "this.post === '" + req.body.post + "'"
```
The attack could be the string '\'; return \'\' == \'' and the where clause would be evaluated to this.name === ''; return '' == '', that results in returning all users instead of only those who matches the clause.

Another one is
```javascript
 $where: "someID > " + req.body.someID
```
In this case if the input string is '0; return true'. Your where clause will be evaluated as someID > 0; return true and all users would be returned.

Or you could receive '0; while(true){}' as input and suffer a DoS attack.

And another well known:

You receive the following request:

```javascript
{
    "username": {"$ne": null},
    "password": {"$ne": "null"}
}
```
As **$ne** is the not equal operator, this request would return the first user (possibly an admin) without knowing its name or password.

```javascript
{
    "username": "admin",
    "password": {"$gt": ""}
}
```
In MongoDB, **$gt** selects those documents where the value of the field is greater than (i.e. >) the specified value. Thus above statement compares password in database with empty string for greatness, which returns true.

### How to prevent an injection?

NoSQL injections are easy to prevent by taking this precaution:

* Validate inputs to detect malicious values. For NoSQL databases, also validate input types against expected types

* [Disable server side JavaScript completely via –-noscripting.](https://docs.mongodb.com/manual/faq/fundamentals/#how-does-mongodb-address-sql-or-query-injection) The operations, $where, mapReduce and group will become unusable.
