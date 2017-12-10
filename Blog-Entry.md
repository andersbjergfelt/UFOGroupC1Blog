#Blog entry
# Prevent NoSQL injection attacks in MongoDB

### Authors

**Emil Gräs 
&
Anders Bjergfelt**

NoSQL injection attacks potential impacts are greater than traditional SQL injection.

De fleste kender SQL injections. Det er et velkendt angreb der foregår ved, at en ondsindet bruger tilføjer kode ind via et inputfelt, som ender i en database query, der bliver eksekveret.
Man undgår desværre ikke problemet ved at vælge en NoSQL database. Ens website vil stadig være sårbar for de former for angreb. Et lignede angreb mod en NoSQL database kaldes for en NoSQL injektion og har samme fremgangsmåde og konsekvenser som en SQL injektion. De resulterer oftes i, at data forsvinder eller bliver korrupt. Vælger man dog at tage sine forbehold, er NoSQL injections lette at sikre sig i mod. Det kræver blot, at man adresserer disse forbehold og følger dem. Det vil gøre dit website uskadelig for en af de farligste sikkerhedsrisici.  

Dette blogindlæg vil vi kort diskutere og vise, hvordan en NoSQL database kan være sårbar over for NoSQL injektioner og hvordan man kan forhindre dem. Der vil blive fokusere på MongoDB som [den mest populære NoSQL database i øjeblikket] (https://db-engines.com/en/ranking). Begreber som bliver beskrevet i dette blogindlæg er også aktuelle for andre NoSQL databaser.


NoSQL og SQL databer adskiller sig på mange måder fra hinanden. NoSQL databaser har langt færre restriktioner når der fx skal laves skemaer. De bruger heller ikke relationer eller constraints, hvilke i SQL verden er med til at sikre at data er konsistent. "Manglen" på restriktioner er netop en af grundene til hvorfor NoSQL databaser ofte udkonkurerer de mere traditionelle SQL databaser på performance og skalérbarhed. Men det gør dem også mere sårbare overfor angreb. Database injections er i dag stadigvæk et omfattende og alvorligt problem, og det er til trods for at det  oftest ikke kræver meget arbejde at sikre sig imod dem.

Når man laver et databasekald med NoSQL skrives det ofte i det samme sprog som systemet er lavet i. Laver man fx en java applikation, skriver man typisk sine database queries i java. Oftest findes der et hav af APIer til lige netop dette. Disse APIer varierer meget i forhold til hvordan de er lavet og hvad de indeholder. I dag findes der mere end 200 officielle NoSQL databaser (http://nosql-database.org/), og de fleste kan integreres i mange sprog. Da der findes så mange forskellige løsninger kan det være svært at vide hvor sikkert et API er, og om de overhovedet håndtere injections eller andre potentielle sikkerhedsbrister. Det er altså uhyre svært at teste sin applikation for NoSQL injections. Det kræver et godt kendskab til APIet, typisk på et ret teknisk niveau.

## Hvor meget skade kan en injection forårsage?

[OWASP](https://www.owasp.org/index.php/Main_Page) ser et injektion angreb som det mest kritiske webapplikationssikkerhedsrisiko. 

Det tillader ondsindede brugere at manipulere med eksisterende data, tillade fuldstændig offentliggørelse af alle data på systemet, ødelægge data eller gøre det utilgængeligt.

## How does MongoDB address injection?

## Hvordan adresserer MongoDB injection?

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

MongoDB expects BSON objects. It does not prevent that it is possible to query unserialized JSON and JavaScript expressions in alternative query parameters.
The **$where** operator is the most commonly used API call that allows arbitrary JavaScript input.  
The $where operator is commonly used as a filter.

```javascript
db.myCollection.find( { $where: "this.username == this.name" } );
```
If an attacker were able to manipulate the data passed into the **$where** operator and could include arbitrary JavaScript to be evaluated then the attack could be **the string '\'; return \'\' == \''** and the where clause would be evaluated to **this.name == ''; return '' == ''**, that results in returning all users instead of only those who matches the clause.

Another one is
```javascript
db.myCollection.find( { $where: "this.someID > this.anotherID" } );
```
In this case if the input string is '0; return true'. Your where clause will be evaluated as someID > 0; return true and all users would be returned.

Or you could receive **'0; while(true){}'** as input and suffer a DoS attack.

And another well known:

You receive the following request:

```javascript
{
    "username": {"$ne": null},
    "password": {"$ne": "null"}
}
```
As **$ne** is the not equal operator, this request would return the first user without knowing its name or password.

```javascript
{
    "username": "admin",
    "password": {"$gt": ""}
}
```
In MongoDB, **$gt** selects those documents where the value of the field is greater than (i.e. >) the specified value. Thus above statement compares password in database with empty string for greatness, which returns true.

### How to prevent an injection?

NoSQL injections are easy to prevent by taking these precaution:

1. Validate inputs to detect malicious values. For NoSQL databases, also validate input types against expected types

2. [Disable server side JavaScript completely via –-noscripting.](https://docs.mongodb.com/manual/faq/fundamentals/#how-does-mongodb-address-sql-or-query-injection) The operations, $where, mapReduce and group will become unusable.

1. It's important to ensure that user input received and used to build the API call expression do not contains any character that have a special meaning in the target API syntax.

It's also important to not use string concatenation to build API call expression but use the API to create the expression.

Here's a quick way to do it in Java targeting MongoDB

```java
ArrayList<String> specialCharsList = new ArrayList<String>() {{
    add("'");
    add("\"");
    add("\\");
    add(";");
    add("{");
    add("}");
    add("$");
}};
@RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public Post getPost(@PathVariable("id") String id){
		    for (String c : specialCharsList) {
             if (id.contains(c)) {
                 return error;
             }
         }
        return repository.findOne(id);
    }

```
The arraylist contains all chars that have a special meaning in target API. It will prevent arbitrary JavaScript to be evaluated.



