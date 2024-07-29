# Odyssey Voyage I

In this project we have two services: locations and reviews,

To create supergraph and combine this services to one we need to convert service to a subraph

1. need to isntall `npm install @apollo/subgraph`

2. Add this to .graphql file in your project `extend schema
@link(url: "https://specs.apollo.dev/federation/v2.0", import: ["@key"])`

3. import `const { buildSubgraphSchema } = require('@apollo/subgraph')`

4. Add `buildSubgraphSchema` to server configuration: `const server = new ApolloServer({
  schema: buildSubgraphSchema({ typeDefs, resolvers }),
});`

## Next step to create supergraph in Apollo Studio and publish our subgraphs

Let's start by spinning up a new supergraph for FlyBy.

1. Open a new browser window and go to GraphOS Studio(studio.apollographql.com
   ).

2. If you haven't created a graph in Studio before now, you'll can get started by clicking Connect a GraphQL API.

Otherwise, we can click the + Create New Graph button in the upper right corner of the Graphs tab.

studio.apollographql.com

3. We'll give our graph a descriptive title, keep the default settings for Graph Architecture as "Supergraph", then click Next.

Storing GraphOS environment variables
To publish our subgraphs using Rover, we'll need to save two environment variables from Studio:

APOLLO_KEY: An API key for authenticating Rover. It starts with something like service:your-graph-name.
APOLLO_GRAPH_REF: The graph reference (or graph ref) for our supergraph, which we'll use to tell Rover where to publish our subgraphs.
A graph ref starts with the graph's ID, followed by an @ symbol, followed by the graph variant.
We can get the values for APOLLO_KEY and APOLLO_GRAPH_REF out of the publishing options modal in Studio.

Go back to the configuration options in Studio that appeared after you created your supergraph. Make sure you're on the Schema Document tab.

First, make sure that the Supergraph Pipeline Track dropdown is set to Federation 2.7 Supergraph. This specifies that our supergraph should be built using the latest features of Apollo Federation.

Below, take a little peek at the command for publishing a subgraph schema. We'll be running this command shortly, but for now, we're more interested in the APOLLO_KEY environment variable here.

APOLLO_KEY=your-graphs-apollo-key \
rover subgraph publish your-graph-name@current \
 --name products --schema ./products-schema.graphql \
 --routing-url http://products.prod.svc.cluster.local:4001/graphql
Click on the eye icon on the code block to reveal the full value of APOLLO_KEY. Copy APOLLO_KEY and its value to your clipboard.

Back in our code editor, we'll create a new file in the router directory called .env.

Paste your APOLLO_KEY into router/.env.

router/.env
APOLLO_KEY=your-graphs-apollo-key
Now let's go back to Studio to get our graph ref. The value we're looking for appears in the same code block, directly after the "rover subgraph publish" part of the command. We'll copy this value onto our clipboard.

APOLLO_KEY=your-graphs-apollo-key \
 rover subgraph publish your-graph-name@current \
--name products --schema ./products-schema.graphql \
 --routing-url http://products.prod.svc.cluster.local:4001/graphql
Note: Our graph ref uses the current variant for our supergraph, which is the default variant. We'll talk more about variants in Voyage III: Federation in Production.

In your router/.env file, add a new line and set your APOLLO_GRAPH_REF environment variable:

router/.env
APOLLO_KEY=your-graphs-apollo-key
APOLLO_GRAPH_REF=your-graph-name@current

We've got the values we need to publish our subgraphs!

The rover subgraph publish command
Rover has a command ready to help us with this important task: rover subgraph publish. This command pushes the latest version of a single subgraph schema to GraphOS.

rover subgraph publish <APOLLO_GRAPH_REF> \
 --name <SUBGRAPH NAME> \
 --schema <SCHEMA FILE PATH> \
 --routing-url <ROUTING URL>
To use this command, we need the graph ref for the supergraph we want to publish to and the following command line options:

Option What is it?
--name What we want to call our subgraph in GraphOS
--schema The relative path to our subgraph's schema file
--routing-url The URL where our subgraph runs (locally, for now)
We'll fill in these options with the details for each of our subgraphs.

✏️ Publishing the locations subgraph
Let's see this command in action by publishing our locations subgraph.

Bounce back to the terminal and make sure we're in the root directory for the project.

Now let's type out the rover subgraph publish command:

We'll paste in the value of our APOLLO_GRAPH_REF environment variable.

For the name option, we'll pass in locations.

For the schema option, we'll pass the relative path to our locations.graphql file.

And for the routing-url option, we'll pass in localhost:4001.

rover subgraph publish <APOLLO_GRAPH_REF> \
 --name locations \
 --schema ./subgraph-locations/locations.graphql \
 --routing-url http://localhost:4001
Note: We've used the \ character in this command to improve legibility by putting each command-line option on its own line. If you choose to type the entire rover subgraph publish command on a single line, you don't need to include the \.

After running the command, you'll see the following message:

The host `localhost` is not routable via the public internet.
Continuing the publish will make this subgraph reachable in local environments only.
Would you still like to publish? [y/N]
Type in y to allow it. We're in tutorial land and working in local environments for now!

