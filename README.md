# Firebase && React

## Initial Set Up

- Take a tour of the application.
- Set up a new project in the Firebase console.
- Take a tour of the Firebase console.
- Go to the Database section and create a new Cloud Firestore.
  - Put it into test mode.

## Installing Firebase in Your React Application

Let's make a new file called `firebase.js`.

```js
import firebase from 'firebase/app';

const config = {
  apiKey: 'AIzaSyAudsj8rc2TsUjwUx1ISskz-FPwEYuYlCw',
  authDomain: 'think-piece.firebaseapp.com',
  databaseURL: 'https://think-piece.firebaseio.com',
  projectId: 'think-piece',
  storageBucket: 'think-piece.appspot.com',
  messagingSenderId: '98218894562',
};

firebase.initializeApp(config);

export default firebase;
```

Explain the following:

- The apiKey just associates you with a Firebase project. We don't need to hide it.
  - Your project will be protected by security rules later.
  - There is a second, more important key that we'll use later that *should* be hidden.
- We're just pulling in `firebase/app` so that we don't end up pulling in more than we need in our client-side application.
- We configure Firebase and then we'll export it for use in other places in our application.

### Setting Up Cloud Firestore

This basic installation of firebase does *not* include Cloud Firestore. So, let's get that in place as well.

```js
import firebase from 'firebase/app';
import 'firebase/firestore'; // NEW

const config = {
  apiKey: 'AIzaSyAudsj8rc2TsUjwUx1ISskz-FPwEYuYlCw',
  authDomain: 'think-piece.firebaseapp.com',
  databaseURL: 'https://think-piece.firebaseio.com',
  projectId: 'think-piece',
  storageBucket: 'think-piece.appspot.com',
  messagingSenderId: '98218894562',
};

firebase.initializeApp(config);

export const firestore = firebase.firestore(); // NEW

export default firebase;
```

## Cloud Firestore

### Fetching Posts from Cloud Firestore

Let's start by fetching posts whenenver the `Application` component mounts.

First, let's pull in Cloud Firestore from our new `firebase.js` file.

```js
import { firestore } from '../firebase';
```

Now, we'll get all of  the posts from Cloud Firestore whenenver the `Application` component mounts.

```js
componentDidMount = async () => {
  const posts = await firestore.collection('posts').get();

  console.log(posts);
}
```

Hmm… that looks like a `QuerySnapshot` not our posts. What is that?

### QuerySnapshots

<!-- SLIDES -->

A `QuerySnapshot` has the following properties:

- `docs`: All of the documents in the snapshot.
- `empty`: This is a boolean that lets us know if the snapshot was empty.
- `metadata`:  Metadata about this snapshot, concerning its source and if it has local modifications.
  - Example: `SnapshotMetadata {hasPendingWrites: false, fromCache: false}`
- `query`: A reference to the query that you fired.
- `size`: The number of documents in the `QuerySnapshot`.

…and the following methods:

- `docChanges()`: An array of the changes since the last snapshot.
- `forEach()`: Iterates over the entire array of snapshots.
- `isEqual()`: Let's you know if it matches another snapshot.

`QuerySnapshots` typically hold onto a number `QueryDocumentSnapshot`s, which inherit from `DocumentSnapshot` and have the following properties:

- `id`: The `id` of the given document.
- `exists`: Is this even a thing in the database?
- `metadata`: Pretty much the same as `QuerySnapshot` above.
- `ref`: A reference to the the documents location in the database.

…and the following methods:

- `data()`: Gets all of the fields of the object.
- `get()`: Allows you to access a particular property on the object.
- `isEqual()`: Useful for comparisons.

References allow you to access the database itself. This is useful for getting the collection that document is from, deleting the document, listening for changes, setting and updating properties.

### Dealing With That Gnarly Error

You'll notice that we have a very mean error message at the top of our console. Cloud Firestore made an API change that we need to opt into. This is a new application, so that seems fine.

