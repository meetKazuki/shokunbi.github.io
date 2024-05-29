---
title: "On Improving Data Consistency — _A Retrospective_"
date: 2022-09-03 16:01:01
---

A general goal for database systems is to ensure data consistency. Consistency here means the data adheres to constraints (formal and informal). And if one piece of information is
stored multiple times, then they all agree. But how can we ensure that? What are some things we should not do in order to achieve this consistency?

Let’s pick a scenario (it’s a real life example). You are tasked with building the next social media app. Your users will create posts and share them with the public. This content
will receive interactions from other users. Your users are interested in how well the content is performing. How do you ensure this information is as consistent and correct as possible?

To get through this, let’s retrospect on some of the things I did wrong...

In one of my previous lives, I was tasked with developing a social media application. Not to get into deep implementation, but this app was supposed to allow our users to carry out the following operations:
* A user can create a post.
* A user can like a post.
* A user can make comments on a post.

We tracked certain stats in this application: the number of posts a user has made, the number of likes a post has, and the number of comments a post has, among other things. This information was represented in our database this way:
- We had a collection that kept information about a user in our system (we will call it “user_collection”).
- We had another collection (we will call it “post_collection”) that kept information about posts created in our system.
- We also had a collection (we will call it “like_collection”) that kept information about the posts liked and the users who liked them.

The representation looked this way:

```json
user_collection: {
  _id: string,
  fullname: string,
  [...]
  stats: {
    number_of_posts: {
      type: number,
      default: 0
    },
    number_of_posts_liked: {
      type: number,
      default: 0
    }
  }
}

post_collection: {
  _id: string,
  owner_id: {
    type: string,
    refs: <user_collection>
  }
  [...]
  stats: {
    number_of_comments: {
      type: number,
      default: 0
    },
    number_of_likes: {
      type: number,
      default: 0
    }
  }
}

like_collection: {
  _id: string,
  user_id: {
    type: string,
    refs: <user_collection>
  },
  post_id: {
    type: string,
    refs: <post_collection>
  },
}
```
So what happens when a user carries out any of these operations? If let’s say a user creates a post, we did this in our code:

```ts
// post.ts
await post_repository.create({ ...post_data });
await user_repository.updateOne({ _id: user_id }, { $inc: { 'stats.number_of_posts': 1 } });
```

And if a user likes a post, our code looks like this:

```ts
await like_reposistory.create({ user_id, post_id });
await post_repository.updateOne({ video_id }, { $inc: { 'stats.number_of_likes': 1 } });
await user_repository.updateOne({ _id: user_id }, { $inc: { 'stats.number_of_posts_liked': 1 } });
```
What we did was: whenever a user created or liked a post, we created a record in the database and stored a counter. So if you create a post, we increment the “number_of_posts” count in the “user_collection” by 1, and so on.

#### Enter Inconsistent Data

All was going well until we realized that the stats reported ("number_of_comments," “number_of_likes”) did not match  the records that were in the database. What went wrong? Let’s take a look at our code again:

```ts
// like.ts
await like_reposistory.create({ user_id, post_id });
await post_repository.updateOne({ video_id }, { $inc: { 'stats.number_of_likes': 1 } });
await user_repository.updateOne({ _id: user_id }, { $inc: { 'stats.number_of_posts_liked': 1 } });
```
There is a big chance that when a user carries out a “like” operation, one of these might fail. What if multiple users are trying to like a post at the same time? Network failure?
There is no guarantee that this block of code will execute successfully. To improve our result (or so we thought), we moved the count operation code to an update trigger.

#### Triggers Happy? Still More Inconsistent Data

Database triggers (in MongoDB) allow you to execute server-side logic whenever a document is added, updated, or removed in a linked MongoDB cluster.
It uses change streams to listen for changes to documents in a collection and pass database events to their associated trigger functions. We moved our code around and it looked like this
with this implementation:

```ts
import { ChangeStream, ChangeStreamDocument } from 'mongodb';
import LikeModel from '@schemas/like';
import LockModel from '@schemas/lock';

LikeModel.watch([], { fullDocument: 'updateLookup' }).on('change', async (data) => {
  try {
    const uniqueId = `${data.operationType}-${(data._id as any)._data}-like`;
    await LockModel.create({ uniqueId });

    if (data.operationType === 'insert') {
      /** a whole lot of other boilerplate code */
      await like_reposistory.create({ user_id, post_id });
      await post_repository.updateOne({ video_id }, { $inc: { 'stats.number_of_likes': 1 } });
      await user_repository.updateOne({ _id: user_id }, { $inc: { 'stats.number_of_posts_liked': 1 } });
    }
    // some other logic too to check for when a document is deleted
  } catch (error) {
    // ...
  }
}) as ChangeStream
```
This solution failed as much as the reason for failure of our previous solution. We still ended up getting inconsistent counts with respect to the number of records in the database.

#### The Reliable Source?

Having gone through all these, we still were not able to keep the count up-to-date. And a lot of times, we had to go into the database, run a query, and update the counter to match the records in the database.
It wasn’t ideal and it took a lot of time to debug issues that we shouldn’t have dealt with to begin with. So, we did something very simple — `countDocument`.

> “A man with one watch always knows the time. A man with two watches is never sure.”

Counting the records turned out to be the reliable way to achieve this consistency. At the time, it was okay for us, we didn’t have a gazillion number of records and upfront,
there was no performance issue counting these records on the fly. Getting information about those metrics was as simple as:

```ts
// for getting number of likes for a post
await like_repository.countDocument({ post_id });
```
#### What Did We (I) Learn?

* It is redundant to store the count of database records, but whether or not you should do it depends on your situation (in our case, it wasn’t right).
* Unless you have known performance problems, calculate the counts and totals on the fly in your application and don’t store them.
* Database normalization rules say that you shouldn’t have any values in your database that you can programmatically construct based on other values in your database.
* Of course, it’s a different situation if you have a gazillion number of records. In such a case, you will typically want to revisit your requirements.

#### Resources

* MongoDB Database Triggers
* https://www.mongodb.com/docs/manual/reference/method/db.collection.countDocuments/