Note that you'll need to do this after every subgraph schema publish command.

If all is well in the world, running this command should output a message confirming that the subgraph has been published and the supergraph has been updated!

# Key takeaways

We can use the rover subgraph publish command from the Rover CLI to publish our subgraph schemas to the Apollo schema registry.
Whenever a new subgraph schema is published, GraphOS composes a new supergraph schema with any subgraphs registered to our supergraph.
The supergraph schema consolidates all the types and fields across our published subgraphs. It also includes extra directives to help the router determine which subgraphs can resolve each field.

`Client - request - Router (Supergraph Schema) - Subgraphs`

`[Client] <--Sends Queries/Mutations--> [Apollo Client]
[Apollo Client] <--Sends Requests--> [Apollo Gateway]
[Apollo Gateway] <--Forwards Queries--> [Service A (Apollo Server)]
[Apollo Gateway] <--Forwards Queries--> [Service B (Apollo Server)]
[Service A (Apollo Server)] <--Resolves Queries--> [Apollo Server]
[Service B (Apollo Server)] <--Resolves Queries--> [Apollo Server]
[Apollo Gateway] <--Schema Management--> [Apollo Studio]`

Router use supergraph schema to resolve incoming GraphQL operations from the client.

## Step 1: The client request

First, the client sends a GraphQL operation to the router. The client has no clue which fields belong to which subgraphs—or even that there are subgraphs at all!

```graphql
Client -> {
  query {
    field 1
    field 2
    field 3
  }
} -> Router
```

![Alt text](./assets/F_01_06_IL_15_x4eaok.png)

## Step 2: Building a query plan

Router checks query against supergraph schema

![Alt text](./assets/F_01_05_IL_16_a_dwcrpn.png)

It uses this information to build a query plan, a list of smaller GraphQL operations to execute on the subgraphs. The query plan also specifies the order in which the subgraph operations need to run.

![Alt text](./assets/F_01_05_IL_16_b_nqqohx.png)

## Step 3: Executing the query plan

Next, the router carries out the query plan by sending the smaller GraphQL operations to each of the subgraphs it needs data from.

![Alt text](./assets/F_01_05_IL_103_a_itahgs.png)

The subgraphs resolve the operations the same way as any other GraphQL server: they use their resolvers and data sources to retrieve and populate the requested data.

## Step 4: The subgraph responses

The subgraphs send back the requested data to the router, and then the router combines all those responses into a single JSON object.

![Alt text](./assets/F_01_06_IL_18_th6u9s.png)

## Step 5: Sending data back to the client

Finally, the router sends the final JSON object back to the client. And that's the end of our operation's journey!

![Alt text](./assets/F_01_06_IL_19_nrfrw9.png)

## Recap

Here's the entire journey of a GraphQL operation through the supergraph, summarized in a single diagram:

![Alt text](./assets/federation_1_query_wpi_wad3yi.png)

## Key takeaways

- The router uses the supergraph schema to create a query plan for the incoming GraphQL operation. The query plan is a list of smaller operations the router can execute on different subgraphs to fully resolve the incoming operation.
- The router carries out the query plan by executing the list of operations on the appropriate subgraphs.
- The router combines all the responses from the subgraphs into a single JSON object, which it sends back to the client.

# Setting up the Apollo Router

In this lesson, we will:

- Set up the GraphOS Router locally
- Connect the router to GraphOS
- Send our first query to our supergraph

# 8. ROUTER CONFIGURATION AND UPLINK

### Overview

> So far, FlyBy's subgraphs are running and their schemas have been published, but we still need one piece to tie everything together: the router.

In this lesson, we will:

- Set up the GraphOS Router locally
- Connect the router to GraphOS
- Send our first query to our supergraph

### Downloading the router

The GraphOS Router is a high-performance graph router available as an executable binary that you can add to your project in a few steps:

1. Open a terminal window and navigate to the router directory in the FlyBy project.

`cd router`

So far, we only have the `.env` file in here with our environment variables.

> 📦 router<br>
> ┣ 📄 .env

2. We'll download the Router by running the install command in the terminal.

`curl -sSL https://router.apollo.dev/download/nix/v1.46.0 | sh`

3. Now when we check the contents of our router directory, we'll see that we have a new file also called router!

> 📦 router<br>
> ┣ 📄 .env<br>
> ┗ 📄 router

### ✏️ Running the router

1. Back in the same terminal window, run the command below. You'll need to replace the <APOLLO_KEY> and <APOLLO_GRAPH_REF> placeholder values with your supergraph's corresponding values from the router/.env file. This command starts up the router locally and tells the router which supergraph to connect to.

`APOLLO_KEY=<APOLLO_KEY> APOLLO_GRAPH_REF=<APOLLO_GRAPH_REF> ./router`

2. We'll see a few lines of router output, and finally a message that our router is running on port 4000, ready to receive queries!

Let's copy this address, we'll need it to set our connection settings in Studio. This tells outside consumers of our API what endpoint they can use to query our schema.

`GraphQL endpoint exposed at http://127.0.0.1:4000/ 🚀`+

### ✏️ Connecting the router to GraphOS

Let's flip back over to Studio.

