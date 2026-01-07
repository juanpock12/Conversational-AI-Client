# AI Chat Application (Svelte + Tauri)

This project is an AI-powered chat application built with **Svelte** and deployed as a **cross-platform desktop application using Tauri**.

---

## Prerequisites

Ensure the following tools are installed:

* Node.js
* npm or pnpm
* Rust (`rustup`)
* Tauri CLI
* MySQL
* Windows PowerShell (for execution policy commands)

---

## Project Setup

### Create the Workspace

Set the required execution policy and create the Tauri project:

````
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy Unrestricted
npm create tauri-app@latest
cd `<project-directory>`
npm install
npm run dev
````
---

## Running the Application

### Start the Backend Server

node server.js

### Run the Tauri Application

````
npm run tauri dev
````

---

## Rebuild After Pull / Push

If you pull new changes or need to rebuild the project:
````
pnpm install
rustup show
pnpm tauri dev
pnpm tauri build
````
---

## Installing Required Dependencies
````
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy Unrestricted
npm create tauri-app@latest
cd `<project-directory>`
npm install
npm install --save-dev @sveltejs/adapter-static
npm install @tauri-apps/plugin-fs
npm add @tauri-apps/plugin-fs
npm run tauri add fs
npm run tauri dev
````
---

## Database Usage (MySQL)

### Basic Database Commands
````
SHOW DATABASES;
USE chat_db;
SHOW TABLES;
DESCRIBE chats;
````
### View Chat History
````
SELECT * FROM chats ORDER BY timestamp DESC;
````
---

## Tech Stack

* **Frontend:** Svelte
* **Desktop Runtime:** Tauri
* **Backend:** Node.js
* **Database:** MySQL
* **Languages:** JavaScript, TypeScript, Rust
