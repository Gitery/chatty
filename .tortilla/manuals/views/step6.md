# Step 6: GraphQL Subscriptions

[//]: # (head-end)


This is the sixth blog in a multipart series where we will be building Chatty, a WhatsApp clone, using [React Native](https://facebook.github.io/react-native/) and [Apollo](http://dev.apollodata.com/).

In this tutorial, we’ll focus on adding [GraphQL Subscriptions](http://graphql.org/blog/subscriptions-in-graphql-and-relay/), which will give our app real-time instant messaging capabilities!

Here’s what we will accomplish in this tutorial:
1. Introduce Event-based Subscriptions
2. Build server-side infrastructure to handle **GraphQL Subscriptions** via WebSockets
3. Design GraphQL Subscriptions and add them to our GraphQL Schemas and Resolvers
4. Build client-side infrastructure to handle GraphQL Subscriptions via WebSockets
5. Subscribe to GraphQL Subscriptions on our React Native client and handle real-time updates

# Event-based Subscriptions
Real-time capable apps need a way to be pushed data from the server. In some real-time architectures, all data is considered live data, and anytime data changes on the server, it’s pushed through a WebSocket or long-polling and updated on the client. While this sort of architecture means we can expect data to update on the client without writing extra code, it starts to get tricky and non-performant as apps scale. For one thing, you don’t need to keep track of every last bit of data if it’s not relevant to the user. Moreover, it’s not obvious what changes to data should trigger an event, what that event should look like, and how our clients should react.

With an event-based subscription model in GraphQL — much like with queries and mutations — a client can tell the server exactly what data it wants to be pushed and what that data should look like. This leads to fewer events tracked on the server and pushed to the client, and precise event handling on both ends!

# GraphQL Subscriptions on the Server
It’s probably easiest to think about our event based subscriptions setup from the client’s perspective. All queries and mutations will still get executed with standard HTTP requests. This will keep request execution more reliable and the WebSocket connection unclogged. We will only use WebSockets for subscriptions, and the client will only subscribe to events it cares about — the ones that affect stuff for the current user.

## Designing GraphQL Subscriptions
Let’s focus on the most important event that ever happens within a messaging app — getting a new message.

When someone creates a new message, we want all group members to be notified that a new message was created. We don’t want our users to know about every new message being created by everybody on Chatty, so we’ll create a system where users **subscribe** to new message events just for their own groups. We can build this subscription model right into our GraphQL Schema!

Let’s modify our GraphQL Schema in `server/data/schema.js` to include a **GraphQL Subscription** for when new messages are added to a group we care about:

[{]: <helper> (diffStep 6.1)

#### Step 6.1: Add Subscription to Schema

##### Changed server&#x2F;data&#x2F;schema.js
```diff
@@ -67,10 +67,17 @@
 ┊67┊67┊    leaveGroup(id: Int!, userId: Int!): Group # let user leave group
 ┊68┊68┊    updateGroup(id: Int!, name: String): Group
 ┊69┊69┊  }
+┊  ┊70┊
+┊  ┊71┊  type Subscription {
+┊  ┊72┊    # Subscription fires on every message added
+┊  ┊73┊    # for any of the groups with one of these groupIds
+┊  ┊74┊    messageAdded(userId: Int, groupIds: [Int]): Message
+┊  ┊75┊  }
 ┊70┊76┊  
 ┊71┊77┊  schema {
 ┊72┊78┊    query: Query
 ┊73┊79┊    mutation: Mutation
+┊  ┊80┊    subscription: Subscription
 ┊74┊81┊  }
 ┊75┊82┊`];
```

[}]: #

That’s it!

## GraphQL Subscription Infrastructure
Our Schema uses GraphQL Subscriptions, but our server infrastructure has no way to handle them.

We will use two excellent packages from the Apollo team  — [` subscription-transport-ws`](https://www.npmjs.com/package/subscriptions-transport-ws) and [`graphql-subscriptions`](https://www.npmjs.com/package/graphql-subscriptions)  —  to hook up our GraphQL server with subscription capabilities:
```
yarn add graphql-subscriptions subscriptions-transport-ws
```

First, we’ll use `graphql-subscriptions` to create a `PubSub` manager. `PubSub` is basically just event emitters wrapped with a function that filters messages. It can easily be replaced later with something more advanced like [`graphql-redis-subscriptions`](https://github.com/davidyaha/graphql-redis-subscriptions).

Let’s create a new file `server/subscriptions.js` where we’ll start fleshing out our subscription infrastructure:

[{]: <helper> (diffStep 6.2 files="server/subscriptions.js")

#### Step 6.2: Create subscriptions.js

##### Added server&#x2F;subscriptions.js
```diff
@@ -0,0 +1,5 @@
+┊ ┊1┊import { PubSub } from 'graphql-subscriptions';
+┊ ┊2┊
+┊ ┊3┊export const pubsub = new PubSub();
+┊ ┊4┊
+┊ ┊5┊export default pubsub;
```

[}]: #

We're going to need the same `executableSchema` we created in `server/index.js`, so let’s pull out executableSchema from `server/index.js` and put it inside `server/data/schema.js` so other files can use `executableSchema`.

[{]: <helper> (diffStep 6.3)

#### Step 6.3: Refactor schema.js to export executableSchema

##### Changed server&#x2F;data&#x2F;schema.js
```diff
@@ -1,3 +1,8 @@
+┊ ┊1┊import { addMockFunctionsToSchema, makeExecutableSchema } from 'graphql-tools';
+┊ ┊2┊
+┊ ┊3┊import { Mocks } from './mocks';
+┊ ┊4┊import { Resolvers } from './resolvers';
+┊ ┊5┊
 ┊1┊6┊export const Schema = [`
 ┊2┊7┊  # declare custom scalars
 ┊3┊8┊  scalar Date
```
```diff
@@ -81,4 +86,15 @@
 ┊ 81┊ 86┊  }
 ┊ 82┊ 87┊`];
 ┊ 83┊ 88┊
-┊ 84┊   ┊export default Schema;
+┊   ┊ 89┊export const executableSchema = makeExecutableSchema({
+┊   ┊ 90┊  typeDefs: Schema,
+┊   ┊ 91┊  resolvers: Resolvers,
+┊   ┊ 92┊});
+┊   ┊ 93┊
+┊   ┊ 94┊// addMockFunctionsToSchema({
+┊   ┊ 95┊//   schema: executableSchema,
+┊   ┊ 96┊//   mocks: Mocks,
+┊   ┊ 97┊//   preserveResolvers: true,
+┊   ┊ 98┊// });
+┊   ┊ 99┊
+┊   ┊100┊export default executableSchema;
```

##### Changed server&#x2F;index.js
```diff
@@ -1,29 +1,13 @@
 ┊ 1┊ 1┊import express from 'express';
 ┊ 2┊ 2┊import { graphqlExpress, graphiqlExpress } from 'apollo-server-express';
-┊ 3┊  ┊import { makeExecutableSchema, addMockFunctionsToSchema } from 'graphql-tools';
 ┊ 4┊ 3┊import bodyParser from 'body-parser';
 ┊ 5┊ 4┊import { createServer } from 'http';
 ┊ 6┊ 5┊
-┊ 7┊  ┊import { Resolvers } from './data/resolvers';
-┊ 8┊  ┊import { Schema } from './data/schema';
-┊ 9┊  ┊import { Mocks } from './data/mocks';
+┊  ┊ 6┊import { executableSchema } from './data/schema';
 ┊10┊ 7┊
 ┊11┊ 8┊const GRAPHQL_PORT = 8080;
 ┊12┊ 9┊const app = express();
 ┊13┊10┊
-┊14┊  ┊const executableSchema = makeExecutableSchema({
-┊15┊  ┊  typeDefs: Schema,
-┊16┊  ┊  resolvers: Resolvers,
-┊17┊  ┊});
-┊18┊  ┊
-┊19┊  ┊// we can comment out this code for mocking data
-┊20┊  ┊// we're using REAL DATA now!
-┊21┊  ┊// addMockFunctionsToSchema({
-┊22┊  ┊//   schema: executableSchema,
-┊23┊  ┊//   mocks: Mocks,
-┊24┊  ┊//   preserveResolvers: true,
-┊25┊  ┊// });
-┊26┊  ┊
 ┊27┊11┊// `context` must be an object and can't be undefined when using connectors
 ┊28┊12┊app.use('/graphql', bodyParser.json(), graphqlExpress({
 ┊29┊13┊  schema: executableSchema,
```

[}]: #

Now that we’ve created a `PubSub`, we can use this class to publish and subscribe to events as they occur in our Resolvers.

We can modify `server/data/resolvers.js` as follows:

[{]: <helper> (diffStep 6.4)

#### Step 6.4: Add Subscription to Resolvers

##### Changed server&#x2F;data&#x2F;resolvers.js
```diff
@@ -1,5 +1,8 @@
 ┊1┊1┊import GraphQLDate from 'graphql-date';
 ┊2┊2┊import { Group, Message, User } from './connectors';
+┊ ┊3┊import { pubsub } from '../subscriptions';
+┊ ┊4┊
+┊ ┊5┊const MESSAGE_ADDED_TOPIC = 'messageAdded';
 ┊3┊6┊
 ┊4┊7┊export const Resolvers = {
 ┊5┊8┊  Date: GraphQLDate,
```
```diff
@@ -32,6 +35,10 @@
 ┊32┊35┊        userId,
 ┊33┊36┊        text,
 ┊34┊37┊        groupId,
+┊  ┊38┊      }).then((message) => {
+┊  ┊39┊        // publish subscription notification with the whole message
+┊  ┊40┊        pubsub.publish(MESSAGE_ADDED_TOPIC, { [MESSAGE_ADDED_TOPIC]: message });
+┊  ┊41┊        return message;
 ┊35┊42┊      });
 ┊36┊43┊    },
 ┊37┊44┊    createGroup(_, { name, userIds, userId }) {
```
```diff
@@ -73,6 +80,12 @@
 ┊73┊80┊        .then(group => group.update({ name }));
 ┊74┊81┊    },
 ┊75┊82┊  },
+┊  ┊83┊  Subscription: {
+┊  ┊84┊    messageAdded: {
+┊  ┊85┊      // the subscription payload is the message.
+┊  ┊86┊      subscribe: () => pubsub.asyncIterator(MESSAGE_ADDED_TOPIC),
+┊  ┊87┊    },
+┊  ┊88┊  },
 ┊76┊89┊  Group: {
 ┊77┊90┊    users(group) {
 ┊78┊91┊      return group.getUsers();
```

[}]: #

Whenever a user creates a message, we trigger `pubsub` to publish the `messageAdded` event along with the newly created message. `PubSub` will emit an event to any clients subscribed to `messageAdded` and pass them the new message.

But we only want to emit this event to clients who care about the message because it was sent to one of their user’s groups! We can modify our implementation to filter who gets the event emission:

[{]: <helper> (diffStep 6.5)

#### Step 6.5: Add withFilter to messageAdded

##### Changed server&#x2F;data&#x2F;resolvers.js
```diff
@@ -1,4 +1,6 @@
 ┊1┊1┊import GraphQLDate from 'graphql-date';
+┊ ┊2┊import { withFilter } from 'graphql-subscriptions';
+┊ ┊3┊
 ┊2┊4┊import { Group, Message, User } from './connectors';
 ┊3┊5┊import { pubsub } from '../subscriptions';
 ┊4┊6┊
```
```diff
@@ -82,8 +84,16 @@
 ┊82┊84┊  },
 ┊83┊85┊  Subscription: {
 ┊84┊86┊    messageAdded: {
-┊85┊  ┊      // the subscription payload is the message.
-┊86┊  ┊      subscribe: () => pubsub.asyncIterator(MESSAGE_ADDED_TOPIC),
+┊  ┊87┊      subscribe: withFilter(
+┊  ┊88┊        () => pubsub.asyncIterator(MESSAGE_ADDED_TOPIC),
+┊  ┊89┊        (payload, args) => {
+┊  ┊90┊          return Boolean(
+┊  ┊91┊            args.groupIds &&
+┊  ┊92┊            ~args.groupIds.indexOf(payload.messageAdded.groupId) &&
+┊  ┊93┊            args.userId !== payload.messageAdded.userId, // don't send to user creating message
+┊  ┊94┊          );
+┊  ┊95┊        },
+┊  ┊96┊      ),
 ┊87┊97┊    },
 ┊88┊98┊  },
 ┊89┊99┊  Group: {
```

[}]: #

Using `withFilter`, we create a `filter` which returns true when the `groupId` of a new message matches one of the `groupIds` passed into our `messageAdded` subscription. This filter will be applied whenever `pubsub.publish(MESSAGE_ADDED_TOPIC, { [MESSAGE_ADDED_TOPIC]: message })` is triggered, and only clients whose subscriptions pass the filter will receive the message.

Our Resolvers are all set up. Time to hook our server up to WebSockets!

## Creating the SubscriptionServer
Our server will serve subscriptions via WebSockets, keeping an open connection with clients. `subscription-transport-ws` exposes a `SubscriptionServer` module that, when given a server, an endpoint, and the `execute` and `subscribe` modules from `graphql`, will tie everything together. The `SubscriptionServer` will rely on the Resolvers to manage emitting events to subscribed clients over the endpoint via WebSockets. How cool is that?!

Inside `server/index.js`, let’s attach a new `SubscriptionServer` to our current server and have it use `ws://localhost:8080/subscriptions` (`SUBSCRIPTIONS_PATH`) as our subscription endpoint:

[{]: <helper> (diffStep 6.6)

#### Step 6.6: Create SubscriptionServer

##### Changed server&#x2F;index.js
```diff
@@ -2,10 +2,15 @@
 ┊ 2┊ 2┊import { graphqlExpress, graphiqlExpress } from 'apollo-server-express';
 ┊ 3┊ 3┊import bodyParser from 'body-parser';
 ┊ 4┊ 4┊import { createServer } from 'http';
+┊  ┊ 5┊import { SubscriptionServer } from 'subscriptions-transport-ws';
+┊  ┊ 6┊import { execute, subscribe } from 'graphql';
 ┊ 5┊ 7┊
 ┊ 6┊ 8┊import { executableSchema } from './data/schema';
 ┊ 7┊ 9┊
 ┊ 8┊10┊const GRAPHQL_PORT = 8080;
+┊  ┊11┊const GRAPHQL_PATH = '/graphql';
+┊  ┊12┊const SUBSCRIPTIONS_PATH = '/subscriptions';
+┊  ┊13┊
 ┊ 9┊14┊const app = express();
 ┊10┊15┊
 ┊11┊16┊// `context` must be an object and can't be undefined when using connectors
```
```diff
@@ -15,9 +20,23 @@
 ┊15┊20┊}));
 ┊16┊21┊
 ┊17┊22┊app.use('/graphiql', graphiqlExpress({
-┊18┊  ┊  endpointURL: '/graphql',
+┊  ┊23┊  endpointURL: GRAPHQL_PATH,
+┊  ┊24┊  subscriptionsEndpoint: `ws://localhost:${GRAPHQL_PORT}${SUBSCRIPTIONS_PATH}`,
 ┊19┊25┊}));
 ┊20┊26┊
 ┊21┊27┊const graphQLServer = createServer(app);
 ┊22┊28┊
-┊23┊  ┊graphQLServer.listen(GRAPHQL_PORT, () => console.log(`GraphQL Server is now running on http://localhost:${GRAPHQL_PORT}/graphql`));
+┊  ┊29┊graphQLServer.listen(GRAPHQL_PORT, () => {
+┊  ┊30┊  console.log(`GraphQL Server is now running on http://localhost:${GRAPHQL_PORT}${GRAPHQL_PATH}`);
+┊  ┊31┊  console.log(`GraphQL Subscriptions are now running on ws://localhost:${GRAPHQL_PORT}${SUBSCRIPTIONS_PATH}`);
+┊  ┊32┊});
+┊  ┊33┊
+┊  ┊34┊// eslint-disable-next-line no-unused-vars
+┊  ┊35┊const subscriptionServer = SubscriptionServer.create({
+┊  ┊36┊  schema: executableSchema,
+┊  ┊37┊  execute,
+┊  ┊38┊  subscribe,
+┊  ┊39┊}, {
+┊  ┊40┊  server: graphQLServer,
+┊  ┊41┊  path: SUBSCRIPTIONS_PATH,
+┊  ┊42┊});
```

[}]: #

You might have noticed that we also updated our `/graphiql` endpoint to include a subscriptionsEndpoint. That’s right  —  we can track our subscriptions in GraphIQL!

A GraphQL Subscription is written on the client much like a query or mutation. For example, in GraphIQL, we could write the following GraphQL Subscription for `messageAdded`:
```
subscription messageAdded($groupIds: [Int]){
  messageAdded(groupIds: $groupIds) {
    id
    to {
      name
    }
    from {
      username
    }
    text
  }
}
```

Let’s check out GraphIQL and see if everything works: ![GraphIQL Gif](https://s3-us-west-1.amazonaws.com/tortilla/chatty/step6-6.gif)

## New Subscription Workflow
We’ve successfully set up GraphQL Subscriptions on our server.

Since we have the infrastructure in place, let’s add one more subscription for some extra practice. We can use the same methodology we used for subscribing to new `Messages` and apply it to new `Groups`. After all, it’s important that our users know right away that they’ve been added to a new group.

The steps are as follows:
1. Add the subscription to our Schema:

[{]: <helper> (diffStep 6.7)

#### Step 6.7: Add groupAdded to Schema

##### Changed server&#x2F;data&#x2F;schema.js
```diff
@@ -77,6 +77,7 @@
 ┊77┊77┊    # Subscription fires on every message added
 ┊78┊78┊    # for any of the groups with one of these groupIds
 ┊79┊79┊    messageAdded(userId: Int, groupIds: [Int]): Message
+┊  ┊80┊    groupAdded(userId: Int): Group
 ┊80┊81┊  }
 ┊81┊82┊  
 ┊82┊83┊  schema {
```

[}]: #

2. Publish to the subscription when a new `Group` is created and resolve the subscription in the Resolvers:

[{]: <helper> (diffStep 6.8)

#### Step 6.8: Add groupAdded to Resolvers

##### Changed server&#x2F;data&#x2F;resolvers.js
```diff
@@ -5,6 +5,7 @@
 ┊ 5┊ 5┊import { pubsub } from '../subscriptions';
 ┊ 6┊ 6┊
 ┊ 7┊ 7┊const MESSAGE_ADDED_TOPIC = 'messageAdded';
+┊  ┊ 8┊const GROUP_ADDED_TOPIC = 'groupAdded';
 ┊ 8┊ 9┊
 ┊ 9┊10┊export const Resolvers = {
 ┊10┊11┊  Date: GraphQLDate,
```
```diff
@@ -51,8 +52,13 @@
 ┊51┊52┊            users: [user, ...friends],
 ┊52┊53┊          })
 ┊53┊54┊            .then(group => group.addUsers([user, ...friends])
-┊54┊  ┊              .then(() => group),
-┊55┊  ┊            ),
+┊  ┊55┊              .then((res) => {
+┊  ┊56┊                // append the user list to the group object
+┊  ┊57┊                // to pass to pubsub so we can check members
+┊  ┊58┊                group.users = [user, ...friends];
+┊  ┊59┊                pubsub.publish(GROUP_ADDED_TOPIC, { [GROUP_ADDED_TOPIC]: group });
+┊  ┊60┊                return group;
+┊  ┊61┊              })),
 ┊56┊62┊          ),
 ┊57┊63┊        );
 ┊58┊64┊    },