1. Click on the README tab in the sidebar.

2. Next, tap the Connection Settings link at the top of the page.

![Connecting the router to GraphOS](./assets/endpoint-connection-callout_v2.png)

3. We'll paste the router address we copied (http://127.0.0.1:4000) as the endpoint, then save.

![Connecting the router to GraphOS](./assets/set-connection-url-supergraph.png)

Let's get to querying our supergraph!

### ✏️ Testing our schema

1. Select the Explorer tab from the sidebar.

2. Let's put together a query that retrieves data from both of our subgraphs. We'll call our query `GetLocationsAndLatestReviews`.

3. Now let's fire in some fields. We'll start with `locations`. Click the plus button (⊕) next to Fields to add all the location fields to our query.

Next let's go back and add `latestReviews`, and all the reviews subfields.

Our query should look like this:

```graphql
query GetLocationsAndLatestReviews {
  locations {
    id
    name
    description
    photo
  }
  latestReviews {
    id
    comment
    rating
  }
}
```

4. Before we run the query, let's change the Response dropdown on the right to Query Plan Preview. This shows us a diagram for the query plan the router will use to resolve our current operation.

![Testing our schema](./assets/query-plan-preview-diagram.png)

5. By choosing the icon to Show plan as text, we'll see a more detailed breakdown of the query plan. We won't worry about all the syntax here, but we can get a general idea of how the router plans to handle this query: the `locations` subgraph will resolve the `locations` fields, and the `reviews` subgraph will handle `latestReviews` and its subfields.

![Testing our schema](./assets/show-plan-as-text-btn.png)

6. Now let's run this query.

![Testing our schema](./assets/get-locations-query.png)

### JSON response

```json
{
  "data": {
    "locations": [
      {
        "id": "loc-1",
        "name": "The Living Ocean of New Lemuria",
        "description": "Surviving is usually extremely difficult, especially when nutrients are scarce and you have to choose between growing or reproducing. One species on this planet has developed a nifty method to prepare for this. Once full matured, this species will split into 2 versions of itself and attached to each other, so it's essentially reproducing. Once those 2 are fully grown, they newly grown version will either detach itself if enough nutrients are available or it becomes a storage unit for the original, if nutrients are scarce. If nutrients continue to be scarce, the original will use slowly consume the nutrients in the new version in the hope that new nutrients become available again and it can repeat the cycle.",
        "photo": "https://res.cloudinary.com/apollographql/image/upload/v1644381344/odyssey/federation-course1/FlyBy%20illustrations/Landscape_4_lkmvlw.png"
      },
      {
        "id": "loc-2",
        "name": "Vinci",
        "description": "Many of the creatures on this planet have evolved into gliders, so to speak. Most of the fish and aquatic mammals, despite coming in various shapes and sizes, tend to glide through the water without effort, similar to how manta's glide on Earth. However, the surface species are more astonishing. Similar to the flying squirrels or the vultures of Earth, many of the species on this planet have developed ways to effortlessly move from one place to another by using the winds. But there is one species which shows signs of sentience. These species, a type of bird, love to play and have become masters of flight. Similar to how dolphins play, explore and learn, these species use their intellect and courage to play and sometimes challenge each other to death defying tricks.",
        "photo": "https://res.cloudinary.com/apollographql/image/upload/v1644381349/odyssey/federation-course1/FlyBy%20illustrations/Landscape_15_tiqel5.png"
      },
      {
        "id": "loc-3",
        "name": "Asteroid B-612",
        "description": "Nutrients are always needed but not always around, so organisms have to find ways to get them. Common ways are using different roots to find them in deep or shallow grounds or even stealing them from others, but on this planet many species have found a different balance. Unlike most plants on Earth who tend to only produce oxygen and nutrients, usually in the form of sugars, for itself, the organisms on this planet also produce other forms of nutrients for itself, usually for different purposes. These processes often lead to many byproducts which it doesn't need and are thus discarded. These discarded products are exactly what other species need to live and in turn produce byproducts it discards for the other organisms, leading to a delicate balance.",
        "photo": "https://res.cloudinary.com/apollographql/image/upload/v1644381343/odyssey/federation-course1/FlyBy%20illustrations/Landscape_6_vt6y3v.png"
      },
      {
        "id": "loc-4",
        "name": "Krypton",
        "description": "Similar to the surface, the underwater world has little more to offer than basic lifeforms. However, this planet has an astonishing water world. Almost everything is covered in a type of sea-grass. This grass varies in length depending on the region, but they're all part of the same species. But what's probably more surprising are the 'flowers' you'll find in these fields of sea-grass. These flowers can only be described as primitive soft corals, but they're neither coral nor plant.",
        "photo": "https://res.cloudinary.com/apollographql/image/upload/v1644381344/odyssey/federation-course1/FlyBy%20illustrations/Landscape_9_kbenjj.png"
      },
      {
        "id": "loc-5",
        "name": "Zenn-la",
        "description": "The plant-like organisms on this planet are made up of millions of flowers. Their combined colors and scents make for an amazing spectacle, but they leave little space for other species, which is why there are only very few bush and shrub species. Fungi, grasses and trees are non-existent.",
        "photo": "https://res.cloudinary.com/apollographql/image/upload/v1644381346/odyssey/federation-course1/FlyBy%20illustrations/Landscape_8_zd1e68.png"
      }
    ],
    "latestReviews": [
      {
        "id": "rev-8",
        "comment": "This is simply unbelievable! It's the perfect solution for our business. Really good. I don't always clop, but when I do, it's because of planet",
        "rating": 5
      },
      {
        "id": "rev-9",
        "comment": "Planet is exactly what our business has been lacking. It's incredible. If you want real marketing that works and effective implementation - planet's got you covered.",
        "rating": 5
      },
      {
        "id": "rev-10",
        "comment": "Thanks planet! I was amazed at the quality of planet. Planet did exactly what you said it does.",
        "rating": 5
      }
    ]
  }
}
```

