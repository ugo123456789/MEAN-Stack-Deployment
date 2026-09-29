# MEAN Stack Book Management App on AWS EC2

## Project Overview
Development and deployment of a Book Management application built 
with the MEAN (MongoDB, Express.js, AngularJS, Node.js) stack on 
an AWS EC2 instance as part of the Steghub DevOps Cloud Engineering 
program.

---

## What is the MEAN Stack?
- **M** — MongoDB: NoSQL document database
- **E** — Express.js: Backend web framework for Node.js
- **A** — AngularJS: Frontend JavaScript framework
- **N** — Node.js: JavaScript runtime environment

---

## MEAN vs MERN — Key Difference

| | MERN | MEAN |
|---|---|---|
| **Frontend** | React.js | AngularJS |
| **Architecture** | Component based | MVC (Model View Controller) |
| **Data binding** | One way | Two way |
| **Build tool** | Vite | None needed |
| **Learning curve** | Moderate | Steeper |

---

## Architecture

```
Browser (AngularJS Frontend)
        │
        │  HTTP requests (GET, POST, DELETE)
        ▼
Express.js (Backend - Port 3300)
        │
        │  Mongoose queries
        ▼
MongoDB (Local - Port 27017)
```

---

## Application Features

- Add new books with name, ISBN, author and page count
- View all books in a table
- Delete books by ISBN
- Data persisted in MongoDB

---

## Prerequisites

- AWS Account
- EC2 Instance (Ubuntu 22.04)
- SSH Key Pair (.pem file)
- Node.js knowledge
- Basic AngularJS knowledge

---

## Step 1 — Launch EC2 Instance

Created Ubuntu EC2 instance on AWS with the following
Security Group inbound rules:

| Type | Protocol | Port |
|---|---|---|
| SSH | TCP | 22 |
| HTTP | TCP | 80 |
| Custom TCP | TCP | 3300 |

Connected via SSH:

```bash
chmod 400 LAMP.pem
ssh -i LAMP.pem ubuntu@<public-ip>
```

---

## Step 2 — Install Node.js and npm

```bash
sudo apt update
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install nodejs -y
node -v
npm -v
```

---

## Step 3 — Install MongoDB

```bash
# Import MongoDB key
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor

# Add repository
echo "deb [ arch=amd64,arm64 \
signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] \
https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" \
| sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Install
sudo apt update
sudo apt install -y mongodb-org

# Start and enable
sudo systemctl start mongod
sudo systemctl enable mongod
```

---

## Step 4 — Project Setup

```bash
mkdir Books && cd Books
npm init -y
npm install express body-parser mongoose path
```

### Project Structure

```
Books/
├── server.js              ← Express server entry point
├── package.json           ← Project dependencies
├── apps/
│   ├── routes.js          ← API endpoints
│   └── models/
│       └── book.js        ← Mongoose schema and model
└── public/
    ├── index.html         ← AngularJS frontend
    └── script.js          ← AngularJS controller
```

---

## Step 5 — Express Server (server.js)

```javascript
const express = require('express');
const bodyParser = require('body-parser');
const mongoose = require('mongoose');
const path = require('path');

const app = express();
const PORT = process.env.PORT || 3300;

mongoose.connect('mongodb://localhost:27017/test', {})
.then(() => console.log('MongoDB connected'))
.catch(err => console.error('MongoDB connection error:', err));

app.use(express.static(path.join(__dirname, 'public')));
app.use(bodyParser.json());

require('./apps/routes')(app);

app.listen(PORT, () => {
    console.log(`Server up: http://localhost:${PORT}`);
});
```

---

## Step 6 — Book Model (apps/models/book.js)

```javascript
const mongoose = require('mongoose');

const bookSchema = new mongoose.Schema({
    name:   { type: String, required: true },
    isbn:   { type: String, required: true, unique: true, index: true },
    author: { type: String, required: true },
    pages:  { type: Number, required: true, min: 1 }
}, {
    timestamps: true
});

module.exports = mongoose.model('Book', bookSchema);
```

---

## Step 7 — API Routes (apps/routes.js)

```javascript
const Book = require('./models/book');
const path = require('path');

