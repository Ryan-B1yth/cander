# Cander

## Synopsis

- Social (?) platform where users can submit, rate, and refine questions and opinions and put them infront of other users to be voted on.
- All submissions and answers to them are completely anonymous and can be about anything.
- Users can see top submissions and answers of the week / month / year etc.

- The original idea stemmed from the hope of having a platform where users could vote on political issues in real time. The app would notify a random set of users with a political discussion currently in progress, or a vote that was taking place, and then display the users collective response in an easy to find location. Over time, the idea has evolved to span more than just political issues and moreso anything that the users wanted a collective vote or discussion on.

## Tech stack

| Frontend     | API     | Backend  |
| ------------ | ------- | -------- |
| React Native | Axios   | Firebase |
| Redux        | Express |          |

# Active Database

## User data

- Since all submissions and answers are anonymous the user data required is minimal:

```
{
  id: unique_id
  replies: [],
  questions: [],
  statements: [],
  skipped: [],
}

{
  id: unique_id,
  firstName: string,
  lastName: string,
  birthDate: timestamp,
  email?: string,
  phoneNumber?: string
  interactions: interactions_id
}
```

- The only data relating to the submissions a user has made or interacted with is its' ID, which is stored in the interactions field of the user object. This way when presenting a user with submissions to interact with, we can check on these arrays to make sure we don't show the user a submission they've already interacted with. This also means that there's no way of knowing which submissions the user made or just interacted with, protecting their anonymity.

- Since users are casting votes, a majority of users must interact with a submission before it can be approved and move onto the next stage.

## Submissions

- Submissions can be of a few types; Question, Reply, Statement, or Tag.

### Tag

```
{
  id: unique_id,
  body: string,
  weight: number
}
```

### Reply

```
Reply
{
  id: unique_id,
  questionId: question_id
  body: string,
  weight: number
}
```

### Question

```
{
  id: unique_id,
  tags: {
    tagId: times_used
  },
  body: string,
  replies: reply_id[]
  weight: number,
  skips: number,
  activeState: Tagging || Collection || Locked
}
```

### Statement

```
{
  id: unique_id,
  tags: {
    tagId: times_used
  },
  body: string,
  weight: number,
  skips: number,
  activeState: Tagging || Collection || Locked
}
```

## Submission life cycle

### Submission

- A user creates a question or a statement and adds up to 5 tags.
  - The submission ID is added to the users interactions.questions or interactions.statements array respectively.

### Tagging

- The submission is presented to other users who can add their own tags.
  - Every user who submits tags has the submission ID added to their interactions array.
- Each time a tag is used for a submission, its' times_used increases in the submission object.
- Responder's submissions can be the same or different to the poster's.
- After a majority percentage of users have submitted responses to the submission, the 5 highest rated tags are locked into the post - the rest are deleted.

### Collect responses

- The submission is put before users again. There are two paths from here:
- Statements:
  - Users are presented with a scale from strongly agree to strongly disagree.
  - Submission weights are affected as follows: strongly agree by +2, agree by +1, neutral by 0, disagree by -1, strongly disagree by -2.
    - The submission ID is added to the user's interactions.statements array.
- Questions:
  - Users are presented with a question and prompted to respond with a comment.
    - This increases the submissions weight by 1.
    - Responding to the question adds the submission ID to the user's interactions.question and interactions.replies arrays.
  - After a comment has been made, the user is presented a different question and 5 other users' responses to that question, which they can order based on how much they agree with the statements. Agreement logic follows the same structure as above.
    - This increases this second question's weight by 1.
    - The 5 responses are added to the user's interactions.replies array.

### Reveal consensus

- After a majority percentage of users have interacted with a submission, its' data is locked and it can no longer be interacted with.
  - Statements:
    - Statements are ranked based on their weights and displayed to all users on the communal screen.
  - Questions:
    - Questions are displayed with their top rated replies.

## Skipping

