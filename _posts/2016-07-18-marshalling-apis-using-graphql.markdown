---
layout: post
title:  "Marshalling API's using GraphQL"
date:   2016-07-18
categories: jekyll update
---
### Marshalling API's using GraphQL

In this post we'll demonstrate how easy it is to use [GraphQL](http://graphql.org) to marshall different endpoints in an API.

The first thing we need is some data. The site [JSONPlaceholder](http://jsonplaceholder.typicode.com) contains a few commonly used endpoints that you can experiment on. So let's take a look at what's returned by the **post** and **comment** endpoints: 

To get the post with id equal to 1, you just call the endpoint [http://jsonplaceholder.typicode.com/posts/1](http://jsonplaceholder.typicode.com/posts/1) to return the JSON

{% highlight json %}
{
  "userId": 1,
  "id": 1,
  "title": "sunt aut facere repellat provident occaecati excepturi optio reprehenderit",
  "body": "quia et suscipit\nsuscipit recusandae consequuntur expedita et cum\nreprehenderit molestiae ut ut quas totam\nnostrum rerum est autem sunt rem eveniet architecto"
}
{% endhighlight %}

Then, to get the comments related to the above post, you need to call the endpoint [http://jsonplaceholder.typicode.com/comments?postId=1](http://jsonplaceholder.typicode.com/comments?postId=1) to return the JSON
{% highlight json %}
[
  {
    "postId": 1,
    "id": 1,
    "name": "id labore ex et quam laborum",
    "email": "Eliseo@gardner.biz",
    "body": "laudantium enim quasi est quidem magnam voluptate ipsam eos\ntempora quo necessitatibus\ndolor quam autem quasi\nreiciendis et nam sapiente accusantium"
  },
  {
    "postId": 1,
    "id": 2,
    "name": "quo vero reiciendis velit similique earum",
    "email": "Jayne_Kuhic@sydney.com",
    "body": "est natus enim nihil est dolore omnis voluptatem numquam\net omnis occaecati quod ullam at\nvoluptatem error expedita pariatur\nnihil sint nostrum voluptatem reiciendis et"
  },
  {
    "postId": 1,
    "id": 3,
    "name": "odio adipisci rerum aut animi",
    "email": "Nikita@garfield.biz",
    "body": "quia molestiae reprehenderit quasi aspernatur\naut expedita occaecati aliquam eveniet laudantium\nomnis quibusdam delectus saepe quia accusamus maiores nam est\ncum et ducimus et vero voluptates excepturi deleniti ratione"
  },
  {
    "postId": 1,
    "id": 4,
    "name": "alias odio sit",
    "email": "Lew@alysha.tv",
    "body": "non et atque\noccaecati deserunt quas accusantium unde odit nobis qui voluptatem\nquia voluptas consequuntur itaque dolor\net qui rerum deleniti ut occaecati"
  },
  {
    "postId": 1,
    "id": 5,
    "name": "vero eaque aliquid doloribus et culpa",
    "email": "Hayden@althea.biz",
    "body": "harum non quasi et ratione\ntempore iure ex voluptates in ratione\nharum architecto fugit inventore cupiditate\nvoluptates magni quo et"
  }
]
{% endhighlight %}

Now let's say we want to get comments with posts, or a partial selection of fields in either comments or posts. For example, we might want to know the email address of all the comments associated to a post, or we might want to know the title of the post clubbed with all its comments, and so on. There are many ways to do this: adding a new endpoint, using object queries, writing repositories, and so on. It would be nice if we could marshall these endpoints together and create a new api that we can ask for just the data we want. We can do this using GraphQL. 

GraphQL was created at Facebook in 2012. It's a data query language and it's used in their web and mobile apps. It's easy to learn and there are loads great tutorials that cover the basics. See the [getting started](http://graphql.org/docs/getting-started/) guide to try it out. 

We're not going to go into depth on how to set up a GraphQL server since there are plenty of guides that tell you how to do that. (Although I'll include the code so you can see how it all hangs together.) Instead, I'll concentrate on how to create a query that gets the data we're looking for:

We'll start by defining a `GraphQLObjectType` for comments. If you look at the JSON returned from the comments endpoint, then you'll see that it contains the following fields: **id**, **postId**, **name**, **email** and **body**. We can use this information to define the following type:
{% highlight js %}
const CommentType = new GraphQLObjectType({
    name: 'Comment',
    fields: () => ({
        id: {type: GraphQLInt},
        postId: {type: GraphQLInt},
        name: {type: GraphQLString},
        email: {type: GraphQLString},
        body: {type: GraphQLString}
    })
})
{% endhighlight %}
Here, we've used the datatype `GraphQLInt` for id and postId but just `GraphQLString` for the other fields.

The next thing we need to do is define the `PostType`. Similarily, by looking at the JSON returned by the post endpoint, we do this as follows:
{% highlight js %}
const PostType = new GraphQLObjectType({
  name: 'Post',
  fields: () => ({
    id: {type: GraphQLInt},
    userId: {type: GraphQLInt},
    title: {type: GraphQLString},
    body: {type: GraphQLString}
})
{% endhighlight %}
So how do we associate comments to a post? To do this, we need to create a new field called comments on the `PostType`, give it the `GraphQLList` type (since we're expecting a list of comments) and tell it how to get the comments associated to the post. We do this by defining the resolve function for the comments field. Since this has to be a function that returns JSON that matches the comment type, we'll implement it by defining a function that calls the comments endpoint with the querystring parameter posts with value equal to the post's id. We do this as follows:
{% highlight js %}
const PostType = new GraphQLObjectType({
  name: 'Post',
  fields: () => ({
    id: {type: GraphQLInt},
    userId: {type: GraphQLInt},
    title: {type: GraphQLString},
    body: {type: GraphQLString},
    comments: {
        type: new GraphQLList(CommentType),
        resolve: (post) => getJSON(`/comments?postId=${post.id}`)
    }})
})
{% endhighlight %}
Where `getJSON` is simply defined as
{% highlight js %}
let getJSON = rel => fetch(`http://jsonplaceholder.typicode.com${rel}`)
                    .then(res => res.json())
{% endhighlight %}
Now we just need to define a `QueryType` that specifies how to query a `PostType`:
{% highlight js %}
const QueryType = new GraphQLObjectType({
    name: 'PostQuery',
    fields: () => ({
      post: {
        type: PostType,
        args: {
           id: {type: GraphQLInt}
        },
        resolve: (parent, args) => getJSON(`/posts/${args.id}/`)
    }})
})
{% endhighlight %}

Let's try this out. Create the file schema.js with the contents:
{% highlight js %}
import {
    GraphQLSchema,
    GraphQLObjectType,
    GraphQLString,
    GraphQLInt,
    GraphQLList
} from 'graphql';

import fetch from 'node-fetch';

const CommentType = new GraphQLObjectType({
    name: 'Comment',
    fields: () => ({
        id: {type: GraphQLInt},
        postId: {type: GraphQLInt},
        name: {type: GraphQLString},
        email: {type: GraphQLString},
        body: {type: GraphQLString}
    })
})

const PostType = new GraphQLObjectType({
  name: 'Post',
  fields: () => ({
    id: {type: GraphQLInt},
    userId: {type: GraphQLInt},
    title: {type: GraphQLString},
    body: {type: GraphQLString},
    comments: {
        type: new GraphQLList(CommentType),
        resolve: post => getJSON(`/comments?postId=${post.id}`)
    }})
})

const QueryType = new GraphQLObjectType({
    name: 'PostQuery',
    fields: () => ({
      post: {
        type: PostType,
        args: {
           id: {type: GraphQLInt}
        },
        resolve: (parent, args) => getJSON(`/posts/${args.id}/`)
    }})
})

let getJSON = rel => fetch(`http://jsonplaceholder.typicode.com${rel}`).then(res => res.json())

export default new GraphQLSchema({
    query: QueryType
})
{% endhighlight %}
And the file index.js with the content:
{% highlight js %}
import express from 'express';
import graphQLHTTP from 'express-graphql';
import schema from './schema';

const app = express();

app.use(graphQLHTTP({
  schema,
  graphiql: true
}))

app.listen(8000);
{% endhighlight %}

After installing any packages you need, run the code by pointing node at index.js (or babel-node index.js --presets es2015, etc...). Navigating to port 8000 opens up the GraphiQL user interface that lets you run your queries live:

![GraphiQL](/images/graphql.png)

So let's try out some queries:

Suppose we want a post's title and all its emails. Then,

{% highlight js %}
{
  post(id:1){
    title
    comments{
      email
    }
  }
}
{% endhighlight %}
returns 

{% highlight json %}
{
  "data": {
    "post": {
      "title": "sunt aut facere repellat provident occaecati excepturi optio reprehenderit",
      "comments": [
        {
          "email": "Eliseo@gardner.biz"
        },
        {
          "email": "Jayne_Kuhic@sydney.com"
        },
        {
          "email": "Nikita@garfield.biz"
        },
        {
          "email": "Lew@alysha.tv"
        },
        {
          "email": "Hayden@althea.biz"
        }
      ]
    }
  }
}
{% endhighlight %}

Now suppose we want a post's id, and the fields name, postId, email, and body from its comments. Then,

{% highlight js %}
{
  post(id:84){
    id,
    comments{
      name,
      postId,
      email,
      body
    }
  }
}
{% endhighlight %}

returns 

```
{
  "data": {
    "post": {
      "id": 84,
      "comments": [
        {
          "name": "neque nemo consequatur ea fugit aut esse suscipit dolore",
          "postId": 84,
          "email": "Aurelie_McKenzie@providenci.biz",
          "body": "enim velit consequatur excepturi corporis voluptatem nostrum\nnesciunt alias perspiciatis corporis\nneque at eius porro sapiente ratione maiores natus\nfacere molestiae vel explicabo voluptas unde"
        },
        {
          "name": "quia reiciendis nobis minima quia et saepe",
          "postId": 84,
          "email": "May_Steuber@virgil.net",
          "body": "et vitae consectetur ut voluptatem\net et eveniet sit\nincidunt tenetur voluptatem\nprovident occaecati exercitationem neque placeat"
        },
        {
          "name": "nesciunt voluptates amet sint et delectus et dolore culpa",
          "postId": 84,
          "email": "Tessie@emilie.co.uk",
          "body": "animi est eveniet officiis qui\naperiam dolore occaecati enim aut reiciendis\nanimi ad sint labore blanditiis adipisci voluptatibus eius error\nomnis rerum facere architecto occaecati rerum"
        },
        {
          "name": "omnis voluptate dolorem similique totam",
          "postId": 84,
          "email": "Priscilla@colten.org",
          "body": "cum neque recusandae occaecati aliquam reprehenderit ullam saepe veniam\nquasi ea provident tenetur architecto ad\ncupiditate molestiae mollitia molestias debitis eveniet doloremque voluptatem aut\ndolore consequatur nihil facere et"
        },
        {
          "name": "aut recusandae a sit voluptas explicabo nam et",
          "postId": 84,
          "email": "Aylin@abigale.me",
          "body": "voluptas cum eum minima rem\ndolorem et nemo repellendus voluptatem sit\nrepudiandae nulla qui recusandae nobis\nblanditiis perspiciatis dolor ipsam reprehenderit odio"
        }
      ]
    }
  }
}
```