module.exports = function(app) {

    // GET all books
    app.get('/book', async (req, res) => {
        try {
            const books = await Book.find();
            res.json(books);
        } catch (err) {
            res.status(500).json({ message: 'Error fetching books' });
        }
    });

    // POST create a book
    app.post('/book', async (req, res) => {
        try {
            const book = new Book({
                name:   req.body.name,
                isbn:   req.body.isbn,
                author: req.body.author,
                pages:  req.body.pages
            });
            const savedBook = await book.save();
            res.status(201).json({
                message: 'Successfully added book',
                book: savedBook
            });
        } catch (err) {
            res.status(400).json({ message: 'Error adding book' });
        }
    });

    // DELETE a book by ISBN
    app.delete('/book/:isbn', async (req, res) => {
        try {
            const result = await Book.findOneAndDelete({
                isbn: req.params.isbn
            });
            if (!result) {
                return res.status(404).json({ message: 'Book not found' });
            }
            res.json({ message: 'Successfully deleted the book' });
        } catch (err) {
            res.status(500).json({ message: 'Error deleting book' });
        }
    });

    // Serve frontend for all other routes
    app.get('{*path}', (req, res) => {
        res.sendFile(path.join(__dirname, '../public', 'index.html'));
    });
};
```

---

## Step 8 — AngularJS Frontend (public/script.js)

```javascript
angular.module('myApp', [])
    .controller('myCtrl', function($scope, $http) {

        function fetchBooks() {
            $http.get('/book')
                .then(response => {
                    $scope.books = response.data;
                });
        }

        fetchBooks();

        $scope.add_book = function() {
            const newBook = {
                name:   $scope.Name,
                isbn:   $scope.Isbn,
                author: $scope.Author,
                pages:  $scope.Pages
            };
            $http.post('/book', newBook)
                .then(() => {
                    fetchBooks();
                    $scope.Name = $scope.Isbn = 
                    $scope.Author = $scope.Pages = '';
                });
        };

        $scope.del_book = function(book) {
            $http.delete(`/book/${book.isbn}`)
                .then(() => fetchBooks());
        };
    });
```

---

## Step 9 — Running the Application

```bash
# Start MongoDB
sudo systemctl start mongod

# Start Express server
node server.js
```

Output:
```
MongoDB connected ✅
Server up: http://localhost:3300 ✅
```

Access the application:
```
http://<your-ec2-public-ip>:3300
```

---

## Errors Encountered and Fixed

### 1. Unprotected Private Key File
**Error:** WARNING: UNPROTECTED PRIVATE KEY FILE
**Cause:** .pem file permissions too open
**Fix:**
```bash
chmod 400 LAMP.pem
```

### 2. PathError on Wildcard Route
**Error:** PathError: Missing parameter name at index 1
**Cause:** Old wildcard syntax broken in newer Express versions
**Fix:**
```javascript
// Old ❌
app.get('*', ...)

// New ✅
app.get('{*path}', ...)
```

### 3. MongoDB Connection Refused
**Error:** connect ECONNREFUSED 127.0.0.1:27017
**Cause:** MongoDB service not running
**Fix:**
```bash
sudo systemctl start mongod
```

### 4. Server Not Binding to Port 3300
**Error:** curl: Failed to connect to localhost port 3300
**Cause:** Dependencies not listed in package.json
**Fix:**
```bash
npm install express body-parser mongoose path
```

### 5. index.html Not Found
**Cause:** index.html placed in root directory instead of public/
**Fix:**
```bash
mv index.html public/
```

---

## Key Learnings

- Always run `npm install <package>` from correct directory
  to ensure dependencies are saved to package.json
- MongoDB must be started before the Express server
- AngularJS uses two way data binding —
  changes in model automatically update the view
- Express wildcard syntax changed in newer versions —
  use `{*path}` instead of `*`
- Verify dependencies in package.json before pushing to GitHub
- MongoDB runs locally on port 27017 by default

---

## Screenshots

![EC2 Instance Running](screenshots/ec2-instance.png)
![Book App Loading](screenshots/book-app.png)
![Book List Displaying](screenshots/adding-a-book.png)

---

*Project completed as part of Steghub  Cloud Engineering Program*
*Author: [Ugonna Onyekwuluje]*
*Date: [15/09/2026]*