```js
firestore.settings({ timestampsInSnapshots: true });
```

Now the error should be gone.

### Iteraring Through Documents

So, now let's iterate through all zero of our documents.

```js
componentDidMount = async () => {
  const snapshot = await firestore.collection('posts').get();

  snapshot.forEach(doc => {
    const id = doc.id;
    const data = doc.data();

    console.log({ id, data });
  });
}
```

There won't be a lot to see here. Let's go into the Cloud Firestore console and create a document.

Now, we should see it in the console.

```js
componentDidMount = async () => {
  const snapshot = await firestore.collection('posts').get();

  const posts = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));

  this.setState({ posts });
}
```

An aside, combining the document IDs with the data is something we're going to be doing a lot. Let's make a utility method in `utilities.js`:

```js
export const collectIdsAndData = doc => ({ id: doc.id, ...doc.data() })
```

Now, we'll refactor that code as follows in `Application.js`:

```js
componentDidMount = async () => {
    const snapshot = await firestore.collection('posts').get();

    const posts = snapshot.docs.map(collectIdsAndData);

    this.setState({ posts });
  }
```

Now, we can rid of the those posts in state.

```js
state = {
  posts: [],
};
```

### Adding a New Post

First of all, we need to get rid of that `Date.now()` based `id` in `AddPost`. It was useful for us for a second or two there, but now have Firebase generating for us on our behalf.

```js
handleCreate = async post => {
  const docRef = await firestore.collection('posts').add(post);
  const doc = await docRef.get();

  const newPost = {
    id: doc.id,
    ...doc.data(),
  };

  const { posts } = this.state;
  this.setState({ posts: [newPost, ...posts] });
};
```

**Important**: Get rid of the automatically generated date-based ID!

### Removing a Post

In `Application.js`:

```js
import React, { Component } from 'react';

import Posts from './Posts';
import { firestore } from '../firebase';

class Application extends Component {
  // …

  handleRemove = async (id) => { // NEW
    const allPosts = this.state.posts;

    const posts = allPosts.filter(post => id !== post.id);

    this.setState({ posts });
  };

  render() {
    const { posts } = this.state;

    return (
      <main className="Application">
        <h1>Think Piece</h1>
        <Posts
          posts={posts}
          onCreate={this.handleCreate}
          onRemove={this.handleRemove} // NEW
        />
      </main>
    );
  }
}

export default Application;
```

In `Posts.js`:

```js
const Posts = ({ posts, onCreate, onRemove /* NEW */ }) => {
  return (
    <section className="Posts">
      <AddPost onCreate={onCreate} />
      {posts.map(post => (
        <Post {...post} onRemove={onRemove} key={post.id} /> /* NEW */
      ))}
    </section>
  );
};
```

In `Post.js`:

```js
<button className="delete" onClick={() => onRemove(id)}>Delete</button>
```

Now, we need to actually remove it from the Firestore.

```js
handleRemove = async (id) => { // NEW
  const allPosts = this.state.posts;

  const posts = allPosts.filter(post => id !== post.id);

  await firestore.doc(`posts/${id}`).delete();

  this.setState({ posts });
};
```

### Subscribing to Changes

Instead of managing data manually, you can also subscribe to changes in the database. Instead of a `.get()` on the collection. You'd go with `.onSnapshot()`.

```js
import React, { Component } from 'react';

import Posts from './Posts';
import { firestore } from '../firebase';

class Application extends Component {
  state = {
    posts: [],
  };

  unsubscribe = null; // NEW

  componentDidMount = async () => {
    this.unsubscribe = firestore.collection('posts').onSnapshot(snapshot => { // NEW
      const posts = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      this.setState({ posts });
    });
  };

  componentWillUnmount = () => { // NEW
    this.unsubscribe();
  }

  handleCreate = async post => {
    const docRef = await firestore.collection('posts').add(post);
    // const doc = await docRef.get();

    // const newPost = {
    //   id: doc.id,
    //   ...doc.data(),
    // };

    // const { posts } = this.state;
    // this.setState({ posts: [newPost, ...posts] });
  };

  handleRemove = async (id) => {
    // const allPosts = this.state.posts;

    try {
      await firestore.collection('posts').doc(id).delete();
      // const posts = allPosts.filter(post => id !== post.id);
      // this.setState({ posts });
    } catch (error) {
      console.error(error);
    }
  };

  render() {
    // …
  }
}

export default Application;
```

