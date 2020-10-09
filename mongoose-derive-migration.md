# Introduction
This document is a comparison between the **Mongoose** framework (another popular ODM) and **DeriveJS**, showing the benifts, differences, down-sides and up-sides of both.
It can also be used as a "migration guide" for people used to Mongoose and are considering to move to DeriveJS.
It basically follows Mongoose's features as they are presented in their documentation, and compare it to equalivent features in Derive.

## Schemas and Models
A schema is the definition of available data properties in a data model (in derive it is referred to as "model definition"). In both frameworks you use a JS object literal to
describe the schema. Moongose enforce the usage of **strictly typed** data types (you need to define the data type for each property) - while Derive does not enforce data types.
You may use "default values" in the schema definition in Derive, and they can also be treated as "data type hints" (e.g. a default value of an empty string hints a string value, 
a default numerical hints a number value, and so on...)

### Model instances
In Mongoose, you first define a schema and create an instance of a `Schema` object, and then create a model, by passing a collection name and the schema object -- then you can create
data instances from the model.
In Derive this is done in a single step: You use a `Model` function, passing it a model definition (a "schema"), and a name - you get back a class (connected to the DB, 
this is what Mongoose calls "documents" (as to make them equaivelent to Mongo's documents), which you can then create instances of.

### Ids
Both framework automatically save documents with an `_id`_ property (of type `ObjectID`). Derive does not allow overrding the default `_id` property, and doing so by adding it to 
the model definition may yield unexpected results. If you need to use your own `id`, define a different id property (e.g. `_ID`).

### Instance methods
In Mongoose you can add "user-defined" methods by adding them to the `methods` property of a schema.
In Derive there are two ways: you can directly define instance methods on the model definition, or you can extend a Model class, and define new methods on it.

Example 1 - defining a method in the model defintion:
```javascript
   const Animal = Model({
       name:"", 
       type:"", 
       findSimilarTypes() { 
           return Animal.getAll({type: this.type}); 
        }
    }, "Animal");

    let dog = new Animal();
    dog.findSimilarTypes().then(dogs=> {
        // Got all "dogs" here.
    });
```

Example 2 - defining a method in a subclass:
```javascript
   const Animal = class extends Model({ name:"", type:"" }, "Animal") {
       findSimilarTypes() {
           return Animal.getAll({type: 'dog'});
       }
   }

    let dog = new Animal();
    dog.type = "dog";
    dog.findSimilarTypes().then(dogs=> {
        // Got all "dogs" here in an array.
        console.log (dogs);
    });
```
### Static methods
In Mongoose you define statics by extending the statics object of schemas.
In Derive, you can define static methods by simply defining them on a subclass of a Model class. e.g.:
```javascript
   const Animal = class extends Model({ name:"", type:"" }, "Animal") {
        static findByName(name) {
            return this.get({ name: new RegExp(name, 'i') });
        }
    }

   Animal.findByName("vincent").then(vincent=> {
        console.log (vincent);
   });
```
### Indexes
In Mongoose you define indexes by using an object literal with `{index: true}` as the value of a data property.
In Derive you use special "modifier" characters in the name of the property. Use an underscore (`_`) at the begining of a property name, to define it as an index:

```javascript
const Animal = Model({
  name: "",
  _type: ""
}, "Animal");
```

Note the name of the property will be `_type` (*including* the underscore).
Whenever you declare a model, Derive will automatically check for the existence of the defined indexes in the collection, and create them if neccesary.

### Meta properties ("Secret" properties), known in Mongoose as "virtuals".
Meta properties are properties that their values won't be persisted on the DB, and are only used locally.
In Mongoose you use the virtuals method of a schema to define a meta property,
In Derive you use the special modifier character - the dollar sign ('$') at the beginning of the name of the property:

```javascript
   const Person = class extends Model({
        name: { first: "", last: "" },
        // will contain full name
        $fullName: "",
        //fullName() { return this.name.first + ' ' + this.name.last; }
    }, "Person") {
        
        constructor() {
            super();
            this.$fullName = this.name.first + ' ' + this.name.last;
        }
    }
```

Note, in this example, you can acheive the same result, in a less verbose way, with a model function:
```javascript
   const Person = Model({
        name: { first: "", last: "" },
        fullName() { return this.name.first + ' ' + this.name.last; }
   }, "Person");
```

The difference being that in the second example, a function is used (and should be called), in the first one a property is used.

### Aliases
There is no concept of "aliases" in Derive, but the same principle can easily be achieved using model functions.

### Connections
Derive connects to the db upon triggering the `Model` module:
```javascript
const derive = require('derivejs');
derive.Model
({
    dbUrl: "mongodb://localhost:27017/",
    dbName: "mydatabase",
})
.then(
    Model=> {
        // `Model` is a function here.
    }
);
```

### Options
Certain configurable options can be passed in an object literal as an argument, the first time the `Model` module is initialized for a collection. See the documentation 
for a full list of options.

### Error handling
Every data class instance has built-in functions for error handling (that can be overrided), these are the `_error()` function and `_isDuplicate()` (a function that
will be called whenever trying to update or insert an object with a duplicate value for a unique index.

### Constructing
To create new data object instances, you simply call the native `new` operator, Derive will take care of the rest -- including inserting an equavilent document into the DB:
`let person = new Person();`

### Querying
Data classes in Derive have several built in functions for querying data from the collection. For a full list see [the documentation](https://www.npmjs.com/package/derivejs#gettingrestoringretrieving-objects-from-the-database).

### Deleting
In Derive ou can use the `clear()` static method with a query to delete documents.

### Updating
To update a data object instance, you use the normal assignment operator (`=`). E.g.:
`person.name = "Jean Luc Ricard"` . Derive will also make sure that the property in the equalivent document will be updated in the DB.

### Validating
There is no explicit validation mechanism in Derive, but it can be achieved in several ways:

Using instance methods:
```javascript
  const Person = Model({
    name: "",
    _email: "",
    setEmail(email) {
        // Perform email validation here.
        this._email = email;
    }
  }, "Person");

Another option is to use the built-in `$Listen` array to listen for changes to a property, and perform validation in the `changed` function. E.g.:
```javascript
   const Person = Model({
        name: "",
        _email: "",
        $Listen: ["_email"],
        changed(property, newValue, oldValue) {
            switch (property) {
                case "_email":
                    // Validate email here
                    console.log (this._email);
                    // If validation fails, set to null or empty string
                    this._email = null;
                    // and throw an error...
            }

        }
   }, "Person");
```

### Overwriting
You may simply overwrite data objects using the normal assignment (`=`) operator, and Derive will make sure to update them in the DB as well.

### Nested documents, and data objects within other data objects.
Derive supports "nesting" just as Mongo supports it, and it also allows you to save a Derive object within another object (effectively saving a [DBRef](https://docs.mongodb.com/manual/reference/database-references/#dbrefs) 
in the DB. E.g.:
```javascript
const Spaceship = Model({
    _name$: "",
    TYPE: "",
    crew: [],
    addCrew: function(crewMember) {
        this.crew.push(crewMember);
    }
},"Spaceship");
const CrewMember = Model({
    _name: "",
    rank: null,
    role:  null
}, "CrewMemeber");

var ricard = new CrewMember("Ricard");
ricard.rank = "captain";
ricard.role = "captain";

var thebeyond = await Spaceship.get("The Beyond");
thebeyond.addCrew(ricard);
```

### Resolving DBRefs ("subdocuments")
In Derive you can use the `get` function to resolve, or dereference a DBRef. Continuing the previous example:
var thebeyond = await Spaceship.get("The Beyond");
var ricard    = await thebeyond.crew[0];

### Mixins (Plugins).
Derivee enables the creation of "mixins", via the built-in `use` function, which you can pass an object literal with functions in it - and those functions 
will be available to all model instances of that data class. See [the documentation](https://www.npmjs.com/package/derivejs#mixins) for more details.