```
```diff
@@ -95,6 +101,9 @@
 ┊ 95┊101┊        },
 ┊ 96┊102┊      ),
 ┊ 97┊103┊    },
+┊   ┊104┊    groupAdded: {
+┊   ┊105┊      subscribe: () => pubsub.asyncIterator(GROUP_ADDED_TOPIC),
+┊   ┊106┊    },
 ┊ 98┊107┊  },
 ┊ 99┊108┊  Group: {
 ┊100┊109┊    users(group) {
```

[}]: #

3. Filter the recipients of the emitted new group with `withFilter`:

[{]: <helper> (diffStep 6.9)

#### Step 6.9: Add withFilter to groupAdded

##### Changed server&#x2F;data&#x2F;resolvers.js
```diff
@@ -1,5 +1,6 @@
 ┊1┊1┊import GraphQLDate from 'graphql-date';
 ┊2┊2┊import { withFilter } from 'graphql-subscriptions';
+┊ ┊3┊import { map } from 'lodash';
 ┊3┊4┊
 ┊4┊5┊import { Group, Message, User } from './connectors';
 ┊5┊6┊import { pubsub } from '../subscriptions';
```
```diff
@@ -102,7 +103,16 @@
 ┊102┊103┊      ),
 ┊103┊104┊    },
 ┊104┊105┊    groupAdded: {
-┊105┊   ┊      subscribe: () => pubsub.asyncIterator(GROUP_ADDED_TOPIC),
+┊   ┊106┊      subscribe: withFilter(
+┊   ┊107┊        () => pubsub.asyncIterator(GROUP_ADDED_TOPIC),
+┊   ┊108┊        (payload, args) => {
+┊   ┊109┊          return Boolean(
+┊   ┊110┊            args.userId &&
+┊   ┊111┊            ~map(payload.groupAdded.users, 'id').indexOf(args.userId) &&
+┊   ┊112┊            args.userId !== payload.groupAdded.users[0].id, // don't send to user creating group
+┊   ┊113┊          );
+┊   ┊114┊        },
+┊   ┊115┊      ),
 ┊106┊116┊    },
 ┊107┊117┊  },
 ┊108┊118┊  Group: {
```

[}]: #

All set!

# GraphQL Subscriptions on the Client
Time to add subscriptions inside our React Native client. We’ll start by adding a few packages to our client:
```
# make sure you're adding the package in the client!!!
cd client
yarn add apollo-link-ws apollo-utilities subscriptions-transport-ws
```

We’ll use `subscription-transport-ws` on the client to create a WebSocket client connected to our `/subscriptions` endpoint. We then add this client into our Apollo workflow via `apollo-link-ws`. We need to split the various operations that flow through Apollo into `subscription` operations and non-subscription (`query` and `mutation`) and handle them with the appropriate links:

[{]: <helper> (diffStep "6.10" files="client/src/app.js")

#### Step 6.10: Add wsClient to networkInterface

##### Changed client&#x2F;src&#x2F;app.js
```diff
@@ -9,6 +9,9 @@
 ┊ 9┊ 9┊import { Provider } from 'react-redux';
 ┊10┊10┊import { ReduxCache, apolloReducer } from 'apollo-cache-redux';
 ┊11┊11┊import ReduxLink from 'apollo-link-redux';
+┊  ┊12┊import { WebSocketLink } from 'apollo-link-ws';
+┊  ┊13┊import { getMainDefinition } from 'apollo-utilities';
+┊  ┊14┊import { SubscriptionClient } from 'subscriptions-transport-ws';
 ┊12┊15┊
 ┊13┊16┊import AppWithNavigationState, {
 ┊14┊17┊  navigationReducer,
```
```diff
@@ -34,9 +37,32 @@
 ┊34┊37┊
 ┊35┊38┊const httpLink = createHttpLink({ uri: `http://${URL}/graphql` });
 ┊36┊39┊
+┊  ┊40┊// Create WebSocket client
+┊  ┊41┊export const wsClient = new SubscriptionClient(`ws://${URL}/subscriptions`, {
+┊  ┊42┊  reconnect: true,
+┊  ┊43┊  connectionParams: {
+┊  ┊44┊    // Pass any arguments you want for initialization
+┊  ┊45┊  },
+┊  ┊46┊});
+┊  ┊47┊
+┊  ┊48┊const webSocketLink = new WebSocketLink(wsClient);
+┊  ┊49┊
+┊  ┊50┊const requestLink = ({ queryOrMutationLink, subscriptionLink }) =>
+┊  ┊51┊  ApolloLink.split(
+┊  ┊52┊    ({ query }) => {
+┊  ┊53┊      const { kind, operation } = getMainDefinition(query);
+┊  ┊54┊      return kind === 'OperationDefinition' && operation === 'subscription';
+┊  ┊55┊    },
+┊  ┊56┊    subscriptionLink,
+┊  ┊57┊    queryOrMutationLink,
+┊  ┊58┊  );
+┊  ┊59┊
 ┊37┊60┊const link = ApolloLink.from([
 ┊38┊61┊  reduxLink,
-┊39┊  ┊  httpLink,
+┊  ┊62┊  requestLink({
+┊  ┊63┊    queryOrMutationLink: httpLink,
+┊  ┊64┊    subscriptionLink: webSocketLink,
+┊  ┊65┊  }),
 ┊40┊66┊]);
 ┊41┊67┊
 ┊42┊68┊export const client = new ApolloClient({
```

[}]: #

That’s it — we’re ready to start adding subscriptions!

# Designing GraphQL Subscriptions
Our GraphQL Subscriptions are going to be ridiculously easy to write now that we’ve had practice with queries and mutations. We’ll first write our `messageAdded` subscription in a new file `client/src/graphql/message-added.subscription.js`:

[{]: <helper> (diffStep 6.11)

#### Step 6.11: Create MESSAGE_ADDED_SUBSCRIPTION

##### Added client&#x2F;src&#x2F;graphql&#x2F;message-added.subscription.js
```diff
@@ -0,0 +1,14 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊
+┊  ┊ 3┊import MESSAGE_FRAGMENT from './message.fragment';
+┊  ┊ 4┊
+┊  ┊ 5┊const MESSAGE_ADDED_SUBSCRIPTION = gql`
+┊  ┊ 6┊  subscription onMessageAdded($userId: Int, $groupIds: [Int]){
+┊  ┊ 7┊    messageAdded(userId: $userId, groupIds: $groupIds){
+┊  ┊ 8┊      ... MessageFragment
+┊  ┊ 9┊    }
+┊  ┊10┊  }
+┊  ┊11┊  ${MESSAGE_FRAGMENT}
+┊  ┊12┊`;
+┊  ┊13┊
+┊  ┊14┊export default MESSAGE_ADDED_SUBSCRIPTION;
```

[}]: #

I’ve retitled the subscription `onMessageAdded` to distinguish the name from the subscription itself.

The `groupAdded` component will look extremely similar:

[{]: <helper> (diffStep 6.12)

#### Step 6.12: Create GROUP_ADDED_SUBSCRIPTION

##### Added client&#x2F;src&#x2F;graphql&#x2F;group-added.subscription.js
```diff
@@ -0,0 +1,23 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊
+┊  ┊ 3┊import MESSAGE_FRAGMENT from './message.fragment';
+┊  ┊ 4┊
+┊  ┊ 5┊const GROUP_ADDED_SUBSCRIPTION = gql`
+┊  ┊ 6┊  subscription onGroupAdded($userId: Int){
+┊  ┊ 7┊    groupAdded(userId: $userId){
+┊  ┊ 8┊      id
+┊  ┊ 9┊      name
+┊  ┊10┊      messages(first: 1) {
+┊  ┊11┊        edges {
+┊  ┊12┊          cursor
+┊  ┊13┊          node {
+┊  ┊14┊            ... MessageFragment
+┊  ┊15┊          }
+┊  ┊16┊        }
+┊  ┊17┊      }
+┊  ┊18┊    }
+┊  ┊19┊  }
+┊  ┊20┊  ${MESSAGE_FRAGMENT}
+┊  ┊21┊`;
+┊  ┊22┊
+┊  ┊23┊export default GROUP_ADDED_SUBSCRIPTION;
```

[}]: #

Our subscriptions are fired up and ready to go. We just need to add them to our UI/UX and we’re finished.

## Connecting Subscriptions to Components
Our final step is to connect our new subscriptions to our React Native components.

Let’s first apply `messageAdded` to the `Messages` component. When a user is looking at messages within a group thread, we want new messages to pop onto the thread as they’re created.

The `graphql` module in `react-apollo` exposes a `prop` function named `subscribeToMore` that can attach subscriptions to a component. Inside the `subscribeToMore` function, we pass the subscription, variables, and tell the component how to modify query data state with `updateQuery`.

Take a look at the updated code in our `Messages` component in `client/src/screens/messages.screen.js`:

[{]: <helper> (diffStep 6.13)

#### Step 6.13: Apply subscribeToMore to Messages

##### Changed client&#x2F;src&#x2F;screens&#x2F;messages.screen.js
```diff
@@ -17,11 +17,13 @@
 ┊17┊17┊import _ from 'lodash';
 ┊18┊18┊import moment from 'moment';
 ┊19┊19┊
+┊  ┊20┊import { wsClient } from '../app';
 ┊20┊21┊import Message from '../components/message.component';
 ┊21┊22┊import MessageInput from '../components/message-input.component';
 ┊22┊23┊import GROUP_QUERY from '../graphql/group.query';
 ┊23┊24┊import CREATE_MESSAGE_MUTATION from '../graphql/create-message.mutation';
 ┊24┊25┊import USER_QUERY from '../graphql/user.query';
+┊  ┊26┊import MESSAGE_ADDED_SUBSCRIPTION from '../graphql/message-added.subscription';
 ┊25┊27┊
 ┊26┊28┊const styles = StyleSheet.create({
 ┊27┊29┊  container: {
```
```diff
@@ -107,6 +109,35 @@
 ┊107┊109┊        });
 ┊108┊110┊      }
 ┊109┊111┊
+┊   ┊112┊      // we don't resubscribe on changed props
+┊   ┊113┊      // because it never happens in our app
+┊   ┊114┊      if (!this.subscription) {
+┊   ┊115┊        this.subscription = nextProps.subscribeToMore({
+┊   ┊116┊          document: MESSAGE_ADDED_SUBSCRIPTION,
+┊   ┊117┊          variables: {
+┊   ┊118┊            userId: 1, // fake the user for now
+┊   ┊119┊            groupIds: [nextProps.navigation.state.params.groupId],
+┊   ┊120┊          },
+┊   ┊121┊          updateQuery: (previousResult, { subscriptionData }) => {
+┊   ┊122┊            const newMessage = subscriptionData.data.messageAdded;
+┊   ┊123┊
+┊   ┊124┊            return update(previousResult, {
+┊   ┊125┊              group: {
+┊   ┊126┊                messages: {
+┊   ┊127┊                  edges: {
+┊   ┊128┊                    $unshift: [{
+┊   ┊129┊                      __typename: 'MessageEdge',
+┊   ┊130┊                      node: newMessage,
+┊   ┊131┊                      cursor: Buffer.from(newMessage.id.toString()).toString('base64'),
+┊   ┊132┊                    }],
+┊   ┊133┊                  },
+┊   ┊134┊                },
+┊   ┊135┊              },
+┊   ┊136┊            });
+┊   ┊137┊          },
+┊   ┊138┊        });
+┊   ┊139┊      }
+┊   ┊140┊
 ┊110┊141┊      this.setState({
 ┊111┊142┊        usernameColors,
 ┊112┊143┊      });
```
```diff
@@ -211,6 +242,7 @@
 ┊211┊242┊  }),
 ┊212┊243┊  loading: PropTypes.bool,
 ┊213┊244┊  loadMoreEntries: PropTypes.func,
+┊   ┊245┊  subscribeToMore: PropTypes.func,
 ┊214┊246┊};
 ┊215┊247┊
 ┊216┊248┊const ITEMS_PER_PAGE = 10;
```
```diff
@@ -221,9 +253,10 @@
 ┊221┊253┊      first: ITEMS_PER_PAGE,
 ┊222┊254┊    },
 ┊223┊255┊  }),