#### Refactoring

In `Post.jsx`:

```js
<button className="delete" onClick={() => firestore.collection('posts').doc(id).delete()}>
  Delete
</button>
```

In `AddPost.js`:

```js
handleSubmit = async event => {
  event.preventDefault();

  const { title, content } = this.state;

  const post = {
    title,
    content,
    user: {
      uid: '1111',
      displayName: 'Steve Kinney',
      email: 'steve@mailinator.com',
      photoURL: 'http://placekitten.com/g/200/200',
    },
    favorites: 0,
    comments: 0,
    createdAt: new Date(),
  }

  firestore.collection('posts').add(post); // NEW

  this.setState({ title: '', content: '' });
};
```

In `Application.jsx`:

- Remove the `handleCreate` method completely.
- Remove the `handleRemove` method completely.
- Remove `onCreate` and `onRemove` from the `<Post />` component in the `render()` method.

### Getting the Ordering Right

```js
this.unsubscribe = firestore.collection('posts').orderBy('createdAt', 'desc').onSnapshot(snapshot => { // NEW
  const posts = snapshot.docs.map(collectIdsAndData);
  this.setState({ posts });
});
```

### Using Firestore's Timestamps

Remember when we calmed Firebase down about timestamps? Take a good hard look at the date of the new posts we're creating. Uh oh! Moment.js is choking on invalid dates. This is because we're getting special `Timestamp` objects back from Firebase. We need to convert these to dates.

In `Post.jsx`:

```js
moment(createdAt.toDate()).calendar()
```

### Exercise: Updating Documents

Let's implement a naive approach to updating documents in Cloud Firestore.

We have that "Star" button. When a user clicks the "Star" button, we should increment the Stars on a post. We'll eventually write a better implementation of this.

#### Solution

```js
<button
  className="star"
  onClick={() => {
    firestore
      .collection('posts')
      .doc(id)
      .update({ stars: stars + 1 });
  }}
>
  Star
</button>
```

#### Quick Refactoring

```js
const postRef = firestore.doc(`posts/${id}`);

//…

<div>
  <button className="star" onClick={() => postRef.update({ stars: stars + 1 })} >Star</button>
  <button className="delete" onClick={() => postRef.delete()}>Delete</button>
</div>
```

## Authentication

Right now, the application is wide open. If we pushed this to production, any user could do literally anything they wanted to our database. That's not good.

Let's implement authentication in our application.

First, let's head over to the dashboard and turn on some authentication. We'll be using two forms of authentication.

- Email and password authentication
- Google sign-in

Let's go an turn those on.

### Wiring the Current User Up to Application State

Let's store the current user in the state of the `Application` component for now.

```js
state = {
  posts: [],
  user: null
};
```

Cool.We have a `CurrentUser`, `SignIn`, and `SignUp` components ready to rock.

We're going to start with Google Sign-in because I can assume you have a Google account if you can create a Firebase application.

In `Application.jsx`:

```js
render() {
  const { posts, user } = this.state;

  return (
    <main className="Application">
      <h1>Think Piece</h1>
      {user ? <CurrentUser {...user} /> : <SignIn />}
      <Posts posts={posts} />
    </main>
  );
}
```

In `firebase.js`:

```js
import 'firebase/auth';

// …

export const auth = firebase.auth();
export const provider = new firebase.auth.GoogleAuthProvider();
export const signInWithGoogle = () => auth.signInWithPopup(provider);
```