> Fantastic! We can see that the data object in our response contains both locations and reviews.

> This is huge. We've just unlocked one of the powers of our supergraph: we can write one query to our router and hit both subgraphs at once!

# 9. CONNECTING DATA USING ENTITIES

### Overview

> The foundation of our supergraph is complete. We've separated FlyBy's schema into location data and review data, and we've implemented subgraphs that only define the types they're concerned with.

Revisiting our schema agreement checklist, we still have three fields that we don't know how to implement yet:

- `Location.reviewsForLocation` and `Location.overallRating`: These are both fields of the `Location` type, but we decided we want them to be owned by the `reviews` subgraph.
- `Review.location`: This is a field on the `Review` type (which lives in the `reviews` subgraph), but the field has a return type of `Location`, which is defined in the `locations` subgraph.

![Connecting-data-using-entities](./assets/F_01_08_DG_45_v2_wipd1l.png)

To implement these fields, we need to add a new tool to our developer tool belts: entities!

In this lesson, we will:

- Learn what an entity is and what it's used for
- Learn how to define an entity
- Learn how the router represents entities when it talks between subgraphs

### What's an entity?

An _entity_ is an object type with fields split between multiple subgraphs.

This means we can define a type that both of our subgraphs can contribute fields to and resolve independently.

![Connecting-data-using-entities](./assets/F_01_09_DG_entity_tpcr7g.png)

In FlyBy, we want our `reviews` to include the `location` that they are written about. Our `Location` type needs to be used by both subgraphs, so we'll be turning the `Location` type into an _entity_.

A subgraph that defines an _entity_ can do one or both of the following:

1. Reference the entity
2. Contribute fields to the entity

### Reference the entity

Referencing an _entity_ means using it as a return type for another field defined in the subgraph.

For example, in the `reviews` subgraph, we can add a `location` field to the `Review` type, which will reference the `Location` _entity_ as its return type.

![Connecting-data-using-entities](./assets/F_01_09_DG_review-location-return-type.png)

### Contribute fields to the entity

Contributing fields to an entity means that one subgraph adds new fields to an entity that are specific to that subgraph's concerns.

For example, the `Location` entity will have fields for name, description, and photo, which will live in the `locations` subgraph. In other words, the `locations` subgraph contributes these fields to the `Location` entity.

And the `reviews` subgraph contributes two review-specific fields to the `Location` entity: `reviewsForLocation` and `overallRating`.

![Connecting-data-using-entities](./assets/F_01_09_DG_location-entity-fields-enlarged.png)

## How to create an entity

To convert an object into an entity in the subgraph schema, we need to do two things:

1. Define a primary key
2. Define a reference resolver

### Defining a primary key

An entity's primary key is the field (or fields) that can uniquely identify an instance of that entity within a subgraph. The router uses primary keys to collect data from across multiple subgraphs and associate it with a single entity instance.

For example, a location entity's primary key is its id. The router uses that id to collect data about a specific location instance, like a location with id "loc-1".

![Connecting-data-using-entities](./assets/F_01_09_DC_entities_ids_efuu5v.png)

In each of our subgraph schemas, we can define a primary key for an entity, by adding the `@key` directive after the type's name.

The `@key` directive needs a property called `fields`, which we'll set to the field we want to use as the entity's primary key.

![Connecting-data-using-entities](./assets/Entity_syntax.png)

![Connecting-data-using-entities](./assets/F_01_09_DG_entity_syntax_z3jd3u.png)

## Defining a reference resolver function

Each subgraph that contributes fields to an entity also needs to define a special resolver function for that entity called a `reference resolver`. The router uses `reference resolver` to directly access the entity fields that each subgraph contributes.

![Connecting-data-using-entities](./assets/F_01_09_IL_reference-resolver_jz8ypv.png)

Every reference resolver has the name: `__resolveReference`. We define each entity's reference resolver right alongside all the field resolvers for that type.

The `__resolveReference` function has a slightly different signature from other resolver functions. Instead of the usual four arguments, `__resolveReference` only takes three:

- `reference`: The entity representation object that's passed in by the router. This tells the subgraph which instance of an entity is being requested. We'll cover what an entity representation is in the section below.
- `context`: The object shared across all resolvers. (This is the same as in normal resolvers, but note that by convention, we refer to this `__resolveReference` argument as `context`, rather than `contextValue` as in other resolvers!)
- `info`: Contains information about the operation's execution state, just like in a normal resolver. We won't use this argument much.