-┊224┊   ┊  props: ({ data: { fetchMore, loading, group } }) => ({
+┊   ┊256┊  props: ({ data: { fetchMore, loading, group, subscribeToMore } }) => ({
 ┊225┊257┊    loading,
 ┊226┊258┊    group,
+┊   ┊259┊    subscribeToMore,
 ┊227┊260┊    loadMoreEntries() {
 ┊228┊261┊      return fetchMore({
 ┊229┊262┊        // query: ... (you can specify a different query.
```

[}]: #

After we connect `subscribeToMore` to the component’s props, we attach a subscription property on the component (so there’s only one) which initializes `subscribeToMore` with the required parameters. Inside `updateQuery`, when we receive a new message, we make sure its not a duplicate, and then unshift the message onto our collection of messages.

Does it work?! ![Working Image](https://s3-us-west-1.amazonaws.com/tortilla/chatty/step6-13.gif)

We need to subscribe to new Groups and Messages so our Groups component will update in real time. The Groups component needs to subscribe to `groupAdded` and `messageAdded` because in addition to new groups popping up when they’re created, the latest messages should also show up in each group’s preview. 

However, instead of using `subscribeToMore` in our Groups screen, we should actually consider applying these subscriptions to a higher order component (HOC) for our application. If we navigate away from the Groups screen at any point, we will unsubscribe and won't receive real-time updates while we're away from the screen. We'd need to refetch queries from the network when returning to the Groups screen to guarantee that our data is up to date. 

If we attach our subscription to a higher order component, like `AppWithNavigationState`, we can stay subscribed to the subscriptions no matter where the user navigates and always keep our state up to date in real time! 

Let's apply the `USER_QUERY` to `AppWithNavigationState` in `client/src/navigation.js` and include two subscriptions using `subscribeToMore` for new `Messages` and `Groups`:

[{]: <helper> (diffStep 6.14)

#### Step 6.14: Apply subscribeToMore to AppWithNavigationState

##### Changed client&#x2F;src&#x2F;navigation.js
```diff
@@ -7,6 +7,10 @@
 ┊ 7┊ 7┊} from 'react-navigation-redux-helpers';
 ┊ 8┊ 8┊import { Text, View, StyleSheet } from 'react-native';
 ┊ 9┊ 9┊import { connect } from 'react-redux';
+┊  ┊10┊import { graphql, compose } from 'react-apollo';
+┊  ┊11┊import update from 'immutability-helper';
+┊  ┊12┊import { map } from 'lodash';
+┊  ┊13┊import { Buffer } from 'buffer';
 ┊10┊14┊
 ┊11┊15┊import Groups from './screens/groups.screen';
 ┊12┊16┊import Messages from './screens/messages.screen';
```
```diff
@@ -14,6 +18,10 @@
 ┊14┊18┊import GroupDetails from './screens/group-details.screen';
 ┊15┊19┊import NewGroup from './screens/new-group.screen';
 ┊16┊20┊
+┊  ┊21┊import { USER_QUERY } from './graphql/user.query';
+┊  ┊22┊import MESSAGE_ADDED_SUBSCRIPTION from './graphql/message-added.subscription';
+┊  ┊23┊import GROUP_ADDED_SUBSCRIPTION from './graphql/group-added.subscription';
+┊  ┊24┊
 ┊17┊25┊const styles = StyleSheet.create({
 ┊18┊26┊  container: {
 ┊19┊27┊    flex: 1,
```
```diff
@@ -82,6 +90,35 @@
 ┊ 82┊ 90┊const addListener = createReduxBoundAddListener("root");
 ┊ 83┊ 91┊
 ┊ 84┊ 92┊class AppWithNavigationState extends Component {
+┊   ┊ 93┊  componentWillReceiveProps(nextProps) {
+┊   ┊ 94┊    if (!nextProps.user) {
+┊   ┊ 95┊      if (this.groupSubscription) {
+┊   ┊ 96┊        this.groupSubscription();
+┊   ┊ 97┊      }
+┊   ┊ 98┊
+┊   ┊ 99┊      if (this.messagesSubscription) {
+┊   ┊100┊        this.messagesSubscription();
+┊   ┊101┊      }
+┊   ┊102┊    }
+┊   ┊103┊
+┊   ┊104┊    if (nextProps.user &&
+┊   ┊105┊      (!this.props.user || nextProps.user.groups.length !== this.props.user.groups.length)) {
+┊   ┊106┊      // unsubscribe from old
+┊   ┊107┊
+┊   ┊108┊      if (typeof this.messagesSubscription === 'function') {
+┊   ┊109┊        this.messagesSubscription();
+┊   ┊110┊      }
+┊   ┊111┊      // subscribe to new
+┊   ┊112┊      if (nextProps.user.groups.length) {
+┊   ┊113┊        this.messagesSubscription = nextProps.subscribeToMessages();
+┊   ┊114┊      }
+┊   ┊115┊    }
+┊   ┊116┊
+┊   ┊117┊    if (!this.groupSubscription && nextProps.user) {
+┊   ┊118┊      this.groupSubscription = nextProps.subscribeToGroups();
+┊   ┊119┊    }
+┊   ┊120┊  }
+┊   ┊121┊
 ┊ 85┊122┊  render() {
 ┊ 86┊123┊    return (
 ┊ 87┊124┊      <AppNavigator navigation={addNavigationHelpers({
```
```diff
@@ -93,8 +130,84 @@
 ┊ 93┊130┊  }
 ┊ 94┊131┊}
 ┊ 95┊132┊
+┊   ┊133┊AppWithNavigationState.propTypes = {
+┊   ┊134┊  dispatch: PropTypes.func.isRequired,
+┊   ┊135┊  nav: PropTypes.object.isRequired,
+┊   ┊136┊  subscribeToGroups: PropTypes.func,
+┊   ┊137┊  subscribeToMessages: PropTypes.func,
+┊   ┊138┊  user: PropTypes.shape({
+┊   ┊139┊    id: PropTypes.number.isRequired,
+┊   ┊140┊    email: PropTypes.string.isRequired,
+┊   ┊141┊    groups: PropTypes.arrayOf(
+┊   ┊142┊      PropTypes.shape({
+┊   ┊143┊        id: PropTypes.number.isRequired,
+┊   ┊144┊        name: PropTypes.string.isRequired,
+┊   ┊145┊      }),
+┊   ┊146┊    ),
+┊   ┊147┊  }),
+┊   ┊148┊};
+┊   ┊149┊
 ┊ 96┊150┊const mapStateToProps = state => ({
 ┊ 97┊151┊  nav: state.nav,
 ┊ 98┊152┊});
 ┊ 99┊153┊
-┊100┊   ┊export default connect(mapStateToProps)(AppWithNavigationState);
+┊   ┊154┊const userQuery = graphql(USER_QUERY, {
+┊   ┊155┊  options: () => ({ variables: { id: 1 } }), // fake the user for now
+┊   ┊156┊  props: ({ data: { loading, user, subscribeToMore } }) => ({
+┊   ┊157┊    loading,
+┊   ┊158┊    user,
+┊   ┊159┊    subscribeToMessages() {
+┊   ┊160┊      return subscribeToMore({
+┊   ┊161┊        document: MESSAGE_ADDED_SUBSCRIPTION,
+┊   ┊162┊        variables: {
+┊   ┊163┊          userId: 1, // fake the user for now
+┊   ┊164┊          groupIds: map(user.groups, 'id'),
+┊   ┊165┊        },
+┊   ┊166┊        updateQuery: (previousResult, { subscriptionData }) => {
+┊   ┊167┊          const previousGroups = previousResult.user.groups;
+┊   ┊168┊          const newMessage = subscriptionData.data.messageAdded;
+┊   ┊169┊
+┊   ┊170┊          const groupIndex = map(previousGroups, 'id').indexOf(newMessage.to.id);
+┊   ┊171┊
+┊   ┊172┊          return update(previousResult, {
+┊   ┊173┊            user: {
+┊   ┊174┊              groups: {
+┊   ┊175┊                [groupIndex]: {
+┊   ┊176┊                  messages: {
+┊   ┊177┊                    edges: {
+┊   ┊178┊                      $set: [{
+┊   ┊179┊                        __typename: 'MessageEdge',
+┊   ┊180┊                        node: newMessage,
+┊   ┊181┊                        cursor: Buffer.from(newMessage.id.toString()).toString('base64'),
+┊   ┊182┊                      }],
+┊   ┊183┊                    },
+┊   ┊184┊                  },
+┊   ┊185┊                },
+┊   ┊186┊              },
+┊   ┊187┊            },
+┊   ┊188┊          });
+┊   ┊189┊        },
+┊   ┊190┊      });
+┊   ┊191┊    },
+┊   ┊192┊    subscribeToGroups() {
+┊   ┊193┊      return subscribeToMore({
+┊   ┊194┊        document: GROUP_ADDED_SUBSCRIPTION,
+┊   ┊195┊        variables: { userId: user.id },
+┊   ┊196┊        updateQuery: (previousResult, { subscriptionData }) => {
+┊   ┊197┊          const newGroup = subscriptionData.data.groupAdded;
+┊   ┊198┊
+┊   ┊199┊          return update(previousResult, {
+┊   ┊200┊            user: {
+┊   ┊201┊              groups: { $push: [newGroup] },
+┊   ┊202┊            },
+┊   ┊203┊          });
+┊   ┊204┊        },
+┊   ┊205┊      });
+┊   ┊206┊    },
+┊   ┊207┊  }),
+┊   ┊208┊});
+┊   ┊209┊
+┊   ┊210┊export default compose(
+┊   ┊211┊  connect(mapStateToProps),
+┊   ┊212┊  userQuery,
+┊   ┊213┊)(AppWithNavigationState);
```

##### Changed client&#x2F;src&#x2F;screens&#x2F;groups.screen.js
```diff
@@ -191,7 +191,7 @@
 ┊191┊191┊    const { loading, user, networkStatus } = this.props;
 ┊192┊192┊
 ┊193┊193┊    // render loading placeholder while we fetch messages
-┊194┊   ┊    if (loading) {
+┊   ┊194┊    if (loading || !user) {
 ┊195┊195┊      return (
 ┊196┊196┊        <View style={[styles.loading, styles.container]}>
 ┊197┊197┊          <ActivityIndicator />
```

[}]: #

We have to do a little extra work to guarantee that our `messageSubscription` updates when we add or remove new groups. Otherwise, if a new group is created and someone sends a message, the user won’t be subscribed to receive that new message. When we need to update the subscription, we unsubscribe by calling the subscription as a function `messageSubscription()` and then reset `messageSubscription` to reflect the latest `nextProps.subscribeToMessages`.

One of the cooler things about Apollo is it caches all the queries and data that we've fetched and reuses data for the same query in the future instead of requesting it from the network (unless we specify otherwise). `USER_QUERY` will  make a request to the network and then data will be reused for subsequent executions. Our app setup tracks any data changes with subscriptions, so we only end up requesting the data we need from the server once!

## Handling broken connections
We need to do one more step to make sure our app stays updated in real-time. Sometimes users will lose internet connectivity or the WebSocket might disconnect temporarily. During these blackout periods, our client won't receive any subscription events, so our app won't receive new messages or groups. `subscriptions-transport-ws` has a built-in reconnecting mechanism, but it won't track any missed subscription events.

The simplest way to handle this issue is to refetch all relevant queries when our app reconnects. `wsClient` exposes an `onReconnected` function that will call the supplied callback function when the WebSocket reconnects. We can simply call `refetch` on our queries in the callback.

[{]: <helper> (diffStep 6.15)

#### Step 6.15: Apply onReconnected to refetch missed subscription events

##### Changed client&#x2F;src&#x2F;navigation.js
```diff
@@ -22,6 +22,8 @@
 ┊22┊22┊import MESSAGE_ADDED_SUBSCRIPTION from './graphql/message-added.subscription';
 ┊23┊23┊import GROUP_ADDED_SUBSCRIPTION from './graphql/group-added.subscription';
 ┊24┊24┊
+┊  ┊25┊import { wsClient } from './app';
+┊  ┊26┊
 ┊25┊27┊const styles = StyleSheet.create({
 ┊26┊28┊  container: {
 ┊27┊29┊    flex: 1,
```
```diff
@@ -99,6 +101,15 @@
 ┊ 99┊101┊      if (this.messagesSubscription) {
 ┊100┊102┊        this.messagesSubscription();
 ┊101┊103┊      }
+┊   ┊104┊
+┊   ┊105┊      // clear the event subscription
+┊   ┊106┊      if (this.reconnected) {
+┊   ┊107┊        this.reconnected();
+┊   ┊108┊      }
+┊   ┊109┊    } else if (!this.reconnected) {
+┊   ┊110┊      this.reconnected = wsClient.onReconnected(() => {
+┊   ┊111┊        this.props.refetch(); // check for any data lost during disconnect
+┊   ┊112┊      }, this);
 ┊102┊113┊    }
 ┊103┊114┊
 ┊104┊115┊    if (nextProps.user &&
```
```diff
@@ -133,6 +144,7 @@
 ┊133┊144┊AppWithNavigationState.propTypes = {
 ┊134┊145┊  dispatch: PropTypes.func.isRequired,
 ┊135┊146┊  nav: PropTypes.object.isRequired,
+┊   ┊147┊  refetch: PropTypes.func,
 ┊136┊148┊  subscribeToGroups: PropTypes.func,
 ┊137┊149┊  subscribeToMessages: PropTypes.func,
 ┊138┊150┊  user: PropTypes.shape({
```
```diff
@@ -153,9 +165,10 @@
 ┊153┊165┊
 ┊154┊166┊const userQuery = graphql(USER_QUERY, {
 ┊155┊167┊  options: () => ({ variables: { id: 1 } }), // fake the user for now
-┊156┊   ┊  props: ({ data: { loading, user, subscribeToMore } }) => ({
+┊   ┊168┊  props: ({ data: { loading, user, refetch, subscribeToMore } }) => ({
 ┊157┊169┊    loading,
 ┊158┊170┊    user,
+┊   ┊171┊    refetch,
 ┊159┊172┊    subscribeToMessages() {
 ┊160┊173┊      return subscribeToMore({
 ┊161┊174┊        document: MESSAGE_ADDED_SUBSCRIPTION,
```

##### Changed client&#x2F;src&#x2F;screens&#x2F;messages.screen.js
```diff
@@ -138,9 +138,18 @@
 ┊138┊138┊        });
 ┊139┊139┊      }
 ┊140┊140┊
+┊   ┊141┊      if (!this.reconnected) {
+┊   ┊142┊        this.reconnected = wsClient.onReconnected(() => {
+┊   ┊143┊          this.props.refetch(); // check for any data lost during disconnect
+┊   ┊144┊        }, this);
+┊   ┊145┊      }
+┊   ┊146┊
 ┊141┊147┊      this.setState({
 ┊142┊148┊        usernameColors,
 ┊143┊149┊      });
+┊   ┊150┊    } else if (this.reconnected) {
+┊   ┊151┊      // remove event subscription
+┊   ┊152┊      this.reconnected();
 ┊144┊153┊    }
 ┊145┊154┊  }
 ┊146┊155┊
```
```diff
@@ -242,6 +251,7 @@
 ┊242┊251┊  }),
 ┊243┊252┊  loading: PropTypes.bool,
 ┊244┊253┊  loadMoreEntries: PropTypes.func,
+┊   ┊254┊  refetch: PropTypes.func,
 ┊245┊255┊  subscribeToMore: PropTypes.func,
 ┊246┊256┊};
 ┊247┊257┊
```
```diff
@@ -253,9 +263,10 @@
 ┊253┊263┊      first: ITEMS_PER_PAGE,
 ┊254┊264┊    },
 ┊255┊265┊  }),
-┊256┊   ┊  props: ({ data: { fetchMore, loading, group, subscribeToMore } }) => ({
+┊   ┊266┊  props: ({ data: { fetchMore, loading, group, refetch, subscribeToMore } }) => ({
 ┊257┊267┊    loading,
 ┊258┊268┊    group,
+┊   ┊269┊    refetch,
 ┊259┊270┊    subscribeToMore,
 ┊260┊271┊    loadMoreEntries() {
 ┊261┊272┊      return fetchMore({
```

[}]: #

Final product: ![Final Image](https://s3-us-west-1.amazonaws.com/tortilla/chatty/step6-14.gif)


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step5.md) | [Next Step >](step7.md) |
|:--------------------------------|--------------------------------:|

[}]: #
