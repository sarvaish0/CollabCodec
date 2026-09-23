# CollabCodec 🚀

A **real-time collaborative code editor** that lets multiple developers work on the same project simultaneously. Think Google Docs, but for code — with a full file tree, multi-language execution, and live presence tracking.

---

## ✨ Features

- **Real-time Collaboration** — Multiple users can edit the same file simultaneously with conflict-free merging powered by [Yjs CRDT](https://github.com/yjs/yjs).
- **Monaco Editor** — The same editor that powers VS Code, with syntax highlighting for 10+ languages.
- **Live File Tree** — Create files and folders within a project; the tree updates across all collaborators.
- **Multi-language Code Runner** — Execute code directly in the browser:
  - **JavaScript** — Runs via `eval` with `console.log` capture.
  - **TypeScript** — Transpiled via Skypack CDN then evaluated.
  - **Python** — Runs in-browser using [Pyodide](https://pyodide.org/) (WebAssembly).
  - **HTML / CSS** — Rendered live inside an `<iframe>`.
  - **Markdown** — Rendered to HTML using `marked`.
  - **JSON** — Parsed and pretty-printed.
  - **Java / C / C++** — Executed remotely via the [Piston API](https://github.com/engineer-man/piston).
- **Live Presence** — See who else is in the project and which file they are currently viewing.
- **Join/Leave Notifications** — Toast notifications when collaborators enter or leave.
- **Persistent Projects** — Projects, file trees, and file contents are stored in MongoDB.
- **Auto-save with `Ctrl+S`** — Save file content to the database at any time.
- **Run with `Ctrl+Alt+N`** — Keyboard shortcut to run the active file.

---

## 🏗️ Architecture

```
CollabCodec/
├── backend/          # Node.js + Express + Socket.IO + MongoDB
│   └── src/
│       ├── index.js            # Entry point — HTTP server + DB connect
│       ├── app.js              # Express app, CORS, routes, error handlers
│       ├── controllers/
│       │   └── project.controller.js   # All project CRUD logic
│       ├── models/
│       │   ├── Project.model.js        # Project + file tree schema
│       │   └── User.model.js           # User schema (for future auth)
│       ├── routes/
│       │   └── projectRoutes.js        # REST API route definitions
│       ├── socket/
│       │   └── socket.js               # Socket.IO server + Yjs relay
│       ├── db/
│       │   └── connect.db.js           # Mongoose connection helper
│       └── utils/
│           ├── ApiError.util.js
│           ├── ApiResponse.util.js
│           └── asyncHandler.util.js
│
└── frontend/         # React + Vite + TailwindCSS
    └── src/
        ├── App.jsx                 # Router root
        ├── pages/
        │   └── LandingPage.jsx     # Create / Join project UI
        ├── components/
        │   ├── CodeSpace.jsx       # Main IDE layout (file tree + editor + terminal)
        │   ├── Editor.jsx          # Monaco editor + Yjs sync + socket logic
        │   ├── FileTree.jsx        # Collapsible file/folder tree
        │   └── Terminal.jsx        # Output panel
        ├── store/
        │   ├── useEditorStore.js   # Zustand store for file content & DB sync
        │   └── useFileTreeStore.js # Zustand store for file tree state
        ├── lib/
        │   ├── socket.js           # Singleton Socket.IO client
        │   └── axios.js            # Configured Axios instance
        └── utils/
            └── runtimeRunner.js    # Multi-language code execution logic
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| **Frontend Framework** | React 19 + Vite 6 |
| **Styling** | TailwindCSS 4 |
| **Editor** | Monaco Editor (`@monaco-editor/react`) |
| **Real-time Sync** | Yjs (CRDT) + Socket.IO |
| **State Management** | Zustand |
| **Routing** | React Router DOM v7 |
| **HTTP Client** | Axios |
| **Backend** | Node.js + Express 5 |
| **Database** | MongoDB + Mongoose |
| **WebSockets** | Socket.IO 4 |
| **Dev Server** | Nodemon |

---

## 🚀 Getting Started

### Prerequisites

- **Node.js** v18 or higher
- **npm** v9 or higher
- A **MongoDB** instance (Atlas or local)

---

### 1. Clone the repository

```bash
git clone https://github.com/your-username/CollabCodec.git
cd CollabCodec
```

---

### 2. Backend Setup

```bash
cd backend
npm install
```

Create a `.env` file in the `backend/` directory:

```env
PORT=3000
MONGODB_URI=mongodb+srv://<username>:<password>@<cluster>.mongodb.net
DB_NAME=CollabCodec
FRONTEND_URL=http://localhost:5173
BACKEND_URL=http://localhost:3000
```

Start the development server:

```bash
npm run dev
```

The backend will start at `http://localhost:3000`.

---

### 3. Frontend Setup

```bash
cd frontend
npm install
```

The frontend expects the backend at `http://localhost:3000`. If you change the backend port, update `frontend/src/lib/axios.js` and `frontend/src/lib/socket.js`.

Start the dev server:

```bash
npm run dev
```

The frontend will start at `http://localhost:5173`.

---

## 📡 REST API Reference

All routes are prefixed with `/api/projects`.

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/create` | Create a new project |
| `GET` | `/:projectId` | Validate a project ID (used for joining) |
| `GET` | `/:projectId/fetchFiles` | Fetch the file tree (folders first, sorted) |
| `POST` | `/:projectId/addFile` | Add a new file to the project |
| `POST` | `/:projectId/addFolder` | Add a new folder to the project |
| `POST` | `/:projectId/saveFileContent` | Save (overwrite) a file's content |
| `GET` | `/:projectId/fetchFileContent` | Fetch a specific file's content |

---

## 🔌 Socket.IO Events

### Client → Server

| Event | Payload | Description |
|---|---|---|
| `join-project-room` | `roomId` (string) | Join a project's collaboration room |
| `leave-project-room` | `projectId` (string) | Leave a project room |
| `join-file-room` | `fileRoomId, projectId, filePath` | Join a specific file's editing room |
| `leave-file-room` | `projectId, fileRoomId` | Leave a file room |
| `y-update` | `Uint8Array` | Send a local Yjs CRDT update to the server |

### Server → Client

| Event | Payload | Description |
|---|---|---|
| `project-user-joined` | `{ username }` | A user joined the project |
| `project-user-left` | `{ username }` | A user left the project |
| `project-user-map` | `Array<{ username, onFile }>` | Updated presence map for the project |
| `y-update` | `Uint8Array` | A Yjs CRDT update from another user |

---

## 📁 Data Model

### Project

```js
{
  projectName: String,          // required
  files: [
    {
      path: String,             // e.g. "MyProject/src/index.js"
      type: "file" | "folder",
      content: String           // file content, empty for folders
    }
  ],
  owner: ObjectId,              // ref: User (optional, for future auth)
  collaborators: [
    {
      user: ObjectId,           // ref: User
      permission: "read" | "write"
    }
  ],
  createdAt: Date,
  updatedAt: Date
}
```

File paths use `/`-separated strings (e.g. `MyProject/src/utils/helper.js`). The frontend reconstructs the folder hierarchy by splitting on `/`.

---

## ⌨️ Keyboard Shortcuts

| Shortcut | Action |
|---|---|
| `Ctrl + S` | Save the current file to the database |
| `Ctrl + Alt + N` | Run the current file |

---

## 🤝 How Collaboration Works

1. **User A** creates a project → gets a unique **Project ID**.
2. **User B** pastes the Project ID into "Join Project" → both enter the same editor session.
3. When either user opens a file, they both join a **file room** (`room-<projectId>-<filePath>`).
4. Every keystroke is encoded as a **Yjs update** (a compact binary diff).
5. The server receives the update, applies it to an in-memory `Y.Doc`, and **broadcasts** it to all other users in that file room.
6. When the last user leaves a file room and the project room is empty for 10 seconds, the in-memory `Y.Doc` is garbage-collected.

---

## 🔮 Roadmap

- [ ] User authentication (JWT / OAuth)
- [ ] Persistent Yjs state in MongoDB (so new joiners see existing content without a live peer)
- [ ] Real-time cursor / caret positions per user
- [ ] File rename and delete
- [ ] Integrated terminal (PTY via `node-pty`)
- [ ] Multiple themes for the Monaco editor
- [ ] Share link with permission controls (read-only vs. write)

---

## 📄 License

This project is licensed under the **ISC License**.