- A potential idea is to allow users to skip an unlimited amount or a number of submissions per day. These submission IDs would be added to the user's interactions.skipped array. From here, either:

  - By random chance the submission could present itself again and again until the user responds to it. At which point it is remove from the interactions.skipped array and added to its' respective interactions array.
  - Or, the submission ID is stored in the array for certain amount of time and counted towards the submissions that aren't shown to the user. After so long the whole array is emptied and filled up again by the user.
  - Expanding on this, a submission that has been skipped 3 times could be added to a permanently skipped array and never shown to the user.
  - If however, the ID is in a users skipped array when the data is locked in, they become a part of the calculated percentage who could have but did not respond to the submission, which would be displayed on the communal screen.

# Static Database

## Saving data

- To save database space and by extension, cost, data could be collated and transported to a separate database at the point of locking in the responses.

### Questions

```
{
  id: unique_id,
  tags: tag[]
  body: string,
  topReplies: reply[],
  numberOfInteractions: number,
  numberOfResponses: number,
  timestamp: timestamp
}

```

### Statements

```
{
  id: unique_id,
  tags: tag[],
  body: string,
  weight: number,
  numberOfInteractions: number,
  numberOfResponses: number
  timestamp: timestamp
}

```

- All IDs associated with this object would be removed from all active database objects except users. This way we allow users to look through the static database for just the submissions they interacted with.

# Use of the app

- It is undecided how much to allow users to interact with the app. Options include:
  - Unlimited use
    - Make as many submissions as desired
    - Interact with as many submissions as desired
  - Light limitations
    - Make only one of each type of submission per day
    - Interact with as many submissions as desired
  - Medium limitations
    - Make only one of each type of submission per day
    - Interact with a limited number of submissions per day
  - Heavy limitations
    - Users are notified at random on days when they are allowed to create or interact with submissions
    - A majority but not 100% of users on any given day

# Logic

## Initializing a new user after auth

```
import { getAuth } from "firebase/auth";
import { collection, doc, setDoc } from "firebase/firestore";

const auth = getAuth();
const user = auth.currentUser;

const currentUser = doc(collection(db, 'users', user.uid))

const newInteractionsDoc = doc(collection(db, 'interactions'))

const currentUserInteractions = setDoc(newInteractionsDoc, {
  replies: [],
  questions: [],
  statements: [],
  skipped: [],
})

await setDoc(currentUser, {
  interactions: currentUserInteractions
})

```

## Getting the users feed

```
const auth = getAuth();
const user = auth.currentUser;

const userInteractionsId = doc(db, 'users', user.uid).data().interactions
const interactedQuestions = doc(db, 'interactions', userInteractionsId).data().questions
const interactedStatements = doc(db, 'interactions', userInteractionsId).data().statements

const getFeedQuestions = () => {
  const feedQuestions = query(
    collection(db, 'questions'),
    where('activeState', '!=', Locked),
    limit(10)
  )

  for ( let i = 0; i <= interactedQuestions.length; i ++) {
    return feedQuestions.map( (j) => j.id !== interactedQuestions[i] )
  }
}

const getFeedStatements = () => {
  const feedStatements = query(
    collection(db, 'statements'),
    where('activeState', '!=', Locked),
    limit(10)
  )

  for ( let i = 0; i <= interactedStatements.length; i ++) {
    const newStatements = feedStatements.map( (j) => j.id !== interactedStatements[i] )

  }
}

const feed = [ ...getFeedQuesitons(), ...getFeedStataements() ]
```

- Option 1: shuffle the users feed

```
const shuffle = (array) => {
  let m = array.length, t, i;

  while (m) {
    i = Math.floor(Math.random() * m--);

    t = array[m];
    array[m] = array[i];
    array[i] = t;
  }

  return array;
}

const shuffledFeed = shuffle(feed)
```

- Option 2: order the feed by weight

```
const sortedFeed = feed.sort( (a, b) => {
  return a - b
}).reverse()


```

## Skipping submissions