![Connecting-data-using-entities](./assets/F_01_09_IL_rr-args_cmspgb.png)

Let's focus on this first argument, `reference`, and learn more about entity representations.

### What's an entity representation?

An `entity representation` is an object that the router uses to represent a specific instance of an entity. A representation always includes the `typename` for that entity and the `@key` field for the specific instance.

- The `__typename` field: This field exists on all GraphQL types automatically. It always returns the name of its containing type, as a string. For example, `Location.__typename` returns
  "Location".
- The `@key` field: The key-value pair that a subgraph can use to identify the instance of an entity. For example, if we defined the `Location` entity using the "id" field as a primary key, then our entity representation would include an "id" property with a value like "loc-2".

An entity representation for a location might look like this:

![Connecting-data-using-entities](./assets/Entity_Represantation.png)

You can think of an entity representation as a passport that the router uses to refer to a particular object between subgraphs.

The typename field is like a passport's country of origin. It says which entity the object belongs to. And the `@key` field is like a passport's ID number, uniquely identifying this instance of that entity.

![Connecting-data-using-entities](./assets/F_01_09_105_kz9akz.png)

### Key takeaways

- An entity is a type that can resolve its fields across multiple subgraphs.
- To create an entity, we can use the `@key` directive to specify which field(s) can uniquely identify an object of that type.
- We can use entities in two ways:
  - As a return type for a field (referencing an entity).
  - Defining fields for an entity from multiple subgraphs (contributing to an entity).

* Any subgraph that contributes fields to an entity needs to define a reference resolver function for that entity. This `__resolveReference` resolver is called whenever the router needs to access fields of the entity from within another subgraph.
* An entity representation is an object that the router uses to represent a specific instance of an entity. It includes the entity's type and its key field(s).

# 10. DEFINING AN ENTITY

### Overview

Let's jump into the code!

In this lesson, we will:

- Convert the Location type into an entity that can be shared between our subgraphs
- Publish updates to existing subgraphs

# ✏️ Defining the `Location` entity's `@key`

1.Open up the `subgraph-locations/locations.graphql` file and find the `Location` type.

![Defining-an-entity](./assets/entity_subgraph_locations.png)

2. We'll add the `@key` directive to this type definition, specifying the `fields` property and setting its value to `id`.

![Defining-an-entity](./assets/entity_002.png)

## Check changes in Sandbox

Let's test our changes. Open up http://localhost:4001 in your browser and use Sandbox to query our server again.

In the Documentation tab, we should now see that there's a new field on the Query type called `_entities`. This is a special field that the router uses for coordinating data between subgraphs. We'll learn how exactly the router uses this field in the next lesson.

![Defining-an-entity](./assets/F_01_08_SC_47_v2.png)

Other than that, not much has changed! We should still be able to use Explorer to query the `locations` subgraph as before. (Try running a quick query for some data on the `locations` field, to make sure everything still works as expected.)

### Publish the `locations` subgraph

We'll need to publish this subgraph so that the schema registry can pick up our changes.

Let's make sure we're in the top-level directory of our project when we run the `rover subgraph publish` command in the terminal.

![Defining-an-entity](./assets/rover_publish_subgraph.png)

We can omit the `--routing-url` option in the command because we already set that value the first time we published the subgraph to the registry.

Great, it looks like our changes have been published successfully!

# ✏️ Defining our entity in `reviews`

We want to use the `Location` entity in our `reviews` subgraph as well.

1. Open up the `subgraph-reviews/reviews.graphql` file.
2. We'll add the Location type definition, along with the @key directive and set the id field as the primary key.

![Defining-an-entity](./assets/defining_entity_in_review.png)

3. Inside the curly braces, we'll add the id field of type non-nullable ID!.

![Defining-an-entity](./assets/Difining_review_002.png)

The `Location` entity doesn't need to include all the fields we defined for it in the `locations` subgraph. After all, the `reviews` subgraph doesn't know anything about these fields, or how to resolve them!

So far, we've given our `reviews` subgraph a stub of the `Location` entity. A stub serves as a basic representation of a type that includes just enough information to work with that type in the subgraph.

There's one more change we need to make. Because the `reviews` subgraph is not responsible for resolving any of the entity's other fields, we'll add one more property to our `@key` directive.

4. Inside the `@key` directive, add a property called `resolvable` and set it to `false`.

![Defining-an-entity](./assets/Define_review_003.png)

This property tells the router that this subgraph doesn't define a reference resolver for this entity.

Recall that a reference resolver is responsible for returning all of the entity fields that this subgraph contributes. The `reviews` subgraph doesn't contribute any other fields (besides the key field), so it doesn't need to define a reference resolver. The `resolvable: false` property indicates this to the router!

### Publish the `reviews` subgraph

Now let's publish our `reviews` subgraph updates. From a terminal in the root directory of our project, let's run the `rover subgraph publish` command again.

![Defining-an-entity](./assets/Publish_reviews_subgraph.png)

Alright, we see a success message and our changes have made it to the registry!

### Key takeaways

