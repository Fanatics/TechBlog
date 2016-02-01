# Why GraphQL?

We started from the premise that the front-end UI as a cake. Think of we wants to bake a cake by following a recipe. In a way we could get a box with everything we need, already measured and weighted to match our recipe. Pretty easier huh. This is what GraphQL is meant to do. It fits the need to simplify HTTP requests from the front-end UI.

In this post, we’ll write a small GraphQL server to respond to requests from a hypothetical e-commerce application. And see how GraphQL helps baking a light-weight front-end UI. To provide a foundation for this, let's go through a simple example to understand everything to get going with GraphQL. The goal is to keep this as simple as possible.

GraphQL itself is a query language that allows the client to describe the data it needs and its shape by providing a common interface for client and server. It makes retrieving data more efficient because it encourages using only the data needed, rather than retrieving a fixed set of data. For instance, a e-commerce application may have a lot of information for each product. Any product has its `title`, `id`, `details`, `description` and so forth. A GraphQL query to request only the `title` and `id` from a GraphQL server would be:

```
query {
  products {
    id,
    tittle
  }
}
```

Which produces the resulting data (in JSON):

```
{
  "data": {
    "products": [
      {
        "id": 2354264,
        "title": "NaVorro Bowman Team Irvin Nike 2016 Pro Bowl Game Jersey - Black"
      },
      {
        "id": 2326907,
        "title": "Super Bowl 50 Nike Limited Jersey - Black"
      }
    ]
  }
}
```

The JSON response contains just the fields described in the query. As seen, the query has the same JSON format without the values. In other cases, we might need other fields such as `description`. To fetch that, we just need to add the new field name in the query:

```
query {
  products {
    id,
    tittle,
    description
  }
}
```

It requests a list of `products` containing these following fields `id`, `title` and `description`. Again, it describes the entity `product` it needs, also its attributes.

What described so far it's a contract of what is provided by the server to a web application. Basically that, a contract. In order to implement this contract in NodeJS, we can use GraphQL.js. This library provides two important capabilities:

1. support for building a type schema
2. and support for serving queries against that type schema.

To make use of these, first, we'll need to build a GraphQL type schema which maps to our code base. Here's what we'll be mapping in the schema:

```js
var graphql = require ('graphql');
// Here is some dummy data to make this piece of code simpler.
var PRODUCTS = [
  {
    "id": 2354264,
    "title": "NaVorro Bowman Team Irvin Nike 2016 Pro Bowl Game Jersey - Black",
    "description": "Get the look of a tried and true football fan with this 2016 Pro Bowl Game jersey and let everyone know you're a die-hard NaVorro Bowman fan! Rock this Nike jersey to watch your favorite player take on the competition. You'll love paying homage to Team Irvin with this jersey."
  },
  {
    "id": 2326907,
    "title": "Super Bowl 50 Nike Limited Jersey - Black",
    "description": "You've been watching the Super Bowl for as long as you can remember, and now we are at a milestone: 50 Super Bowls. You can celebrate the golden anniversary of the NFL's biggest game when you grab this Super Bowl 50 limited jersey from Nike. Featuring stunning colors and graphics, you'll be able to show your excitement for the big game everywhere you go when you wear this jersey."
  }
];
```

This is a sample data of two products for us to work with for now. Based on that sample data, let's define the product type and its attributes: id, title and description.

```js
var ProductType = new graphql.GraphQLObjectType({
  name: 'productType',
  fields: function () {
    return {
      id: {
        type: graphql.GraphQLInt
      },
      title: {
        type: graphql.GraphQLString
      },
      description: {
        type: graphql.GraphQLString
      }
    }
  }
});
```

With that product data and its type in hand, the next step is to resolve a query by returning the data through a query type.

```js
var queryType = new graphql.GraphQLObjectType({
  name: 'Query',
  fields: function () {
    return {
      products: {
        type: new graphql.GraphQLList(ProductType),
        resolve: function () {
          return PRODUCTS;
        }
      }
    }
  }
});
```

The `resolve` property is the heart to retrieve the data. In this case, we are returning an array in memory, and of course the return is a synchronous operation. In case we need to do any asynchronous operations, such as say retrieving data from a database, we can make use of a promise which can wrap the asynchronous operations for us. Let's use the sync way to keep it simpler.

Finally we export the query type:

```js
module.exports = new graphql.GraphQLSchema({
  query: queryType
});
```

