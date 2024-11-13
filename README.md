# TODO-Assignment

1. Setup
# Client Setup (ReactJS)
npx create-react-app client
cd client
npm install axios react-router-dom

# Server Setup (Node.js, Express)
mkdir server
cd server
npm init -y
npm install express bcryptjs jsonwebtoken cors uuid sqlite3 mongoose

2. Database Setup
# SQLite3:
const sqlite3 = require('sqlite3').verbose();
const db = new sqlite3.Database('./todo.db');

db.serialize(() => {
  db.run(`CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,
    name TEXT,
    email TEXT UNIQUE,
    password TEXT
  )`);

  db.run(`CREATE TABLE IF NOT EXISTS tasks (
    id TEXT PRIMARY KEY,
    userId TEXT,
    title TEXT,
    status TEXT,
    FOREIGN KEY(userId) REFERENCES users(id)
  )`);
});

module.exports = db;

# MongoDB Example:
const mongoose = require('mongoose');

const UserSchema = new mongoose.Schema({
  name: String,
  email: { type: String, unique: true },
  password: String,
});

module.exports = mongoose.model('User', UserSchema);

3. Backend API
# Basic Server Setup
const express = require('express');
const cors = require('cors');
const app = express();
const PORT = 5000;

app.use(cors());
app.use(express.json());

app.listen(PORT, () => console.log(`Server running on port ${PORT}`));


# Authentication Routes (Signup, Login)
const express = require('express');
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');
const { v4: uuidv4 } = require('uuid');
const db = require('../db'); // For SQLite3
const router = express.Router();
const JWT_SECRET = 'your_secret_key';

// Signup
router.post('/signup', async (req, res) => {
  const { name, email, password } = req.body;
  const hashedPassword = await bcrypt.hash(password, 10);
  const id = uuidv4();

  db.run(
    `INSERT INTO users (id, name, email, password) VALUES (?, ?, ?, ?)`,
    [id, name, email, hashedPassword],
    (err) => {
      if (err) return res.status(400).json({ error: 'User already exists' });
      const token = jwt.sign({ id }, JWT_SECRET);
      res.json({ token });
    }
  );
});

// Login
router.post('/login', (req, res) => {
  const { email, password } = req.body;

  db.get(`SELECT * FROM users WHERE email = ?`, [email], async (err, user) => {
    if (err || !user) return res.status(400).json({ error: 'Invalid email' });
    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) return res.status(400).json({ error: 'Invalid password' });

    const token = jwt.sign({ id: user.id }, JWT_SECRET);
    res.json({ token });
  });
});

module.exports = router;

# Middleware for JWT Verification
const jwt = require('jsonwebtoken');
const JWT_SECRET = 'your_secret_key';

const authMiddleware = (req, res, next) => {
  const token = req.header('Authorization');
  if (!token) return res.status(401).send('Access Denied');
  
  try {
    const verified = jwt.verify(token, JWT_SECRET);
    req.user = verified;
    next();
  } catch {
    res.status(400).send('Invalid Token');
  }
};

module.exports = authMiddleware;

# Todo Routes (CRUD Operations)
const express = require('express');
const { v4: uuidv4 } = require('uuid');
const db = require('../db'); // For SQLite3
const auth = require('../middleware/auth');
const router = express.Router();

// Create Task
router.post('/', auth, (req, res) => {
  const { title, status } = req.body;
  const id = uuidv4();
  const userId = req.user.id;
  db.run(
    `INSERT INTO tasks (id, userId, title, status) VALUES (?, ?, ?, ?)`,
    [id, userId, title, status],
    (err) => {
      if (err) return res.status(400).json({ error: 'Error adding task' });
      res.json({ id, title, status });
    }
  );
});

// Read Tasks
router.get('/', auth, (req, res) => {
  const userId = req.user.id;
  db.all(`SELECT * FROM tasks WHERE userId = ?`, [userId], (err, rows) => {
    if (err) return res.status(400).json({ error: 'Error fetching tasks' });
    res.json(rows);
  });
});

// Update Task
router.put('/:id', auth, (req, res) => {
  const { title, status } = req.body;
  db.run(
    `UPDATE tasks SET title = ?, status = ? WHERE id = ?`,
    [title, status, req.params.id],
    (err) => {
      if (err) return res.status(400).json({ error: 'Error updating task' });
      res.json({ message: 'Task updated' });
    }
  );
});

// Delete Task
router.delete('/:id', auth, (req, res) => {
  db.run(`DELETE FROM tasks WHERE id = ?`, [req.params.id], (err) => {
    if (err) return res.status(400).json({ error: 'Error deleting task' });
    res.json({ message: 'Task deleted' });
  });
});

module.exports = router;

4. Frontend (React)
# Basic Setup with React
Install axios for API requests.

import React, { useState, useEffect } from 'react';
import axios from 'axios';

const App = () => {
  const [tasks, setTasks] = useState([]);
  const [title, setTitle] = useState('');
  const [status, setStatus] = useState('pending');

  const fetchTasks = async () => {
    const response = await axios.get('/api/todos', {
      headers: { Authorization: localStorage.getItem('token') },
    });
    setTasks(response.data);
  };

  const addTask = async () => {
    await axios.post(
      '/api/todos',
      { title, status },
      { headers: { Authorization: localStorage.getItem('token') } }
    );
    fetchTasks();
  };

  useEffect(() => {
    fetchTasks();
  }, []);

  return (
    <div>
      <input
        type="text"
        placeholder="Task Title"
        value={title}
        onChange={(e) => setTitle(e.target.value)}
      />
      <button onClick={addTask}>Add Task</button>
      <ul>
        {tasks.map((task) => (
          <li key={task.id}>{task.title} - {task.status}</li>
        ))}
      </ul>
    </div>
  );
};

export default App;

5. Running the Application
# Frontend:
cd client
npm start

# Backend:
cd server
node index.js

# Conclusion:
This setup provides a solid foundation for my Todo App with authentication, CRUD operations, and profile management. I extended the functionality by adding more features like filtering tasks, enhancing the UI, and improving error handling.
