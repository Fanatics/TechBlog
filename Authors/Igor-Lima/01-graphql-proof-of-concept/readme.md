# **Proof of Concept** for two different implementations of GraghQL server

As we already know GraphQL is an alternative to REST endpoints for handling queries and database updates.  And in this tech article I'll try to figure out which programming language, among [Node](https://github.com/igorlima/expressjs-graphql-server) and [Java](https://github.com/igorlima/spark-graphql-server), is way faster to handle GraphQL queries. Also, I will use different tools to measure the perfomance and document the process.

Let's start comparing some code. The idea here is to make sure both implementations handle all requests using the same approach. For instance, the amount of calling to the database and so on. Bellow is the list of the source for each language:

* NodeJS: https://github.com/igorlima/expressjs-graphql-server
* JAVA: https://github.com/igorlima/spark-graphql-server

We tried to keep both the implementations straightforward. 

## Server side code

Let's take a quick look at on how things are implemented in each language. Notice that both languages follow the same approach. I mean the way it creates a GraphQL schema and so forth.

#### __*JAVA*__

To create a GraphQL server in Java, firstly we need to define a [GraphQLObjectType](https://github.com/andimarek/graphql-java/blob/master/src/main/java/graphql/schema/GraphQLObjectType.java). A type of object that will be served through user requests. Here, we have a type `todo` with the following attributes: `id`, `title` and `completed`.

```java
// https://github.com/igorlima/spark-graphql-server/blob/master/src/main/java/com/mycompany/app/TodoSchema.java#L15
public static GraphQLObjectType todoType = newObject()
  .name("todoType")
  .field(newFieldDefinition()
    .type(GraphQLString)
    .name("id")
    .description("Todo id")
    .build())
  .field(newFieldDefinition()
    .type(GraphQLString)
    .name("title")
    .description("Task title")
    .build())
  .field(newFieldDefinition()
    .type(GraphQLBoolean)
    .name("completed")
    .description("Flag to mark if the task is completed")
    .build())
  .build();
```

In addition, we have to define a *query type* by using the same class [GraphQLObjectType](https://github.com/andimarek/graphql-java/blob/master/src/main/java/graphql/schema/GraphQLObjectType.java). The query type below defines a simple type to resolve a list of todos.

```java
// https://github.com/igorlima/spark-graphql-server/blob/master/src/main/java/com/mycompany/app/TodoSchema.java
// lines 34 and 42
static GraphQLObjectType queryType = newObject()
  .name("Todo")
  .field(newFieldDefinition()
    .type(new GraphQLList(todoType))
    .name("todos")
    .description("todo tasks")
    .dataFetcher(TodoData.todoFetcher)
    .build())
  .build();
```

Of course we need to fetch data, and it's done by `TodoData.todoFetcher`, what is an instance of [DataFetcher](https://github.com/andimarek/graphql-java/blob/master/src/main/java/graphql/schema/DataFetcher.java). 

```java
// https://github.com/igorlima/spark-graphql-server/blob/master/src/main/java/com/mycompany/app/TodoData.java
// lines 36 and 49
static DataFetcher todoFetcher = new DataFetcher() {
  @Override
  public Object get(DataFetchingEnvironment environment) {
    List<Map<String, Object>> todos = new ArrayList<Map<String, Object>>();
    FindIterable<Document> iterable = collection.find();
    iterable.forEach(new Block<Document>() {
      @Override
      public void apply(final Document document) {
        todos.add(converter(document));
      }
    });
    return todos;
  }
};
```

Note that the data comes from `collection.find()`. A MongoDB collection. To create a [MongoDB connection in Java](https://docs.mongodb.org/getting-started/java/client/), we need to have a [MongoClientURI](https://api.mongodb.org/java/3.2/com/mongodb/MongoClientURI.html) to instantiate a [MongoClient](https://api.mongodb.org/java/3.2/com/mongodb/MongoClient.html). With that instance of MongoClient, we are ready to get a [MongoDatabase](https://api.mongodb.org/java/3.2/com/mongodb/client/MongoDatabase.html) and finally a [MongoCollection](https://api.mongodb.org/java/3.2/com/mongodb/client/MongoCollection.html).

> **NOTE:** I’m sharing my credentials here. Feel free to use it while you’re learning. After that, create and use your own credential. Thanks.

```java
// https://github.com/igorlima/spark-graphql-server/blob/master/src/main/java/com/mycompany/app/TodoData.java#L23
MongoClientURI uri = new MongoClientURI("mongodb://example:example@candidate.54.mongolayer.com:10775,candidate.57.mongolayer.com:10128/spark-server-with-mongo?replicaSet=set-5647f7c9cd9e2855e00007fb");

MongoClient mongoClient = new MongoClient(uri);
MongoDatabase db = mongoClient.getDatabase("spark-server-with-mongo");
MongoCollection<Document> collection = db.getCollection("todos");
```

Now, we have in place a connection, a query type, and the type of object `todo`. All of that allows us to define our GraphQL Schema in Java:

```java
// https://github.com/igorlima/spark-graphql-server/blob/master/src/main/java/com/mycompany/app/TodoSchema.java#L121
public static GraphQLSchema schema = GraphQLSchema.newSchema()
  .query(queryType)
  .build();
```

#### __*NodeJS*__

In the other side in Node, to create a GraphQL server, firstly we need to define a [GraphQLObjectType](https://github.com/graphql/graphql-js/blob/master/src/type/definition.js#L289). A type of object that will be served through user requests. Here, we have a type `todo` with the following attributes: `id`, `title` and `completed`.

```js
var TodoType = new GraphQLObjectType({
  name: 'todo',
  fields: () => ({
    id: {
      type: GraphQLID,
      description: 'Todo id'
    },
    title: {
      type: GraphQLString,
      description: 'Task title'
    },
    completed: {
      type: GraphQLBoolean,
      description: 'Flag to mark if the task is completed'
    }
  })
})
```

In addition, we have to define a *query type* by using the same object [GraphQLObjectType](https://github.com/andimarek/graphql-java/blob/master/src/main/java/graphql/schema/GraphQLObjectType.java). The query type below defines a simple type to resolve a list of todos.

```js
// https://github.com/igorlima/expressjs-graphql-server/blob/master/schema.js
// lines 81 and 91
var QueryType = new GraphQLObjectType({
  name: 'Query',
  fields: () => ({
    todos: {
      type: new GraphQLList(TodoType),
      resolve: () => {
        return promiseListAll()
      }
    }
  })
})
```

Of course we need to fetch data, and it's done by `promiseListAll()`. A promise to get a list of data from `TODO.find(...)`, what is a MongoDB schema. 

```js
// https://github.com/igorlima/expressjs-graphql-server/blob/master/schema.js
// lines 72 and 79
var promiseListAll = () => {
  return new Promise((resolve, reject) => {
    TODO.find((err, todos) => {
      if (err) reject(err)
      else resolve(todos)
    })
  })
}
```

In order to create a [MongoDB connection in Node](http://mongoosejs.com/index.html), we need to define previously a [Schema](http://mongoosejs.com/docs/guide.html). Once defining a schema, it will get a collection with the given name in *lower case* and *plural*. In this case, the schema name is `'Todo'`, so the collection should be `'todos'`. Then we just need to connect to the database by utilizing the [`mongoose.connect()`](http://mongoosejs.com/docs/connections.html) method.

> **NOTE:** I’m sharing my credentials here. Feel free to use it while you’re learning. After that, create and use your own credential. Thanks.

```js
// https://github.com/igorlima/expressjs-graphql-server/blob/master/schema.js#L12
// Mongoose Schema definition
var TodoSchema = new Schema({
  title: String,
  completed: Boolean
})
var TODO = mongoose.model('Todo', TodoSchema)

// https://github.com/igorlima/expressjs-graphql-server/blob/master/schema.js#L47
var COMPOSE_URI_DEFAULT = 'mongodb://example:example@candidate.54.mongolayer.com:10775,candidate.57.mongolayer.com:10128/spark-server-with-mongo?replicaSet=set-5647f7c9cd9e2855e00007fb'

mongoose.connect(process.env.COMPOSE_URI || COMPOSE_URI_DEFAULT, function (error) {
  if (error) console.error(error)
  else console.log('mongo connected')
})
```

Now, everything is in place to define our GraphQL Schema in NodeJS. A connection, a query type, and the type of object `todo`. 

```js
// https://github.com/igorlima/expressjs-graphql-server/blob/master/schema.js#L279
module.exports = new GraphQLSchema({
  query: QueryType
})
```

## Client side code

Our client side is a [Todo List](http://todomvc.com/examples/react/#/) written in React. You can choose among [many JS frameworks over there](http://todomvc.com/). As I'm working in a React project these days, so I chose React as the front-end framework. However, feel free to choose any other JavaScript framework you're comfortable with. See how's this Todo List looks like:

![Todo List web application](https://cloud.githubusercontent.com/assets/1886786/12223911/13dad23e-b7cb-11e5-92a9-cbf13c5f9e6b.png)

* Source code
  * JAVA: https://github.com/igorlima/spark-graphql-server/tree/gh-pages
  * NodeJS: https://github.com/igorlima/expressjs-graphql-server/tree/gh-pages 
* Live demo
  * JAVA: http://igorlima.github.io/spark-graphql-server/#/
  * NodeJS: http://igorlima.github.io/expressjs-graphql-server/#/ 

The client-side code is exactly the same for both servers. Also, both GraphQL servers, in JAVA and Node, implement a type of query to fetch all `todos` as well as six different mutations: `add`, `toggle`, `toggleAll`, `destroy`, `clearCompleted` and `save`. All kind of queries needed by the client side example.


## Metrics


### Using the tool [webpagetest](https://www.npmjs.com/package/webpagetest)

The time for rendering the assets (*html*, *css* and *js* files) can be disregarded. Because we have the same asset files hosted by [github page](https://pages.github.com/). So let's compare the time spent by calling the GraphQL server.

NodeJS server | JAVA server
------------- | ------------
![expressjs server - screen shot 2016-01-10 at 10 59 05 am](https://cloud.githubusercontent.com/assets/1886786/12223870/0fb6db9a-b7ca-11e5-9523-8e4755bf6eb0.png) | ![spark server - screen shot 2016-01-10 at 10 59 59 am](https://cloud.githubusercontent.com/assets/1886786/12223871/15b56886-b7ca-11e5-924c-68ec89811cab.png)

On the left side, there is the NodeJS server spending *311ms* plus *63ms* with a total of **374ms**. Against JAVA server with *316ms* plus *144ms*, a total of **460ms**.

### Using the tool [loadtest](https://www.npmjs.com/package/loadtest)

To run the *[loadtest](https://www.npmjs.com/package/loadtest)* we need to specify the amount of max request and the concurrency level. Respectively, these arguments are `-n` and `-c`.  First test uses the following query to get all *todos*:

```js
query Todo {
  todos {
    id,
    title,
    completed
  }
}
```

The command line used was:

```sh
# For JAVA
loadtest -c 10 -n 1000 -T application/graphql -P 'query Todo { todos {id, title, completed} }' -m POST 'http://spark-graphql-server.herokuapp.com/graphql'
```

and 

```sh
# For Node
loadtest -c 10 -n 1000 -T application/graphql -P 'query Todo { todos {id, title, completed} }' -m POST 'http://expressjs-graphql-server.herokuapp.com/graphql'
```

Here is the result of the first test.

Metric | ExpressJS (NodeJS) | Spark (JAVA)
-------|--------------------|-------
Completed requests|1000|1000
Concurrency level|10|10
Total errors|0|0
Requests/second|29|29

Percentage of the requests served within a certain time | ExpressJS (NodeJS) | Spark (JAVA)
----|--------------------|-------------
50% | 342 ms | 333 ms
90% | 363 ms | 360 ms
95% | 370 ms | 383 ms
99% | 495 ms | 908 ms
100% (longest request)| 526 ms | 1028 ms
 
-----

The second test uses the following query to *toggle* a specific *todo*:

```js
mutation Todo {
  toggle(id: "569245602404f9e4145b2e02") {
    id,
    title,
    completed
  }
}
```

The command line used was:

```sh
# For JAVA
loadtest -c 10 -n 1000 -T application/graphql -P 'mutation Todo { toggle(id: "569245602404f9e4145b2e02") { id, title, completed }}' -m POST 'http://spark-graphql-server.herokuapp.com/graphql'
```

and 

```sh
# For NodeJS
loadtest -c 10 -n 1000 -T application/graphql -P 'mutation Todo { toggle(id: "569245602404f9e4145b2e02") { id, title, completed }}' -m POST 'http://expressjs-graphql-server.herokuapp.com/graphql'
```

Here is the result of the second test.

Metric | ExpressJS (NodeJS) | Spark (JAVA)
-------|--------------------|-------
Completed requests|1000|1000
Concurrency level|10|10
Total errors|0|0
Requests/second|28|29

Percentage of the requests served within a certain time | ExpressJS (NodeJS) | Spark (JAVA)
----|--------------------|-------------
50% | 349 ms | 339 ms
90% | 367 ms | 365 ms
95% | 380 ms | 378 ms
99% | 698 ms | 909 ms
100% (longest request)| 736 ms | 937 ms

### Using the Chrome browser

In the live demo is noticeable the slowness of JAVA implementation. When we try to *toggle* a todo or even *delete* one, we can see an icon of loading... Again, we are using (*exactly*) the same client side code. Below is an image with a comparison of this *toggling*:

NodeJS server | JAVA server
------------- | ------------
![screen shot 2016-01-10 at 12 05 00 pm](https://cloud.githubusercontent.com/assets/1886786/12223932/73eedc1a-b7cb-11e5-86e2-46e77725c312.png) | ![screen shot 2016-01-10 at 12 07 00 pm](https://cloud.githubusercontent.com/assets/1886786/12223935/7ae33854-b7cb-11e5-90f4-48ebd874c936.png)

As seen, Java needs **2157ms** for this operation. On the other side, Node needs less than a second: **695ms**.

## Conclusion

At the end of the day, in terms of perfomance, I ended up in a conclusion that the GraphQL in Node is way better in comparison to JAVA implementation. Please feel free to use the code and comment on the analysis we have done.