So let's gather everything and put all pieces together in the file `schema.js`:

```js
// schema.js
var graphql = require ('graphql');

// Here is some dummy data to make this piece of code simpler.
var PRODUCTS = [
  {
    "id": 2354264,
    "title": "NaVorro Bowman Team Irvin Nike 2016 Pro Bowl Game Jersey - Black",
    "description": "Get the look of a tried and true football fan with this 2016 Pro Bowl Game jersey and let everyone know you're a die-hard NaVorro Bowman fan! Rock this Nike jersey to watch your favorite player take on the competition. You'll love paying homage to Team Irvin with this jersey."
  },
  {
    "id": 2238973,
    "title": "Super Bowl 50 Nike Limited Jersey - Black",
    "description": "You've been watching the Super Bowl for as long as you can remember, and now we are at a milestone: 50 Super Bowls. You can celebrate the golden anniversary of the NFL's biggest game when you grab this Super Bowl 50 limited jersey from Nike. Featuring stunning colors and graphics, you'll be able to show your excitement for the big game everywhere you go when you wear this jersey."
  }
];

var ProductType = new graphql.GraphQLObjectType({
  name: 'productType',
  fields: function () {
    return {
      id: {
        type: graphql.GraphQLInt
      },
      title: {
        type: graphql.GraphQLString
      },
      description: {
        type: graphql.GraphQLString
      }
    }
  }
});

var queryType = new graphql.GraphQLObjectType({
  name: 'Query',
  fields: function () {
    return {
      products: {
        type: new graphql.GraphQLList(ProductType),
        resolve: function () {
          return PRODUCTS;
        }
      }
    }
  }
});

module.exports = new graphql.GraphQLSchema({
  query: queryType
});
```

In all, this defines a simple schema with a product type and a list of products - of which each elements has three fields - that resolves to a value retrieved from the array.

Here it is the server code that serves queries against our schema:

```js
// server.js
var graphql = require ('graphql').graphql
var express = require('express')
var graphQLHTTP = require('express-graphql')
var Schema = require('./schema')

var app = express()
  .use('/', graphQLHTTP({ schema: Schema, pretty: true }))
  .listen(8080, function (err) {
    console.log('GraphQL Server is now running on localhost:8080');
  });
```

At this point we have provided the two things:

1. a type schema `schema.js`
2. a server `server.js` to serve queries against a type schema

To run the code, execute `npm install graphql express express-graphql` to get all dependencies and then do `node server.js` to run the program.

```sh
# install all dependencies
npm install graphql express express-graphql

# to run the server
node server.js
```

When it runs we should have a server up and running in the port `8080`. We can check it out opening the url `http://localhost:8080` in any browsers or typing in the command line:

```sh
$ curl -XPOST -H "Content-Type:application/graphql"  -d 'query { products { id, title, description } }' http://localhost:8080
{
  "data": {
    "products": [
      {
        "id": 2354264,
        "title": "NaVorro Bowman Team Irvin Nike 2016 Pro Bowl Game Jersey - Black",
        "description": "Get the look of a tried and true football fan with this 2016 Pro Bowl Game jersey and let everyone know you're a die-hard NaVorro Bowman fan! Rock this Nike jersey to watch your favorite player take on the competition. You'll love paying homage to Team Irvin with this jersey."
      },
      {
        "id": 2326907,
        "title": "Super Bowl 50 Nike Limited Jersey - Black",
        "description": "You've been watching the Super Bowl for as long as you can remember, and now we are at a milestone: 50 Super Bowls. You can celebrate the golden anniversary of the NFL's biggest game when you grab this Super Bowl 50 limited jersey from Nike. Featuring stunning colors and graphics, you'll be able to show your excitement for the big game everywhere you go when you wear this jersey."
      }
    ]
  }
}
```

It's basically that how GraphQL works behind scenes. As we can see, GraphQL is a contract that has been designed with a flexible syntax that makes building client applications easier. Also, we can see how GraphQL challenges REST API. For instance, as we know, [a REST API typically delivers all the data a client UI might need about a resource and leaves it up to the client to extract the bits of data it actually wants to show](https://www.compose.io/articles/using-graphql-with-mongodb/). If that data is not a resource it already has, then the client needs to go off to the server and request some more data from another URL. Over time, the link between front-end applications and back-end servers can become pretty rigid. As data gets more complex in structure, it gets harder and slower to query with the traditional REST API.

