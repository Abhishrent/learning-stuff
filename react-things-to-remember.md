# React Things to Remember

## Table of Contents

- [Event Handlers](#event-handlers)
- [State](#state)
  - [State Updates Are Asynchronous](#state-updates-are-asynchronous)
  - [Where to Define State Variables](#where-to-define-state-variables)
  - [State Lives Where It Changes](#state-lives-where-it-changes)
  - [One Source of Truth](#one-source-of-truth)
- [Immutability](#immutability)
  - [Never Mutate State](#never-mutate-state)
  - [Why Not to Mutate State — Step by Step](#why-not-to-mutate-state--step-by-step)
  - [Shallow Copy](#shallow-copy)
- [React Mental Model](#react-mental-model)
  - [UI = Function of State](#ui--function-of-state)
  - [Data Flows Downward](#data-flows-downward)
  - [Pure Display Components Are Ideal](#pure-display-components-are-ideal)
  - [The Universal React Mental Model](#the-universal-react-mental-model)
- [Controlled Components](#controlled-components)
- [Event Object](#event-object)
- [One State Object for All Form Fields](#one-state-object-for-all-form-fields)
- [Reference Code — Notes App](#reference-code--notes-app)
- [JavaScript in the Browser — Async Model](#javascript-in-the-browser--async-model)
- [Dependencies](#dependencies)
- [Promises](#promises)
  - [Things to Remember About Promises](#things-to-remember-about-promises)
  - [What a Promise Represents](#what-a-promise-represents)
  - [The Three Promise States](#the-three-promise-states)
  - [What "Resolved with a Value" Means](#what-resolved-with-a-value-means)
  - [Example Without Axios](#example-without-axios)
  - [Applying This to Axios](#applying-this-to-axios)
  - [What `.then()` Receives](#what-then-receives)
  - [The Full Flow](#the-full-flow)
  - [Key Takeaway](#key-takeaway)
  - [How the Automatic Passing Works](#how-the-automatic-passing-works)
- [Fetching Data with Axios and useEffect](#fetching-data-with-axios-and-useeffect)
- [REST Echo Response](#rest-echo-response)

---

## Event Handlers

An event handler must be a function definition or a reference to one and never a function call, with the exception that it can be a function call if the return value of that function is a function as well (a function returning a function).

Meaning, **event handlers must always evaluate to a function.**

---

## State

### State Updates Are Asynchronous

React state updates are asynchronous and batched, so calling a state setter schedules an update instead of changing the state immediately. As a workaround, we can compute the next state in a local variable and pass it to the setter, while React applies the actual state update asynchronously.

A local variable doesn't directly change what is rendered; instead, it can be used to update state through a state setter function. When the state is updated, React re-renders the component, and the new state value is used during rendering, causing the UI to update accordingly.

```js
import { useState } from 'react'

function App() {
  const [count, setCount] = useState(0)

  const increase = () => {
    const newValue = count + 1   // local variable
    setCount(newValue)           // local variable newValue passed to setter function setCount, this schedules count to update in the next render
  }

  return (
    <div>
      <p> The current count is: {count}</p> 
      <button onClick={increase}>Increase</button>
    </div>
  )
}

export default App
```

### Where to Define State Variables

State lives as low as possible, as high as necessary — if only one component needs it, keep it there; if multiple components need it, move it up to the closest parent that contains all of them.

### State Lives Where It Changes

Put state in the closest common parent that needs to control it.

A counter displayed in two components — parent owns state:

```js
const App = () => {
  const [count, setCount] = useState(0)
  return (
    <>
      <Display value={count} />
      <Button onClick={() => setCount(count + 1)} />
    </>
  )
}
```

Child components don't manage shared data, they receive it.

### One Source of Truth

Never duplicate the same data in multiple states.

Do not store something that can be calculated.

Storing derived data:

```js
// Wrong
const [firstName, setFirstName] = useState("Abhi")
const [lastName, setLastName] = useState("Khatri")
const [fullName, setFullName] = useState("Abhi Khatri")
```

Derive when rendering:

```js
// Correct
const fullName = firstName + " " + lastName
```

Why? Less state = fewer bugs + no synchronization problems.

Never duplicate the same data in multiple states:

```js
// Wrong
const [items, setItems] = useState([1,2,3])
const [itemCount, setItemCount] = useState(3)
```

```js
// Correct
const itemCount = items.length
```

If you store both, they will eventually disagree.

---

## Immutability

### Never Mutate State

In React, you should store **independent pieces of state** separately, and when that state is an array or object, you should **always copy it, modify the copy, and then call the setter** on the modified copy to update state, rather than mutating it directly. Only keep in state what **cannot be computed from other state**; everything else should be derived during render. Components read from state, and event handlers update state in this controlled way, keeping the UI predictable and avoiding inconsistencies.

React decides when to re-render by checking if state has **changed**. If you mutate the original array/object, the reference in memory stays the same, so React doesn't detect a change and **won't re-render**. When you use `concat`, `map`, or spread `{...obj}`, you create a new reference — React sees the change and updates the UI.

Always copy, modify, set:

```js
// Wrong
numbers[0] += 1
setNumbers(numbers)
```

```js
// Correct
const copy = [...numbers]
copy[0] += 1
setNumbers(copy)
```

Why? React detects changes by reference, not by peeking inside arrays.

Never do this:

```js
note.important = false
notes.push(newNote)
notes[0] = something
```

These mutate state directly. Always create **new objects or arrays**:

```js
{ ...object }
[...array]
array.map()
array.filter()
```

### Why Not to Mutate State — Step by Step

Consider this code:

```js
const note = notes.find(n => n.id === id)
note.important = !note.important

axios.put(url, note).then(response => {
  // ...
})
```

**Step 1 — `notes.find()`**

`notes` is an **array stored in React state**. `.find()` returns the **first element matching the condition**.

```js
const note = notes.find(n => n.id === id)
```

Example state:

```js
notes = [
  { id: 1, content: "Learn React", important: true },
  { id: 2, content: "Learn Node", important: false }
]
```

If `id = 1`, then:

```js
note = { id: 1, content: "Learn React", important: true }
```

But **very important detail**: `note` is NOT a new object. It is a reference to the same object inside the **`notes` array**.

Memory view:

```
notes[0]  --------┐
                  │
                  ▼
note ---------> { id:1, content:"Learn React", important:true }
```

Both point to **the same object**.

**Step 2 — What this line does**

```js
note.important = !note.important
```

If `important` was `true`, it becomes `false`. But because `note` references the **same object inside state**, this also changes `notes[0].important`. So the **state is mutated directly**. But React **did not see a state update**.

**Step 3 — Why React says this is bad**

React requires **immutable updates**. You should **never modify state objects directly**. Instead, you create **a new object**. Why? Because React detects updates using **reference comparison**.

React checks:

```js
oldNotes === newNotes ?
```

If they are the **same reference**, React may think **nothing changed**. Direct mutation breaks React's update model.

**Step 4 — The correct approach (copy the object)**

Instead of mutating the object, create a **new object**:

```js
const note = notes.find(n => n.id === id)

const changedNote = {
  ...note,
  important: !note.important
}
```

`...note` copies all properties:

```
note:        { id:1, content:"Learn React", important:true }
changedNote: { id:1, content:"Learn React", important:false }
```

Now we have **two different objects**. The original state **is untouched**.

**Step 5 — Updating the state correctly**

Now React can safely update the array:

```js
setNotes(notes.map(n =>
  n.id !== id ? n : changedNote
))
```

What this does: creates a **new array**, replaces the modified note, keeps other notes unchanged. But the **array reference is new**, so React re-renders correctly.

**One sentence summary:**

`note` is a reference to the object inside the React state array, so modifying it directly mutates the state, which React forbids because state must be treated as immutable.



### Shallow Copy

This is referring to **when we use the spread operator**. `{ ...note }` only copies one level deep. So if `note` had a nested object as a property, like:

```js
const note = {
  id: 1,
  important: true,
  metadata: { author: "Alice", tags: ["js", "react"] }
}

const changedNote = { ...note }
```

Then `changedNote.metadata` and `note.metadata` would point to the **same object in memory**, not a new copy. Mutating `changedNote.metadata.author` would also mutate `note.metadata.author`.

So the shallow copy is safe when all the properties (`id`, `important`, `content`) are primitive values (numbers, booleans, strings), which are always copied by value.

---

## React Mental Model

### UI = Function of State

Change state, UI updates automatically:

```js
const [isLoggedIn, setIsLoggedIn] = useState(false)

return (
  <>
    {isLoggedIn ? <Dashboard /> : <Login />}
  </>
)
```

No manual DOM updates. Just change state.

### Data Flows Downward

Parent controls, child displays:

```js
const Temperature = ({ value }) => <p>{value}°C</p>

const App = () => {
  const [temp, setTemp] = useState(25)
  return <Temperature value={temp} />
}
```

Children don't fetch or own shared data — they receive it.

### Pure Display Components Are Ideal

Components that only use props are easier to reason about:

```js
const Price = ({ amount }) => <h1>Rs. {amount}</h1>
```

Given same input, same output. Predictable = scalable.

### The Universal React Mental Model

If you compress everything into one guiding rule:

**Keep minimal state, update immutably, pass data down, derive the rest.**

That's the backbone of clean React apps — from tiny exercises to production systems.

---

## Controlled Components

Controlled components are form elements whose values are fully controlled by the React state. We make form elements 'controlled' in order to literally have more control over the data that these components house.

Let's understand this practice from a simple example of making an input element a 'controlled component'.

```js
const App = () => {
  return (
    <form>
      <input />
    </form>
  )
}
```

This renders an empty input field that we can input any data into. However, the control we have of the data we write inside this field is in the browser's hand rather than ours. So by changing this into a controlled component, we take things into our own (React's) hand so that the data can easily be molded and manipulated to our needs.

```js
import { useState } from 'react'

const App = () => {

  const [newName, setNewName] = useState('')

  const handleInputChange = (event) => {
    console.log(event.target.value)
    setNewName(event.target.value)
  }
  
  return (
    <div>
        <h1>Petu's Phonebook</h1>
        <form>
          <div>
          Name: <input value={newName} onChange={handleInputChange} />
          </div>
          <div> 
            <button type='submit'>Add</button>
          </div>
        </form>
    </div>
  )
}
export default App
```

---

## Event Object

When a browser event fires — a form submission, a button click, a keystroke — the browser automatically creates an event object and passes it to your handler function. That object contains all the information about what just happened.

You named it `e` but it's just a parameter name. You could call it `event` or `ev` — doesn't matter. The browser always passes it as the first argument.

Some useful things the event object carries depending on the event type:

- `e.preventDefault()` — stops the browser's default behavior (like reloading on form submit)
- `e.target` — the DOM element that triggered the event (the input field, the button, etc.)
- `e.target.value` — the current value of that element (what's typed in the input)

So when you write:

```js
const handleChange = (e) => {
  setInputState(e.target.value)
}
```

You're saying: "whatever DOM element fired this change event, give me its current value and put it in state." The browser does the passing — you just receive it.

---

## One State Object for All Form Fields

Instead of a separate `useState` for every input:

```js
const [name, setName] = useState('')
const [phone, setPhone] = useState('')
```

Use one state object:

```js
const [formState, setFormState] = useState({ name: '', phone: '' })
```

One handler for all fields using `e.target.name` and a computed property key:

```js
const handleChange = (e) => {
  setFormState({
    ...formState,
    [e.target.name]: e.target.value
  })
}
```

Each input needs a `name` attribute matching the key in state:

```jsx
<input name="name" value={formState.name} onChange={handleChange} />
```

`e.target` always refers to the specific element that fired the event — so if the name input fires, `e.target` is that input; if phone fires, `e.target` is that one instead. `e.target.name` dynamically picks which key to update and `e.target.value` gives its current value. One handler, any number of inputs.

After the user types "Abhishrent" and "9841000000":

```js
{ name: 'Abhishrent', phone: '9841000000' }
```

---

## Reference Code — Notes App

```js
import { useState } from 'react'
import Note from './components/Note'

const App = (props) => {
  const [notes, setNotes] = useState(props.notes)
  const [newNote, setNewNote] = useState('a new note')
  const [showAll, setShowAll] = useState(true)

  const notesToShow = showAll ? notes : notes.filter(note => note.important === true)

  const handleNoteChange = (event) => {
    console.log(event.target.value)
    setNewNote(event.target.value)
  }

  const addNote = (event) => {
    event.preventDefault()
    console.log('button clicked', event.target)
    const noteObject = {
      content: newNote, 
      important: Math.random() < 0.5,
      id: String(notes.length + 1)
    }

    setNotes(notes.concat(noteObject))
    setNewNote('')
  }

  return (
    <div>
      <h1>Notes</h1>
      <div>
        <button onClick={()=> setShowAll(!showAll)}>
          show {showAll ? 'important' : 'all'}
        </button>
      </div>
      <ul>
        {notesToShow.map(note => 
          <Note key={note.id} note={note} />
        )}
      </ul>

      <form onSubmit={addNote}>
        <input value={newNote} onChange={handleNoteChange} />
        <button type="submit">save</button>
      </form>   
    </div>
  )
}
export default App
```

---

## JavaScript in the Browser — Async Model

**JavaScript in the browser is single-threaded and non-blocking**, meaning it must use an **asynchronous** model (like the modern `fetch` API) to perform tasks like data fetching without "freezing" the user interface. Because the browser's engine can only execute one piece of code at a time, any long-running synchronous task would block the event loop, making the page unresponsive to user input; therefore, JavaScript handles I/O operations by triggering event handlers or promises only after a task is completed, allowing the main thread to stay free for UI rendering.

---

## Dependencies

We can install dependencies required for our project using npm in two ways because dependencies can either be a runtime dependency or a development dependency.

```bash
npm install axios
npm install json-server --save-dev
```

---

## Promises

### What a Promise Represents

A **Promise** is an object that represents the **future result of an asynchronous operation**.

Example:

```js
axios.post('http://localhost:3001/notes', noteObject)
```

The HTTP request takes time because data must travel to the server, the server must process it, and the response must come back. So JavaScript cannot immediately know the result. Instead, it returns a **Promise**.

### The Three Promise States

| State | Meaning |
|:------|:--------|
| **pending** | operation still running |
| **fulfilled** | operation finished successfully |
| **rejected** | operation failed |

When a promise becomes **fulfilled**, we say: the promise is resolved.

Technically:
- resolve → fulfilled
- reject → rejected

### What "Resolved with a Value" Means

A promise doesn't just resolve — it resolves **with a value**:

```js
resolve(value)
```

That value is then passed to **`.then()`**.

### Example Without Axios

```js
const promise = new Promise((resolve, reject) => {
  resolve(5)
})
```

Here `resolve(5)` means: "The asynchronous operation finished successfully, and the result is 5."

Now if we attach `.then()`:

```js
promise.then(result => {
  console.log(result)
})
```

Output: `5`

Because:

```
resolve(5)
        ↓
.then(result => ...)
        ↓
result = 5
```

### Applying This to Axios

When you do:

```js
axios.post(url, data)
```

Axios internally:
1. sends the HTTP request
2. waits for server response
3. constructs a response object

Then Axios **resolves the promise** like this conceptually:

```js
resolve(responseObject)
```

So the promise becomes: `fulfilled with responseObject`.

### What `.then()` Receives

```js
axios.post(url, data)
  .then(response => {
    console.log(response)
  })
```

This means `.then(onFulfilledFunction)`. When the promise resolves with `resolve(responseObject)`, JavaScript executes `onFulfilledFunction(responseObject)`, so `response = responseObject` and `console.log(response)` prints the **Axios response object**.

### The Full Flow

```
axios.post(...)
        ↓
Promise created
        ↓
HTTP request sent
        ↓
Server responds
        ↓
Axios builds responseObject
        ↓
Promise resolves with responseObject
        ↓
.then(response => ...)
        ↓
response = responseObject
```

### Key Takeaway

When we say "the promise resolves with a response object", we mean:

```js
resolve(responseObject)
```

And that object becomes the argument passed to **`.then()`**.

### Things to Remember About Promises

```js
axios.get('http://localhost:3001/notes') .then(response => { console.log('promise fulfilled') setNotes(response.data)
```

### How the Automatic Passing Works

`axios.get()` returns a Promise, which gets resolved with a response object when the server replies, and that response object is automatically passed as an argument to the function you provide as the first parameter of **`.then()`.**

It's literally: Promise → resolved with response → **`.then()` function receives it**.

That automatic passing is done by the **Promise mechanism built into JavaScript**. Here's what happens under the hood:

1. When you call `axios.get()`, Axios creates a **Promise**.
2. Axios internally calls `resolve(response)` when the server responds.
3. The **Promise system in JavaScript** takes that resolved value and automatically invokes all the functions attached via `.then()`, **passing the resolved value as the first argument**.

So you never call it yourself — the engine does it for you. Conceptually:

```js
// Behind the scenes
promise.resolve(responseObject)  // Axios does this
.then(function(callback) {
    callback(responseObject)    // JS passes the resolved value automatically
})
```

**Key point:** It's the Promise system that "delivers" the resolved value to the `.then()` callback. You just provide the function — JS handles the rest.

---

## Fetching Data with Axios and useEffect

Fetching data from the server can be done using the axios library:

```js
axios
  .get('http://localhost:3001/notes')
  .then(response => {
    console.log('promise fulfilled')
    console.log(response.data)
  })
```

Which can be used with the `useEffect` hook:

```js
useEffect(() => {
  console.log('useEffect called')
  axios
    .get('http://localhost:3001/persons')
    .then(response => {
      console.log('promise fulfilled')
      setPersons(response.data)
    })
}, [])
```

---

## REST Echo Response

When you send data to a server via PUT, the server saves it and responds with what it actually stored. What it stored might not be identical to what you sent — the server could have added, modified, or computed extra fields.

So always use `response.data` to update your local state, not the object you sent. This guarantees your client stays in sync with the server's truth, regardless of what the server did to the data behind the scenes.