- To create an entity, we can use the @key directive to specify which field(s) can uniquely identify an object of that type.
- When a subgraph can't be used to resolve any non-@key fields of an entity, we pass resolvable: false to the @key directive definition.

# 11. ENTITIES AND THE QUERY PLAN

### Overview

We've set up our entity definitions, but how exactly does the router use these entities?

In this lesson, we will:

- Learn how the router uses entities and the query plan to connect data from multiple subgraphs
- Learn how entity representations and reference resolvers work together

### Let's look at an example query

Let's say the client makes a request for our app's latest reviews. In their query, they'll request the id, comment, and rating for each review, along with the name of each review's location.

![Entities-and-the-query-plan](./assets/GetLatestReviews.png)
![Entities-and-the-query-plan](./assets/F_01_11_IL_54_client-request_v608za.png)
Now it's the router's time to shine!

### Step 1: Building the query plan

Like we saw earlier, the router begins by building a query plan that indicates which requests to send to which subgraphs.

The router starts with the incoming query's top-level field, `latestReviews`. With the help of the supergraph schema, the router sees that `latestReviews` is defined in the `reviews` subgraph.

So the router starts the query plan with a request to the `reviews` subgraph.

![Entities-and-the-query-plan](./assets/F_01_11_IL_55_reviews-sg_r9hvdb.png)

The router continues like this for a while, checking each field in the query against the supergraph schema, and adding it to the query plan. The fields for `id`, `comment`, `rating` and `location` all belong to the `reviews` subgraph.

![Entities-and-the-query-plan](./assets/F_01_11_IL_56_reviews-sg-b_d3f5lu.png)