In `SignIn.jsx`:

```js
<button onClick={signInWithGoogle}>Sign In With Google</button>
```

### Updating Based on Authentication State

In `Application.jsx`:

```js
unsubscribeFromFirestore = null;
unsubscribeFromAuth = null;

componentDidMount = async () => {
  this.unsubscribeFromFirestore = firestore
    .collection('posts')
    .onSnapshot(snapshot => {
      const posts = snapshot.docs.map(collectIdsAndData);
      this.setState({ posts });
    });

  this.unsubscribeFromAuth = auth.onAuthStateChanged(user => {
    this.setState({ user });
  });
};

componentWillUnmount = () => {
  this.unsubscribeFromFirestore();
  this.unsubscribeFromAuth();
};
```

### Exercise: Implement Sign Out

I'll add this to `firebase.js`:

```js
export const signOut = () => auth.signOut();
```

This one is pretty simple. There is a method called `auth.signOut()`. Can you write it up to the "Sign Out" button?

#### Solution

In `CurrentUser.jsx`:

```js
<button onClick={signOut}>Sign Out</button>
```

### Showing the Right Component The First TIme

```js
state = {
  posts: [],
  user: null,
  userLoaded: false
};
```

```js
this.unsubscribeFromAuth = auth.onAuthStateChanged(user => {
  this.setState({ user, userLoaded: true });
});
```

```js
render() {
  const { posts, user, userLoaded } = this.state;

  const userInformation = user ? <CurrentUser {...user} /> : <SignIn />

  return (
    <main className="Application">
      <h1>Think Piece</h1>
      { userLoaded && userInformation }
      <Posts posts={posts} onCreate={this.handleCreate} onRemove={this.handleRemove} />
    </main>
  );
}
```

## Security Rules

Up until now, everything has been wide open. That's not great. If we're going to push stuff out to production, we're going to need to start adding some security to our application.

<!-- SLIDES -->

Cloud Firestore rules always following this structure:

```
service cloud.firestore {
  match /databases/{database}/documents {
    // ...
  }
}
```

There is a nice query pattern for rules:

```
service cloud.firestore {
  match /databases/{database}/documents {
    match /posts/{postId} {
      allow read: if <condition>;
      allow write: if <condition>;
    }
  }
}
```

You can combine them into one:

```
service cloud.firestore {
  match /databases/{database}/documents {
    match /posts/{postId} {
      allow read, write: if <condition>;
    }
  }
}
```

You can get a bit more granular if you'd like:

- `read`
  - `get`
  - `list`
- `write`
  - `create`
  - `update`
  - `delete`

You can nest rules to sub-collections:

```
service cloud.firestore {
  match /databases/{database}/documents {
    match /posts/{postId} {
      match /comments/{comment} {
        allow read, write: if <condition>;
      }
    }
  }
}
```

If you want to go to an arbitrary depth, then you can do `{document=**}`.

**Important**: If multiple rules match, then the operation is allowed if _any_ of them are true.

### Practical Examples

Only read or write if you're logged in.

```
service cloud.firestore {
  match /databases/{database}/documents {
    // Allow the user to access documents in the "posts" collection
    // only if they are authenticated.
    match /posts/{postId} {
      allow read, write: if request.auth.uid != null;
    }
  }
}
```

Only read and write your own data:

```
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read, update, delete: if request.auth.uid == userId;
      allow create: if request.auth.uid != null;
    }
  }
}
```

### Validating Based on the Document

