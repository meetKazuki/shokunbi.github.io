---
title: "On Improving Data Consistency — _A Retrospective_"
date: 2022-09-03 16:01:01
---

## Short Intro

You might be working (or have worked) on a project that requires you to track count of a certain metric in your system. You might be wondering how best to go about it.
Well, take a seat, because today we are going to dive in on how not to go about it. To go on this journey, we will be discussing about a fictional project (which I may or may not have worked on).


## Initial Business Requirement

To set the groundwork, a peep into some functionalities of this hypothetical project:
* A user can create a post.
* A User can like a post.
* A User can make comments to a post.
* A User can like comments on a post.

Some of the stats our users were interested in were:
- The number of posts a user has created.
- The number of posts a user has liked.
- The number of comments a post has.

## Assumption Number One (keeping count in the database)

Our earlier schema looked like this:

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

video_collection: {
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
```
The first mistake we did was to increment this count when there was a document insertion or a decrement when a document was removed. So, if a user created a post, we `$inc`remented the `number_of_posts` column in the `user` collection by `1`, and if a user deleted a post, we `$inc`remented the post by `-1`. The code looked like this:

```ts
// post.ts
await post_repository.create({ ...postData })
await user_repository.updateOne({ _id: user_id }, { $inc: { 'stats.number_of_posts': 1 } })
```

If a user liked a post, we did this:

```ts
// like.ts
await like_reposistory.create({ user_id, post_id });
await video_repository.updateOne({ video_id }, { $inc: { 'stats.number_of_likes': 1 } })
await user_repository.updateOne({ _id: user_id }, { $inc: { 'stats.number_of_posts_liked': 1 } })
```
In the event where a user commented on a post, we did this:

```ts
// post.ts
await video_repository.updateOne({ parent_post_id }, { $inc: { 'stats.number_of_comments': 1 } })  // increment number of comments count for the parent post.
```

THIS WAS A BAD IDEA! I can't scream loud enough how bad this was! It wasn't long before everything fell out of place. A post will have 10 comment records in the database, but the `number_of_comments` count returned was 6. What solution did we propose? We doubled-down on the blunder.

## Assumption Number Two (using database transactions)

If you look closely (not even closely), you can see where failures can occur and cause inconsistent counts. So we decided to wrap these operations in database transactions. With this we thought the inconsistency issue will be resolved. We had a helper method for handling DB transactions:

```ts
// transaction.ts
type TransactionCallback = (session: ClientSession) => Promise<void>

export const Transaction = async (callback: TransactionCallback) => {
  const session: ClientSession = await startSession()

  session.startTransaction()

  try {
    await callback(session)
    await session.commitTransaction()
  } catch (error: any) {
    await session.abortTransaction()
    logger.error(error)
    throw new InternalServerError(error)
  } finally {
    session.endSession()
  }
}
```
So if there was a `like` operation, our new structure looked like:

```ts
/* import relevant modules */

//...
await Transaction(async, (session) => {
  await like_reposistory.create({ user_id, post_id }, { session })
  await video_repository.updateOne({ video_id }, { $inc: { 'stats.number_of_likes': 1 } }, { session })
  await user_repository.updateOne({ _id: user_id }, { $inc: { 'stats.number_of_posts_liked': 1 } }, { session })
});
```

We still found ourselves in the same spot. Adding transactions did nothing (at least significantly) to improve the consistency of this count. Now you might be thinking, _they have learnt their lesson. Surely, something better this time_ Nope! We went deeper.

## Assumption Number Three (using database triggers)

We tried as a last resort. We used database triggers on update to keep the counts. I wish this is the part where I say we finally achieved the consisteny we were looking, but far from it. We ended up implementing a fix that was difficult to reason, which in turn made maintainace a pain.
So, we reverted back to what we should have done earlier...

## What We Should Have Done (Calculate the counts on the fly)

<!-- <!--Insert Thanos it brought me back to me meme/> -->

Counting when a user requested it saved a history of debugging and hair-pulling. As we realized painfully, the trouble with redundancy in databases is that when the numbers disagree, you are unsure of which is authoritative. We broke the golden rule of optimizing too early. In our case, there was no performance problem(s) that restricted us from just counting on the fly.

There is a saying "_A man with one watch always knows the time. A man with two watches is never sure_". I would advise to only store count if:
* Performance issues stop you from getting the derived numbers when you need them. Or,
* You have reason to believe that you are losing records from the main table through programmer error or deliberate or accidental user action.

If consistency is what you are aiming, count those records on the fly...