But when the router reaches the `name` field for a particular `Location`, it sees from the supergraph schema that `Location.name` can only be resolved by the `locations` subgraph (because that's where the `Location.name` field is defined).

![Entities-and-the-query-plan](./assets/F_01_11_IL_56-location-schema_sa7cgh.png)

That means the router is going to have to connect data between subgraphs.

To do this, the router needs some more information from the `reviews` subgraph: the entity representation for each review's corresponding `Location` object.

Remember that entity representations are what the router uses to track a specific object between subgraphs. To make an entity representation for a `Location` object, the router needs the location's typename and its primary key (which in this case is the `id` field).

![Entities-and-the-query-plan](./assets/F_01_09_105_kz9akz.png)

The router can get both these fields from the `reviews` subgraph.

![Entities-and-the-query-plan](./assets/F_01_11_IL_entity_rep_wrwdny.png)

From there, the router adds another operation to its query plan to request the location's `name` from the `locations` subgraph.

![Entities-and-the-query-plan](./assets/F_01_11_IL_location_name_ho8vy2.png)

With that, all the fields in the query have been accounted for in the query plan. It's time to move on to the next step: executing the plan.

# Step 2: Querying the `reviews` subgraph

The router begins by requesting data from the `reviews` subgraph.

The `reviews` subgraph resolves all the requested fields as it normally would, including the entity representations for all the requested location objects.

![Entities-and-the-query-plan](./assets/F_01_11_IL_58_y33id4.png)

This subgraph doesn't know that the router plans to do anything special with the location's id or typename. It just sends back the data to the router like it was asked.

![Entities-and-the-query-plan](./assets/F_01_10_IL_59_kqsflm.png)

With that, the router's taken care of the first part of the query plan! The next step is to retrieve the `name` field from the `locations` subgraph.

![Entities-and-the-query-plan](./assets/F_01_11_IL_name_next_zwf6k6.png)

# Step 3: Querying the `locations` subgraph

Remember the `_entities` field that showed up in our subgraph after we first defined an entity? This is where it comes back into the story!

The router builds a request using the `_entities` field.

This field takes in an argument called `representations`, which takes in, well, a list of entity representations! This is where the entity representations that the router received from the `reviews` subgraph will go.

In the same request, the router adds the rest of the fields left in the query plan (in this case, the location's `name`).

![Entities-and-the-query-plan](./assets/F_01_11_IL_60_kps963.png)

The router sends this request to the location's subgraph.

To resolve the `_entities` field, the `locations` subgraph uses its reference resolver. Remember this is a special resolver function used to return all the entity fields that this subgraph contributes.

The `locations` subgraph looks at the `__typename` value of each reference object to determine which entity's reference resolver to use. In this case, because typename is "Location", the `locations` subgraph knows to use the `Location` entity's reference resolver.

![Entities-and-the-query-plan](./assets/F_01_11_IL_61_zwdl2p.png)

The `Location` reference resolver runs once for each entity representation in the query. Each time, it uses the entity representation's primary key to return the corresponding `Location` object.

![Entities-and-the-query-plan](./assets/F_01_11_IL_63_vymzl2.png)

After the `locations` subgraph finishes resolving the request, it sends the data back to the router.

![Entities-and-the-query-plan](./assets/F_01_11_IL_64_z5u2if.png)

That's it for the executing phase!

# Step 4: Sending the final response to the client

Now, the router combines all the data it received from the `reviews` and `locations` subgraphs into a single JSON object. And at last, the router sends the final object back to the client.

![Entities-and-the-query-plan](./assets/F_01_11_IL_66_h3sdix.png)

### Key takeaways

- When the router needs to query for fields from a different subgraph, it also asks for entity representations from the current subgraph it's querying. These representations will be used in the subsequent operation's \_entities field, set as the value for the representations argument.
- The reference resolver takes each representation and returns the matching data for its requested fields.

# 12. REFERENCING AN ENTITY

### Overview

Let's get back to our problem of how to connect data between subgraphs. We'll take care of one of the missing pieces of our schema: the `Review.location` field.

In this lesson, we will:

Learn how to reference an entity in a subgraph as a return type by implementing the `Review.location` field

![Referencing-an-entity](./assets/12-referencing-an-entity/review-location-under-construction-v2-recolor.svg)

# The `Location` entity as a return type

We want to use the `Location` entity as the return type for the `Review.location` field, so let's take a closer look at how to do that.

### ✏️ Adding the `Review.location` field to the schema

1. Open up the `subgraph-reviews/reviews.graphql` file.

2. Let's add a new field called `location`, which should return a `Location` type.

![Referencing-an-entity](./assets/12-referencing-an-entity/12_code_01.png)

3. We can test our changes and open Sandbox for the `reviews` subgraph at http://localhost:4002. We should see the new `location` field show up under `latestReviews`.

![Referencing-an-entity](./assets/12-referencing-an-entity/F_01_10_SC_53.png)

4. Let's try running a query to test out our new field. We'll query for `latestReviews`, and include the `id`, `comment` and `rating` fields. Next, we'll include the new `location` field and its `id`. Let's also give the operation a descriptive name: `GetLatestReviewsAndLocations`.

![Referencing-an-entity](./assets/12-referencing-an-entity/12_02_code.png)

When we submit the query, we can see that we get back `null` for the value of each `location`!

```json {
  "data": {
    "latestReviews": [
      {
        "id": "rev-8",
        "comment": "This is simply unbelievable! It's the perfect solution for our business. Really good. I don't always clop, but when I do, it's because of planet",
        "rating": 5,
        "location": null
      },
      {
        "id": "rev-9",
        "comment": "Planet is exactly what our business has been lacking. It's incredible. If you want real marketing that works and effective implementation - planet's got you covered.",
        "rating": 5,
        "location": null
      },
      {
        "id": "rev-10",
        "comment": "Thanks planet! I was amazed at the quality of planet. Planet did exactly what you said it does.",
        "rating": 5,
        "location": null
      }
    ]
  }
}
```

This is because we haven't defined what the `reviews` subgraph should return when this field is queried! We first need to define a corresponding resolver function.

### ✏️ The `Review.location` resolver function

As we saw before, the router will ask the `reviews` subgraph for an entity representation of the location the review is associated with. The router already knows how to retrieve the typename, but it needs the location's id key field. Let's set that up.

1. Open up the `subgraph-reviews/resolvers.js` file.

2. In the `resolvers` object, we'll add a new key for the `Review` type, and an empty resolver function for the `location` field.

![Referencing-an-entity](./assets/12-referencing-an-entity/12_03_code.png)

3. Name the first parameter of the resolver function `review`, which is the `parent` object of the field.

![Referencing-an-entity](./assets/12-referencing-an-entity/12_04_code.png)

4. For the body of the resolver, we need to return an entity representation with the location's id. So how do we retrieve the id of a location for a particular review?

To answer this question, we'll take a quick detour to look at what reviews data we get back from our data source. Jump over to the `reviews_data.json` file in the `datasources` directory. Here we can see that for each review object, we are storing the `locationId` each review belongs to.

![Referencing-an-entity](./assets/12-referencing-an-entity/12_05_code.png)

This `locationId` field specifies exactly the data we're looking for - a location's `id`!

Back in the `Reviews.location` resolver, let's destructure the `review` object and pull out `locationId`. Then we'll return a new object that reassigns `locationId` to `id`. This will match it to the name of the `Location` entity's `@key` field.

![Referencing-an-entity](./assets/12-referencing-an-entity/12_06_code.png)

### Checking your work

Fantastic! Now let's check that everything's playing nicely. Go back to Apollo Sandbox for the `reviews` subgraph at http://localhost:4002.

Let's try out that query again. This time, we get back each location's id!

![Referencing-an-entity](./assets/12-referencing-an-entity/12_07_code.png)

The response should match the shape of the object below:

```json
{
  "data": {
    "latestReviews": [
      {
        "id": "rev-8",
        "comment": "This is simply unbelievable! It's the perfect solution for our business. Really good. I don't always clop, but when I do, it's because of planet",
        "rating": 5,
        "location": {
          "id": "loc-2"
        }
      },
      {
        "id": "rev-9",
        "comment": "Planet is exactly what our business has been lacking. It's incredible. If you want real marketing that works and effective implementation - planet's got you covered.",
        "rating": 5,
        "location": {
          "id": "loc-3"
        }
      },
      {
        "id": "rev-10",
        "comment": "Thanks planet! I was amazed at the quality of planet. Planet did exactly what you said it does.",
        "rating": 5,
        "location": {
          "id": "loc-4"
        }
      }
    ]
  }
}
```

And now our `reviews` subgraph can resolve a location's `id` field, which is exactly what the router will need to associate data across subgraphs.

To resolve the rest of the `Location` fields (like `name`, `description`, or `photo`), we still have one thing left to add to our schema: the `Location` entity's reference resolver!

### ✏️ Implement the `__resolveReference` resolver

1. Moving over to the `subgraph-locations` directory, open up the `resolvers.js` file.

2. Inside the `resolvers` object, add a new key for `Location`, then a resolver function called `__resolveReference`.

![Referencing-an-entity](./assets/12-referencing-an-entity/12_08_code.png)

3. Next, let's set up this function's arguments.

Destructure the first argument, which is the entity representation object, and pull out the `id` field from it.

Similarly, destructure the second argument (`context`) to access the `dataSources` property.

![Referencing-an-entity](./assets/12-referencing-an-entity/12_09_code.png)

4. The body of the reference resolver function needs to return all the entity fields that this subgraph defines. To do this, we'll use the `LocationsAPI` data source and its `getLocation` method. It returns a `Location` object for a given ID.

![Referencing-an-entity](./assets/12-referencing-an-entity/12_010_code.png)

> Note: You can check out how the `getLocation` method works by peeking inside the `subgraph-locations/datasources/LocationsApi.js` file.

See it in action!

Now that our `__resolveReference` resolver is set up, we can test it out by pretending to be the router and mimicking the request it sends to the `locations` subgraph!

> Note: You aren't required to do this step, as it's a bit advanced. But if you're interested in seeing how things work under the hood, then proceed!

With our `locations` server running, open http://localhost:4001 in Apollo Sandbox.

Add the following query variable in the Variables pane, to imitate the reference objects the router would pass in from the `reviews` subgraph:

```json
{
  "representations": [
    {
      "__typename": "Location",
      "id": "loc-1"
    },
    {
      "__typename": "Location",
      "id": "loc-2"
    }
  ]
}
```

Then run the following query:

```graphql
query GetLocationNames($representations: [_Any!]!) {
  _entities(representations: $representations) {
    ... on Location {
      name
    }
  }
}
```

The response should look something like the object below.

```json
{
  "data": {
    "_entities": [
      {
        "name": "The Living Ocean of New Lemuria"
      },
      {
        "name": "Vinci"
      }
    ]
  }
}
```

> Note: As you can see, the `Query._entities` field is quite powerful and can be used for malicious reasons (anyone can mimic a request and retrieve information they aren't supposed to). For this reason, your subgraphs should not be directly accessible by clients. See the docs for more information. https://www.apollographql.com/docs/apollo-server/using-federation/apollo-subgraph-setup/#securing-your-subgraphs

And with that, our graph is now fully set up to handle referencing entities!

Okay, we should be ready to query our supergraph in Studio, and watch the magic of the router associating data between our subgraphs!

Let's get to building the `GetLatestReviews` query we agreed upon earlier with the frontend team. We'll add our fields... wait a minute, where did our `location` field go? Wasn't this working great locally on Sandbox? What happened?

We forgot to publish our `reviews` subgraph schema changes to the registry!

### ✏️ Publish subgraph change with Rover

Oops! Let's hop over to a terminal in the root of the project, and run `rover subgraph publish`, passing in the variables for the `reviews` subgraph.

![Referencing-an-entity](./assets/12-referencing-an-entity/12_011_code.png)

Now we should be ready to query our supergraph in Studio, and watch the magic of the router associating data between our subgraphs! ✨

### ✏️ Check your work against the router

Let's run this query in Studio.

![Referencing-an-entity](./assets/12-referencing-an-entity/12_012_code.png)

Note: If you see red squiggly lines below the `location` field on `latestReviews`, try refreshing the page. You may have been faster than the supergraph composition!

And we can see all our data is coming back from both the `locations` and `reviews` subgraphs!

We should see a response like this:

```json
{
  "data": {
    "latestReviews": [
      {
        "id": "rev-8",
        "comment": "This is simply unbelievable! It's the perfect solution for our business. Really good. I don't always clop, but when I do, it's because of planet",
        "rating": 5,
        "location": {
          "name": "Vinci"
        }
      },
      {
        "id": "rev-9",
        "comment": "Planet is exactly what our business has been lacking. It's incredible. If you want real marketing that works and effective implementation - planet's got you covered.",
        "rating": 5,
        "location": {
          "name": "Asteroid B-612"
        }
      },
      {
        "id": "rev-10",
        "comment": "Thanks planet! I was amazed at the quality of planet. Planet did exactly what you said it does.",
        "rating": 5,
        "location": {
          "name": "Krypton"
        }
      }
    ]
  }
}
```

Let's update our schema agreement checklist and check off the location field we just added to the Review type.

![Referencing-an-entity](./assets/12-referencing-an-entity/F_01_10_DG_122-recolor.svg)

### Key takeaways

- We can reference an entity in one subgraph as the return value for a type's field.
- Any subgraph that contributes fields to an entity needs to define a `__resolveReference` resolver function for that entity. This resolver is called when the router needs to resolve references to that entity made from within other subgraphs.
