---
track: "Unit 2"
title: "The Backend Rises: Building and Testing Restful APIs Like A Professional Engineer"
type: "lecture"
---

## Lesson: Building an API with Express and Mongoose

### Terminal Objectives:
1. To understand the structure and concepts of building an API with Express and Mongoose.
2. To comprehend the idea of MVC (Model-View-Controller) architecture and how it is implemented in Express applications.
3. To grasp the concepts of Latency & Throughput and their significance in API performance.
4. To understand Authentication (Auth) in Express and Mongoose.
5. To learn how to serve static files such as CSS and images with `express.static`.
6. To learn how to test APIs using Jest and Supertest.
7. Seperate app into models, controllers, routes
8. Add Artillery for load testing
9. Test every route with Jest & Supertest & Mongodb-memory-server
10. Add morgan for logging

---

<center>

<div style="display: flex; justify-content: space-between;">

<img width="871" height="423" alt="mvc" src="https://github.com/user-attachments/assets/0aa4a63f-ecc8-4872-83cb-28da34501cf5" />

![mvcmeme](https://github.com/user-attachments/assets/62b80a79-f865-4cd2-b8f6-63e919a03ade)

</div>
</center>

### Explanation:

#### API with Express and Mongoose
Express is a fast, unopinionated, minimalist web framework for Node.js that is used for building web applications and APIs. Mongoose is an Object Data Modeling (ODM) library for MongoDB and Node.js. It manages relationships between data, provides schema validation, and is used to translate between objects in code and the representation of those objects in MongoDB.

#### MVC
Model-View-Controller (MVC) is a software design pattern commonly used for developing applications that divides the related program logic into three interconnected elements. The model represents the data and the rules that govern access to and updates of this data. The view renders the contents of the model. It specifies exactly how the model data should be presented. The controller handles user input and calls model and view methods accordingly.

#### Latency & Throughput
Latency is the delay before a transfer of data begins following an instruction for its transfer, whereas Throughput is the amount of data moved successfully from one place to another in a given time period. In terms of APIs, minimizing latency and maximizing throughput are crucial for ensuring a smooth and fast user experience.

#### Authentication
Authentication is the process of verifying the identity of a user by obtaining some sort of credentials and using those credentials to verify the user's identity. If the credentials are valid, the authorization process starts.

#### Authorization
Authorization is the process that determines what an authenticated user is allowed to do. While authentication verifies who a user is, authorization verifies what they have access to.

For example, in your API:

Authentication lets a user log in with a valid email and password.

Authorization decides whether that user can edit, update, or delete a specific resource.

#### express.static
`express.static` is a built-in middleware function in Express. It is used to serve static files, such as images, CSS, and JavaScript, directly to the client.

#### Jest and Supertest
Jest is a JavaScript testing framework with a focus on simplicity, and Supertest is a high-level abstraction for testing HTTP built on top of `superagent`. They can be used together to write tests for your Express APIs.

---

### Building an API with Express and Mongoose following MVC Architecture

This lesson is intended to teach you how to build a performant API using Express and Mongoose, applying the MVC (Model-View-Controller) architecture. We'll also be incorporating other technologies like dotenv, bcrypt, jsonwebtoken, mongodb-memory-server for testing, and using morgan for logging.

## Initial Setup 

Firstly, let's create a new Node.js project:

```bash
cd fruits
touch app.js
```

### Why server.js && app.js

1. we will use app.js to store the app object from express
1. we will use server.js to import the app, and setup our connections with app & mongoose
1. this will allow us to test app.js independent of mongodb

Now, install the necessary packages:

```bash
npm i bcrypt jsonwebtoken morgan
npm i -D jest supertest mongodb-memory-server artillery@1.7.9
```


## Building the API with Mongoose and Express 

Create a new file `models/user.js` for our User model:

```js
const mongoose = require('mongoose')
const bcrypt = require('bcrypt')
const jwt = require('jsonwebtoken')

const userSchema = new mongoose.Schema({
  name: String,
  email: String,
  password: String
})

userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 8)
  }
  next()
})

userSchema.methods.generateAuthToken = async function() {
  const token = jwt.sign({ _id: this._id }, 'secret')
  return token
}

const User = mongoose.model('User', userSchema)

module.exports = User
```

Now, we need to structure our code following MVC architecture. 

First, we'll separate our routes and controllers. Create a new file `routes/userRoutes.js`:

```js
const express = require('express')
const router = express.Router()
const userController = require('../controllers/userController')

router.post('/', userController.createUser)
router.post('/login', userController.loginUser)
router.put('/:id', userController.updateUser)
router.delete('/:id', userController.auth, userController.deleteUser)

module.exports = router
```

Now, create a new file `controllers/userController.js`:

```js
const User = require('../models/user')
const bcrypt = require('bcrypt')
const jwt = require('jsonwebtoken')

// instead of creating an object we can use the exports object directly
// this is how

exports.auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization').replace('Bearer ', '')
    const data = jwt.verify(token, 'secret')
    const user = await User.findOne({ _id: data._id })
    if (!user) {
      throw new Error()
    }
    req.user = user
    next()
  } catch (error) {
    res.status(401).send('Not authorized')
  }
}

exports.createUser = async (req, res) => {
  try{
    const user = new User(req.body)
    await user.save()
    const token = await user.generateAuthToken()
    res.json({ user, token })
  } catch(error){
    res.status(400).json({message: error.message})
  }
}

exports.loginUser = async (req, res) => {
  try{
    const user = await User.findOne({ email: req.body.email })
    if (!user || !await bcrypt.compare(req.body.password, user.password)) {
      res.status(400).send('Invalid login credentials')
    } else {
      const token = await user.generateAuthToken()
      res.json({ user, token })
    }
  } catch(error){
    res.status(400).json({message: error.message})
  }
}

exports.updateUser = async (req, res) => {
  try{
    const updates = Object.keys(req.body)
    const user = await User.findOne({ _id: req.params.id })
    updates.forEach(update => user[update] = req.body[update])
    await user.save()
    res.json(user)
  }catch(error){
    res.status(400).json({message: error.message})
  }
  
}

exports.deleteUser = async (req, res) => {
  try{
    await req.user.deleteOne()
    res.json({ message: 'User deleted' })
  }catch(error){
    res.status(400).json({message: error.message})
  }
}
```

In app.js, include the user routes, app setup and middleware

```js
const express = require('express')
const morgan = require('morgan')
const userRoutes = require('./routes/userRoutes')
const fruitsRouter = require('./controllers/routeController')
const app = express()

app.set('view engine', 'jsx')
app.engine('jsx', jsxEngine())

app.use(express.json()) // this is new this for the api
app.use(express.urlencoded({ extended: true })) // req.body
app.use(methodOverride('_method')) // <====== add method override
app.use((req, res, next) => {
    res.locals.data = {}
    next()
})
app.use(express.static('public'))
app.use(morgan('combined'))
app.use('/users', userRoutes)
app.use('/fruits', fruitsRouter)

module.exports = app
```

Finally, in your `server.js`:

```js
require('dotenv').config()
const app = require('./app')
const db = require('./models/db')
const PORT = process.env.PORT || 3000

db.once('open', () => {
    console.log('connected to mongo')
})
db.on('error', (error) => {
  console.error(error.message)
})

app.listen(PORT, () => {
    console.log(`We in the building ${PORT}`)
})


```

## Testing the API 

To set up testing for the API, start by configuring Jest in your `package.json`:

```json
"scripts": {
  "start": "node server.js",
  "test": "jest",
  "dev": "nodemon",
  "load": "artillery run artillery.yml"
},
"jest": {
  "testEnvironment": "node"
}
```



Create a new file `tests/user.test.js`:

```js
const request = require('supertest')
const mongoose = require('mongoose')
const { MongoMemoryServer } = require('mongodb-memory-server')
const app  = require('../app')
const server = app.listen(8080, () => console.log('Testing on PORT 8080'))
const User = require('../models/user')
let mongoServer

beforeAll(async () => {
  mongoServer = await MongoMemoryServer.create()
  await mongoose.connect(mongoServer.getUri(), { useNewUrlParser: true, useUnifiedTopology: true })
})

afterAll(async () => {
  await mongoose.connection.close()
  mongoServer.stop()
  server.close()
})

afterAll((done) => done())

describe('Test the users endpoints', () => {
  test('It should create a new user', async () => {
    const response = await request(app)
      .post('/users')
      .send({ name: 'John Doe', email: 'john.doe@example.com', password: 'password123' })
    
    expect(response.statusCode).toBe(200)
    expect(response.body.user.name).toEqual('John Doe')
    expect(response.body.user.email).toEqual('john.doe@example.com')
    expect(response.body).toHaveProperty('token')
  })

  test('It should login a user', async () => {
    const user = new User({ name: 'John Doe', email: 'john.doe@example.com', password: 'password123' })
    await user.save()

    const response = await request(app)
      .post('/users/login')
      .send({ email: 'john.doe@example.com', password: 'password123' })
    
    expect(response.statusCode).toBe(200)
    expect(response.body.user.name).toEqual('John Doe')
    expect(response.body.user.email).toEqual('john.doe@example.com')
    expect(response.body).toHaveProperty('token')
  })

  test('It should update a user', async () => {
    const user = new User({ name: 'John Doe', email: 'john.doe@example.com', password: 'password123' })
    await user.save()
    const token = await user.generateAuthToken()

    const response = await request(app)
      .put(`/users/${user._id}`)
      .set('Authorization', `Bearer ${token}`)
      .send({ name: 'Jane Doe', email: 'jane.doe@example.com' })
    
    expect(response.statusCode).toBe(200)
    expect(response.body.name).toEqual('Jane Doe')
    expect(response.body.email).toEqual('jane.doe@example.com')
  })

  test('It should delete a user', async () => {
    const user = new User({ name: 'John Doe', email: 'john.doe@example.com', password: 'password123' })
    await user.save()
    const token = await user.generateAuthToken()

    const response = await request(app)
      .delete(`/users/${user._id}`)
      .set('Authorization', `Bearer ${token}`)
    
    expect(response.statusCode).toBe(200)
    expect(response.body.message).toEqual('User deleted')
  })
})
```

## Performance Analysis

Morgan is a middleware for logging HTTP requests, and Artillery is a modern, powerful, easy-to-use load-testing framework. Both are useful for understanding the behavior and performance of your API.

add to your `app.js`:

```js
const morgan = require('morgan')

// ... existing code ...

app.use(morgan('combined'))
```


Create a `artillery.yml` file in your root directory and populate it with the following:

```yml
config:
  target: 'http://localhost:3000'
  phases:
    - duration: 60
      arrivalRate: 20
scenarios:
  - flow:
      - post:
          url: "/users"
          json:
            name: "Test"
            email: "test@example.com"
            password: "Password123"
```

Then, run the test:

```js
  "scripts": {
    "test": "jest",
    "start": "node server.js",
    "load": "artillery run artillery.yml",
    "dev": "nodemon"
  }
```

```bash
npm run load
```

That's all! With this setup, you can easily build APIs using Express and Mongoose following the MVC architecture, have a secure way to store passwords, authenticate users, log HTTP requests, and test the performance of your API under load. You also have a testing setup with a MongoDB in-memory server for faster feedback.

Alright, let's start with the file overview in a markdown table:

| Filename           | Path             | Purpose                                                           |
|------------------  |----------------- |----------------------------------------------------------------- |
| `server.js`           | `./server.js`       | Entry point for the application.                                 |
| `app.js`           | `./app.js`       | Express Application Object.                                 |
| `user.js`          | `./models/user.js` | Defines the User model schema for MongoDB/Mongoose.                      |
| `userController.js` | `./controllers/userController.js` | Handles the logic related to user actions.                   |
| `userRoutes.js`     | `./routes/userRoutes.js` | Defines the endpoints for user-related operations.            |
| `package.json`      | `./package.json` | Lists the project dependencies and metadata.                      |
| `user.test.js`      | `./tests/user.test.js` | Contains the tests related to user operations.                 |
| `artillery.yml`     | `./artillery.yml` | Configuration file for Artillery load tests.                      |
| `.env`     | `./.env` | Enviorment variable files.                      |
| `.gitignore`     | `./.gitignore` | List paths to files and folders we want git not to track for privacy or configuration reasons.                      |

Now, let's follow up with the code for each of these files:

## `server.js`

```js
require('dotenv').config()
const app = require('./app')
const db = require('./models/db')
const PORT = process.env.PORT || 3000

db.once('open', () => {
    console.log('connected to mongo')
})
db.on('error', (error) => {
  console.error(error.message)
})

app.listen(PORT, () => {
    console.log(`We in the building ${PORT}`)
})


```

## `app.js`

```js
const express = require('express')
const morgan = require('morgan')
const userRoutes = require('./routes/userRoutes')
const fruitsRouter = require('./controllers/routeController')
const app = express()

app.set('view engine', 'jsx')
app.engine('jsx', jsxEngine())

app.use(express.json()) // this is new this for the api
app.use(express.urlencoded({ extended: true })) // req.body
app.use(methodOverride('_method')) // <====== add method override
app.use((req, res, next) => {
    res.locals.data = {}
    next()
})
app.use(express.static('public'))
app.use(morgan('combined'))
app.use('/users', userRoutes)
app.use('/fruits', fruitsRouter)

module.exports = app
```

## `models/user.js`

```js
const mongoose = require('mongoose')
const bcrypt = require('bcrypt')
const jwt = require('jsonwebtoken')

const userSchema = new mongoose.Schema({
  name: String,
  email: String,
  password: String
})

userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 8)
  }
  next()
})

userSchema.methods.generateAuthToken = async function() {
  const token = jwt.sign({ _id: this._id }, 'secret')
  return token
}

const User = mongoose.model('User', userSchema)

module.exports = User
```

## `controllers/userController.js` (can be broken up into data, view and api)

```js
const User = require('../models/user')
const bcrypt = require('bcrypt')
const jwt = require('jsonwebtoken')

exports.auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization').replace('Bearer ', '')
    const data = jwt.verify(token, 'secret')
    const user = await User.findOne({ _id: data._id })
    if (!user) {
      throw new Error()
    }
    req.user = user
    next()
  } catch (error) {
    res.status(401).send('Not authorized')
  }
}

exports.createUser = async (req, res) => {
  try{
    const user = new User(req.body)
    await user.save()
    const token = await user.generateAuthToken()
    res.json({ user, token })
  } catch(error){
    res.status(400).json({message: error.message})
  }
}

exports.loginUser = async (req, res) => {
  try{
    const user = await User.findOne({ email: req.body.email })
    if (!user || !await bcrypt.compare(req.body.password, user.password)) {
      res.status(400).send('Invalid login credentials')
    } else {
      const token = await user.generateAuthToken()
      res.json({ user, token })
    }
  } catch(error){
    res.status(400).json({message: error.message})
  }
}

exports.updateUser = async (req, res) => {
  try{
    const updates = Object.keys(req.body)
    const user = await User.findOne({ _id: req.params.id })
    updates.forEach(update => user[update] = req.body[update])
    await user.save()
    res.json(user)
  }catch(error){
    res.status(400).json({message: error.message})
  }
  
}

exports.deleteUser = async (req, res) => {
  try{
    await req.user.deleteOne()
    res.json({ message: 'User deleted' })
  }catch(error){
    res.status(400).json({message: error.message})
  }
}
```

## `controllers/userRoutes.js`

```js
const express = require('express')
const router = express.Router()
const userController = require('../controllers/userController')

router.post('/', userController.createUser)
router.post('/login', userController.loginUser)
router.put('/:id', userController.updateUser)
router.delete('/:id', userController.auth, userController.deleteUser)

module.exports = router
```

## `package.json`

Your `package.json` should list all the project dependencies and metadata. It would look something like this after installing all the necessary dependencies mentioned in the lesson:


## `tests/user.test.js`

```js
const request = require('supertest')
const mongoose = require('mongoose')
const { MongoMemoryServer } = require('mongodb-memory-server')
const app  = require('../app')
const server = app.listen(8080, () => console.log('Testing on PORT 8080'))
const User = require('../models/user')
let mongoServer

beforeAll(async () => {
  mongoServer = await MongoMemoryServer.create()
  await mongoose.connect(mongoServer.getUri(), { useNewUrlParser: true, useUnifiedTopology: true })
})

afterAll(async () => {
  await mongoose.connection.close()
  mongoServer.stop()
  server.close()
})

afterAll((done) => done())

describe('Test the users endpoints', () => {
  test('It should create a new user', async () => {
    const response = await request(app)
      .post('/users')
      .send({ name: 'John Doe', email: 'john.doe@example.com', password: 'password123' })
    
    expect(response.statusCode).toBe(200)
    expect(response.body.user.name).toEqual('John Doe')
    expect(response.body.user.email).toEqual('john.doe@example.com')
    expect(response.body).toHaveProperty('token')
  })

  test('It should login a user', async () => {
    const user = new User({ name: 'John Doe', email: 'john.doe@example.com', password: 'password123' })
    await user.save()

    const response = await request(app)
      .post('/users/login')
      .send({ email: 'john.doe@example.com', password: 'password123' })
    
    expect(response.statusCode).toBe(200)
    expect(response.body.user.name).toEqual('John Doe')
    expect(response.body.user.email).toEqual('john.doe@example.com')
    expect(response.body).toHaveProperty('token')
  })

  test('It should update a user', async () => {
    const user = new User({ name: 'John Doe', email: 'john.doe@example.com', password: 'password123' })
    await user.save()
    const token = await user.generateAuthToken()

    const response = await request(app)
      .put(`/users/${user._id}`)
      .set('Authorization', `Bearer ${token}`)
      .send({ name: 'Jane Doe', email: 'jane.doe@example.com' })
    
    expect(response.statusCode).toBe(200)
    expect(response.body.name).toEqual('Jane Doe')
    expect(response.body.email).toEqual('jane.doe@example.com')
  })

  test('It should delete a user', async () => {
    const user = new User({ name: 'John Doe', email: 'john.doe@example.com', password: 'password123' })
    await user.save()
    const token = await user.generateAuthToken()

    const response = await request(app)
      .delete(`/users/${user._id}`)
      .set('Authorization', `Bearer ${token}`)
    
    expect(response.statusCode).toBe(200)
    expect(response.body.message).toEqual('User deleted')
  })
})
```

## `artillery.yml`

```yaml
config:
  target: 'http://localhost:3000'
  phases:
    - duration: 60
      arrivalRate: 20
scenarios:
  - flow:
      - post:
          url: "/users"
          json:
            name: "Test"
            email: "test@example.com"
            password: "Password123"
```


## Fruits API Controller 

```js
const RESOURCE_PATH = '/fruits'
const apiController = {
  index(req, res, next){
    res.json(res.locals.data.fruits)
  },
  show(req, res, next){
    res.json(res.locals.data.fruit)
  },
  destroy(req, res, next){
    res.status(204).json({msg: 'fruit sucessfully deleted'})
   }
}

module.exports = apiController
```

## Fruit Router Update

```js
const express = require('express');
const router = express.Router();
const viewController = require('./viewController.js')
const dataController = require('./dataController.js')
const apiController = require('./apiController.js')

// add routes
// Index
router.get('/', dataController.index, viewController.index);
// New
router.get('/new', viewController.newView );
// Delete
router.delete('/:id', dataController.destroy, viewController.redirectHome);
// Update
router.put('/:id', dataController.update, viewController.redirectShow);
// Create
router.post('/', dataController.create, viewController.redirectHome);
// Edit
router.get('/:id/edit', dataController.show, viewController.edit);
// Show
router.get('/:id', dataController.show, viewController.show);
// export router
// Index
router.get('/api', dataController.index, apiController.index);
// Delete
router.delete('/api/:id', dataController.destroy, apiController.destroy);
// Update
router.put('/api/:id', dataController.update, apiController.show);
// Create
router.post('/api', dataController.create, apiController.show);
// Show
router.get('/api/:id', dataController.show, apiController.show);
module.exports = router;
```

## Conclusion: API Testing and Best Practices

In conclusion, it is vital to remember that the core job of a server is to respond to HTTP requests. These requests are composed of a method and a path. The method, such as GET or POST, indicates the type of operation to be executed. The path, which follows the domain in the URL, provides the specific location of the data or resource.

HTTP defines five primary methods, each corresponding to one of the CRUD functionalities. GET retrieves data, POST adds data, PUT and PATCH modify existing data (with PUT replacing all data and PATCH modifying only certain fields), and DELETE removes data. Understanding these methods and their differences is essential for effective API operation and testing.

A crucial part of web development is crafting RESTful routes, which pair a method and a path to define an action. These routes offer a clear, standardized way to manage resources, making them easier to test and debug.

Testing is crucial for ensuring the robustness and reliability of your API. Functional testing, which includes both positive and negative tests, verifies that the API operates as expected under different conditions. These tests should evaluate the status code, payload, headers, and performance of the API, as well as the system's data state. Additionally, functional testing should include destructive testing to assess how the API handles failures and errors.

However, passing functional tests, while indicating a good level of maturity for an API, is not enough to ensure high quality and reliability. Testing should go beyond the functional to include aspects such as performance, security, and usability.

As you continue to develop and test your APIs, always keep these principles and best practices in mind. They will guide you towards creating robust, efficient, and reliable web services.


### RESTful HTTP Methods 

| Method | Crud Functionality | DB Action            |
| ------ | ------------------ | -------------------- |
| GET    | read               | retrieve data        |
| POST   | create             | add data             |
| PUT    | update             | modify existing data |
| PATCH  | update             | modify existing data |
| DELETE | delete             | delete existing data |

### RESTful Routes 

| Action | Method | Path           | Action                                                               |
| ------ | ------ | -------------- | -------------------------------------------------------------------- |
| index  | GET    | `/engineers`   | Read information about all engineers                                 |
| create | POST   | `/engineers`   | Create a new engineer                                                |
| show   | GET    | `/engineers/1` | Read information about the engineer whose ID is 1                    |
| update | PUT    | `/engineers/1` | Update the existing engineer whose ID is 1 with all new content      |
| update | PATCH  | `/engineers/1` | Update the existing engineer whose ID is 1 with partially new content|
| destroy| DELETE | `/engineers/1` | Delete the existing engineer whose ID is 1                           |

### Types of Bugs that API testing detects  

Here’s a more readable and structured version of your **Functional Testing** table, using logical grouping, indentation, and clearer visual hierarchy:

---

### ✅ Functional Testing Breakdown

| **Test Type**                        | **Action**                                                      | **Validation**                                                          |
| ------------------------------------ | --------------------------------------------------------------- | ----------------------------------------------------------------------- |
| **Positive Tests (Required Params)** | Execute API with valid required parameters                      |                                                                         |
|                                      |                                                                 | ✅ Status Code: 2XX                                                      |
|                                      |                                                                 | ✅ Payload: Valid JSON, matches data model                               |
|                                      |                                                                 | ✅ Data/System State: GET doesn't change data, others modify as expected |
|                                      |                                                                 | ✅ Headers: Correct, no sensitive leaks                                  |
|                                      |                                                                 | ✅ Performance: Fast response                                            |
| **Positive Tests (Optional Params)** | Execute API with valid required + optional                      |                                                                         |
|                                      |                                                                 | ✅ Status Code, Payload, State, Headers, and Performance match above     |
| **Negative Tests (Valid Input)**     | Attempt illegal operations with valid input                     |                                                                         |
|                                      |                                                                 | ❌ Status Code: Should return 4XX or 5XX                                 |
|                                      |                                                                 | ❌ Payload: Should contain meaningful error response                     |
|                                      |                                                                 | ✅ Headers: Should still be clean                                        |
|                                      |                                                                 | ✅ Performance: Errors should be timely and graceful                     |
| **Negative Tests (Invalid Input)**   | Send invalid input (e.g., bad data types)                       |                                                                         |
|                                      |                                                                 | Same validations as above (Status, State, Headers, Performance)         |
| **Destructive Testing**              | Intentionally break the API (e.g., malformed request, overload) |                                                                         |
|                                      |                                                                 | ❌ Status Code: Fail gracefully with 4XX or 5XX                          |
|                                      |                                                                 | ❌ State: No unintended data corruption                                  |
|                                      |                                                                 | ✅ Headers: Still secure                                                 |
|                                      |                                                                 | ✅ Performance: Still responds predictably                               |

---

### 🔁 Summary

* **Positive tests** validate expected behavior.
* **Negative tests** validate error handling.
* **Destructive tests** simulate edge cases and failures.