- `resource.data` will have the fields on the document as it is stored in the database.
- `request.resource.data` will have the incoming document. (**Note**: This is all you have if you're responding to document creation.)

### Accessing Other Documents

- `exists(/databases/$(database)/documents/users/$(request.auth.uid))` will verify that a document exists.
- `get(/databases/$(database)/documents/users/$(request.auth.uid)).data` will get you the data of another document.

You can write JavaScript functions to make stuff easier if you want.

<!-- TODO: Make an exercise using the emulator to test rules out. -->

### Tasting Notes

- Security rules are all or nothing
- You can limit the size of a query so that malicious users (or you after a big lunch) can't run expensive queries
  - `allow list: if request.query.limit <= 10;`

### The Current Defaults

This is what we have by default:

```
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write;
    }
  }
}
```

Wide open for anyone and anything. Not cool.

Hit publish! Cool. Now things blow up.

Let's make it so that authenticated users can add posts.

### Only Allowing Posts If Logged In

```
service cloud.firestore {
  match /databases/{database}/documents {
    match /posts/{postId} {
      allow read;
      allow write: if request.auth.uid != null;
    }
  }
}
```

**Note**: Now that this can fail, let's add some better error handling.

Okay, so now any logged in user can also delete any other user's posts…

### Users Can Only Delete Their Own Posts

```
service cloud.firestore {
  match /databases/{database}/documents {
    match /posts/{postId} {
      allow read;
      allow create: if request.auth.uid != null;
      allow update, delete: if request.auth.uid == resource.data.user.uid;
    }
  }
}
```

That's better.

### Exercise: Validating Data

Can you create a rule that insists on a title?

#### Solution

```
service cloud.firestore {
  match /databases/{database}/documents {
    match /posts/{postId} {
      allow read;
      allow create: if request.auth.uid != null && !request.resource.data.title;
      allow update, delete: if request.auth.uid == resource.data.user.uid;
    }
  }
}
```

## Implementing Sign Up with Email Authentication

In `SignUp.jsx`:

```js
handleSubmit = async event => {
  event.preventDefault();

  const { email, password, displayName } = this.state;

  try {
    const { user } = await auth.createUserWithEmailAndPassword(
      email,
      password,
    );

    user.updateProfile({ displayName });
  } catch (error) {
    alert(error);
  }

  this.setState({ displayName: '', email: '', password: '' });
};
```

This has some problems:

- The display name won't update immediately.
- There is no `photoURL` because we didn't get one for free.
- We may want to store other information beyond what we get from the use profile.

The solution? Create documents for user profiles in Cloud Firestore.

## Storing User Information in Cloud Firestore

The information on the user object is great, but we're going to run into limitations *real* quick.

- What if we want to let the user set a bio or something?
- What we want to set admin permissions on the users?
- What we we want to keep track of what posts that a user has favorited?

These are very reasonable possibilities, right?

The solution is super simple: We'll make documents based off of the user's `uid` in Cloud Firestore.

Let's give ourselves some of the infrastructure for this.

```js
export const createUserDocument = async (user, additionalData) => {
  // If there is no user, let's not do this.
  if (!user) return;

  // Get a reference to the location in the Firestore where the user
  // document may or may not exist.
  const userRef = firestore.doc(`users/${user.uid}`);

  // Go and fetch a document from that location.
  const snapshot = await userRef.get();

  // If there isn't a document for that user. Let's use information
  // that we got from either Google or our sign up form.
  if (!snapshot.exists) {
    const { displayName, email, photoURL } = user;
    const createdAt = new Date();
    try {
      await userRef.set({
        displayName,
        email,
        photoURL,
        createdAt,
        ...additionalData,
      });
    } catch (error) {
      console.error('Error creating user', console.error);
    }
  }

  // Get the document and return it, since that's what we're
  // likely to want to do next.
  return getUserDocument(user.uid);
};

export const getUserDocument = async uid => {
  if (!uid) return null;
  try {
    const userDocument = await firestore
      .collection('users')
      .doc(uid)
      .get();

    return { uid, ...userDocument.data() };
  } catch (error) {
    console.error('Error fetching user', error.message);
  }
};
```

We're going to put this two places:

- `onAuthStateChanged` in order to get our Google Sign Ups
- In `handleSubmit` in `SignUp` because that's where we'll have that custom display name.

### Updating Security Rules