- If the user swipes away a submission, or presses skip:

```
const { submissionId } = req.body

const submission = doc(db, 'collection', submission.id)

await updateDoc(submission, {
  skips: increment(1)
})

const auth = getAuth();
const user = auth.currentUser;

const interactionId = doc(db, 'users', user.uid).date().interaction
const userInteractions = doc(db, 'interactions', interactionId)

await updateDoc(userInteractions, {
  skips: arrayUnion(submissionId)
})

```

## Interacting with statements

- The state of agreement is captured as an integer (2, 1, 0, -1, -2) and after the user submits their responce:

```
import { doc, getDoc, updateDoc, increment } from "firebase/firestore";
import { getAuth } from "firebase/auth";


const { statementId } = req.body
const statementDoc = doc(db, 'submissions', statementId)

awaut updateDoc(statement, {
  weight: increment(integer)
})

const auth = getAuth();
const user = auth.currentUser;

const userInteractionsId = getDoc(doc(db, 'users', user.uid)).data().interactions

await updateDoc(userInteractionsId, {
  statements: arrayUnion(statementId)
})

```

## Interaction with questions

- The reply IDs of agreement is captured in 5 separate states:

```
const [first, setFirst] = useState('')
const [second, setSecond] = useState('')
const [third, setThird] = useState('')
const [forth, setForth] = useState('')
const [fifth, setFifth] = useState('')

const order = {
  first: 2,
  second: 1,
  third: 0,
  forth: -1,
  fifth: -2
}

```

- This object is passed to the API:

```
const { order, questionId } = req.body

const auth = getAuth();
const user = auth.currentUser;

const userInteractionsId = getDoc(doc(db, 'users', user.uid)).data().interactions

for (let i = 0; i <= Object.keys(order); i++) {
  const reply = doc(db, 'replies', Object.keys(order)[i])
  await updateDoc(reply, {
    weight: Object.values(order)[i]
  })
  await updateDoc(userInteractionsId, {
    replies: arrayUnion(Object.keys(order)[i])
  })
}

const questionDoc = doc(db, 'questions', questionId)

await updateDoc(questionDoc, {
  weight: increment(1)
})


```

## Adding to static database

### Questions

- When a question's activeState changes to Locked, its data gets added to the static database:

```
const { questionId } = req.body

const question = doc(db, 'questions', questionId).data()
const { id, tags, body, tags, skips } = question

const coll = query(
  collection(db, 'replies'),
  where('questionId', '==', questionId),
)

const numberOfReplies = await getCountFromServer(coll).data().count


const replies = query(
  collection(db, 'replies'),
  where('questionId', '==', questionId),
  orderBy('weight', 'desc')
  limit(numberOfReplies / 2)
  )

const staticQuestions = doc(db, 'questions')

await updateDoc(staticQuestions, {
  id,
  tags: arrayUnion(tags),
  body,
  stopReplies: replies,
  numberOfInteractions: weight + skips,
  numberOfResponses: weight,
})

```

### Questions

```
{
  id: unique_id,
  tags: tag[]
  body: string,
  topReplies: reply[],
  numberOfInteractions: number,
  numberOfResponses: number,
  timestamp: timestamp
}

```

### Statements

```
{
  id: unique_id,
  tags: tag[],
  body: string,
  weight: number,
  numberOfInteractions: number,
  numberOfResponses: number
  timestamp: timestamp
}

```

### Statements

## Adding new document

```
import { getAuth } from "firebase/auth";

const auth = getAuth();
const user = auth.currentUser;

const docRef = await addDoc(collection(db, 'collection'), {
  data
})

const currentUserInteractionsId = doc(db, 'interactions', user.uid).data().interactions
const interactionsDoc = doc(db, 'interactions', currentUserInteractionsId)

await updateDoc(interactionsDoc, {
  interactionType: arrayUnion(docRef.id)
})

```

## Social pages

### Questions

- The social feeds come from the static database.

```


```