Further more, GraphQL does not mandate the use of a particular language – JAVA, Ruby and many others. There is no secret on implementing GraphQL in other program languages as well. Feel free to check it out [here](https://github.com/chentsulin/awesome-graphql).

What we have seen so far in this tutorial provide us much foundation to understand how we can query data on each different e-commerce pages. Let's start querying four different products.

```js
query {
  products(ids: ['Product A', 'Product B', 'Product C', 'Product D']) {
    id,
    title,
    image {
      src
    }
  }
}
```

Product A | Product B | Product C | Product D
-------- | -------- | -------- | --------
![screen shot 2016-01-28 at 5 07 19 pm](https://cloud.githubusercontent.com/assets/1886786/12664456/78c1b2cc-c5e4-11e5-8222-5db44a58b3af.png) | ![screen shot 2016-01-28 at 5 07 37 pm](https://cloud.githubusercontent.com/assets/1886786/12664460/7c10a532-c5e4-11e5-9558-9cad1e924793.png) | ![screen shot 2016-01-28 at 5 08 27 pm](https://cloud.githubusercontent.com/assets/1886786/12664464/82b516e8-c5e4-11e5-852f-58d9b28208c7.png) | ![screen shot 2016-01-28 at 5 10 20 pm](https://cloud.githubusercontent.com/assets/1886786/12664466/899444e8-c5e4-11e5-9d9d-357177453412.png)

It's getting much clear huh. Moving forward let's query data to create a grid of products with pagination.

```js
query {
  search( queryString: "Jersey" ) {
    currentPage,
    productPerPage,
    totalProducts,
    products {
      id,
      title,
      image {
        src
      },
      price {
        regular {
          money {
            currency,
            value
          }
        }
      }
    }
  }
}
```

![screen shot 2016-01-28 at 5 23 47 pm](https://cloud.githubusercontent.com/assets/1886786/12664488/b1ea63b4-c5e4-11e5-8c27-62b2751a2bf0.png)


You see? Isn't it a kind of gathering pieces to mount a puzzle? Pretty no? Again the great thing here it requests the exact data the UI needs, neither more nor less than.

We can show up more examples such as the global header and the left navigation. Take a look how it would look like:

```js
query {
  header {
    logo {
      image {
        src
      }
    },
    navigation {
      links {
        title,
        href
      }
    }
  },
  leftNavigation (productId: "Product A") {
    facets {
      type,
      links {
        title,
        href
      }
    }
  }
}
```

global header |   
--- | --- 
![screen shot 2016-01-28 at 5 25 28 pm](https://cloud.githubusercontent.com/assets/1886786/12664486/ac25b19a-c5e4-11e5-88b2-474d96f407b2.png) |

left navigation |  |  |
------- | ---------- | --------- |
![left-nav-1](https://cloud.githubusercontent.com/assets/1886786/12699579/08cbbdb8-c775-11e5-962b-7058a977803f.png) | ![left-nav-2](https://cloud.githubusercontent.com/assets/1886786/12699577/08c8bffa-c775-11e5-89ae-c7bc045fa43c.png) | ![left-nav-3](https://cloud.githubusercontent.com/assets/1886786/12699578/08cb4e14-c775-11e5-9827-a8df33c6dd78.png) |


Let's pick up one more example, the last one, to finish the set of examples as a whole. As a e-commerce, the product detail page is the page where the magic is supposed to happen. It's usually the page with a variety of features. A cleaner HTTP request makes possible to make this page even more light-weight. Here it's a GraphQL query:

```js
query {
  product (id: "Product A") {
    id,
    title,
    details,
    description,
    shipping {
      id,
      description
    },
    image {
      src,
      title
    },
    price {
      regular {
        money {
          currency,
          value
        }
      },
      sale {
        money {
          currency,
          value
        }
      }
    },
    sizes {
      code,
      name
    },
    colors {
      code,
      name
    }
  }
}
```

![screen shot 2016-01-28 at 5 21 11 pm](https://cloud.githubusercontent.com/assets/1886786/12664472/93410cba-c5e4-11e5-91f6-03f0e485cacb.png)


That's it. In summary, GraphQL is an alternative to REST endpoints for handling queries and database updates. An alternative that makes simpler to implement the following idea - instead of defining the structure of responses on the server, the client has the flexibility to define what it wants in response to its queries.
