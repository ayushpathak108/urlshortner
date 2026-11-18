# 🔗 Express.js & MongoDB URL Shortener

A lightweight, high-performance, custom URL shortener web application built with **Node.js**, **Express 5**, and native **MongoDB**. It allows users to create personalized, easy-to-remember short links for lengthy URLs and provides redirection with duplicate slug protection and modern dark-mode UI.

---

## 📑 Table of Contents

- [The Problem](#-the-problem)
- [The Solution](#-the-solution)
- [System Architecture](#-system-architecture)
- [Application Workflow](#-application-workflow)
- [Project Directory Structure](#-project-directory-structure)
- [Tech Stack](#-tech-stack)
- [Data Model & Storage](#-data-model--storage)
- [API & Routing Reference](#-api--routing-reference)
- [Setup & Installation Guide](#-setup--installation-guide)
  - [Prerequisites](#prerequisites)
  - [Step-by-Step Installation](#step-by-step-installation)
  - [Environment Variables](#environment-variables)
  - [Running the Application](#running-the-application)
- [Client-Side Validation & UI](#-client-side-validation--ui)

---

## 🛑 The Problem

Long URLs pose several practical challenges across digital communication:

1. **Cluttered Sharing**: URLs with query parameters, session tokens, and deep routing paths look unappealing and messy when shared on social media, email, messaging apps, or print media.
2. **Prone to Breakage**: Excessively long links often get accidentally truncated or line-wrapped in chat clients and SMS, resulting in broken links and 404 errors.
3. **Lack of Personalization**: Most public shorteners generate random alphanumeric hashes (e.g., `bit.ly/3xY7k9q`) that give users no context about the destination and cannot be customized without paid plans.
4. **Overcomplicated Implementations**: Many developers default to complex ORMs or heavy full-stack frameworks for simple URL shortening, introducing unnecessary latency and maintenance overhead.

---

## 💡 The Solution

This project provides a clean, self-hosted, full-stack solution:

- **Custom Slug Creation**: Users choose their own meaningful short aliases (e.g., `po`, `yu`, `my-portfolio`) rather than forced random characters.
- **Fast Native MongoDB Operations**: Direct usage of the official Node.js `mongodb` driver (`MongoClient`) without the overhead of heavy ODM layers like Mongoose.
- **Duplicate Protection**: Validates requested aliases prior to insertion and returns a `400 Bad Request` if a custom shortcode is already taken.
- **Immediate Confirmation**: Upon submission, the app dynamically renders the newly minted short link with the host domain and target preview, ready to be copied or tested.
- **Zero-Dependency Templating**: Lightweight string interpolation directly into clean static HTML, serving fast pages without heavy template engines.
- **Modern Responsive UI**: Built-in dark glassmorphic interface with cyan accent styling, mobile responsiveness, and client-side form validation.

---

## 🏗 System Architecture

The application adopts the **Model-View-Controller (MVC)** architectural pattern:

- **View Layer**: Static HTML (`views/index.html`) styled with CSS (`pub/style.css`), served via Express static middleware.
- **Controller Layer**: Handles incoming HTTP requests, performs validations, interacts with the data layer, and renders responses.
- **Model Layer**: Interacts with the MongoDB `shorterner` collection using native driver queries (`find`, `findOne`, `insertOne`).
- **Configuration**: Loads environment configurations using `dotenv` and maintains a single `MongoClient` instance.

### Architecture Diagram

```mermaid
graph TD
    User["👤 Client / Web Browser"]

    subgraph Server["🚀 Express.js Web Server (anu02.js)"]
        Static["Static Middleware (pub/style.css)"]
        Parser["Body Parser (express.urlencoded)"]
        Router["Express Router (routes/url.routes.js)"]

        subgraph Controllers["🎮 Controllers"]
            C_Get["gettinData.controller.js<br/>GET /"]
            C_Save["savingtheDetails.controller.js<br/>POST /"]
            C_Parcel["puttingTheLinkOnPage.controller.js<br/>GET /parcel/:shorten"]
            C_Redirect["redirecting.controllers.js<br/>GET /:shorten"]
        end

        subgraph ViewEngine["📄 View & Template Processing"]
            HTML["views/index.html<br/>(replaces {{shoten-code}})"]
        end

        subgraph ModelLayer["📦 Model Layer (models/savers.model.js)"]
            M_Load["loadlinks()"]
            M_Save["save(link)"]
            M_Get["getlinkByShortcode(shorten)"]
        end

        subgraph DBConfig["⚙️ Configuration (config/mongodb.js)"]
            MongoClient["MongoClient (process.env.mongo_url)"]
        end
    end

    subgraph Database["🍃 MongoDB Database"]
        Collection[("Collection: shorterner<br/>{ url, finalShort }")]
    end

    User -->|"HTTP Requests"| Server
    Server --> Parser
    Server --> Static
    Parser --> Router

    Router -->|"GET /"| C_Get
    Router -->|"POST /"| C_Save
    Router -->|"GET /parcel/:shorten"| C_Parcel
    Router -->|"GET /:shorten"| C_Redirect

    C_Get --> HTML
    C_Save --> M_Load
    C_Save --> M_Save
    C_Parcel --> M_Load
    C_Parcel --> HTML
    C_Redirect --> M_Get

    M_Load --> Collection
    M_Save --> Collection
    M_Get --> Collection
    DBConfig --> Database
    C_Redirect -->|"HTTP 302 Redirect"| User
```

---

## 🔄 Application Workflow

The application handles two primary workflows: **Short Link Creation** and **Short Link Redirection**.

### 1. Short Link Creation & Display Workflow

```mermaid
sequenceDiagram
    autonumber
    actor User as User / Browser
    participant Router as url.routes.js
    participant Controller as savingtheDetails.controller.js
    participant Model as savers.model.js
    participant DB as MongoDB (shorterner)
    participant Display as puttingTheLinkOnPage.controller.js
    participant View as views/index.html

    User->>Router: POST / with body { url, shorten }
    Router->>Controller: savingthedetail(req, res)
    Controller->>Model: loadlinks()
    Model->>DB: collect.find().toArray()
    DB-->>Model: Return all existing documents
    Model-->>Controller: [ { url, finalShort }, ... ]

    alt Shortcode Already Exists
        Controller-->>User: 400 "shorten name already takem"
    else Shortcode Available
        Controller->>Model: save({ url, finalShort })
        Model->>DB: collect.insertOne({ url, finalShort })
        DB-->>Model: Acknowledged
        Controller-->>User: HTTP 302 Redirect to /parcel/:finalShort
        User->>Router: GET /parcel/:finalShort
        Router->>Display: showuptheLINK(req, res)
        Display->>Model: loadlinks()
        Model->>DB: collect.find().toArray()
        DB-->>Model: Return documents
        Display->>View: Read index.html & replace {{shoten-code}} with <li> link
        Display-->>User: Render updated page with clickable short URL
    end
```

### 2. URL Redirection Workflow

```mermaid
sequenceDiagram
    autonumber
    actor User as User / Browser
    participant Router as url.routes.js
    participant Redirector as redirecting.controllers.js
    participant Model as savers.model.js
    participant DB as MongoDB (shorterner)

    User->>Router: GET /:shorten (e.g. GET /po)
    Router->>Redirector: redirect(req, res)
    Redirector->>Model: getlinkByShortcode(shorten)
    Model->>DB: collect.findOne({ finalShort: shorten })
    DB-->>Model: Matched document OR null

    alt Shortcode Not Found
        Redirector-->>User: 200 "not found"
    else Shortcode Found
        Redirector-->>User: HTTP 302 Redirect to link.url
        Note over User: Browser immediately navigates to original target URL
    end
```

---

## 📁 Project Directory Structure

```text
UrlSHortener2-with-express-/
├── config/
│   └── mongodb.js                 # MongoDB client connection setup
├── controllers/
│   ├── gettinData.controller.js           # Renders the initial landing page (GET /)
│   ├── puttingTheLinkOnPage.controller.js # Renders landing page with generated link (GET /parcel/:shorten)
│   ├── redirecting.controllers.js         # Handles short URL lookup & HTTP 302 redirect (GET /:shorten)
│   └── savingtheDetails.controller.js     # Validates uniqueness & saves URL pair (POST /)
├── models/
│   └── savers.model.js            # MongoDB database queries (loadlinks, save, getlinkByShortcode)
├── pub/
│   └── style.css                  # Modern dark glassmorphic CSS styling
├── routes/
│   └── url.routes.js              # Express route definitions mapping URLs to controllers
├── views/
│   └── index.html                 # Main frontend template with placeholder {{shoten-code}}
├── .env                           # Local environment variables (ignored by git)
├── .env.example                   # Sample environment variable template
├── .gitignore                     # Git ignore rules (.env, node_modules, package-lock.json)
├── anu02.js                       # Express server entry point & middleware configuration
├── links.json                     # Legacy JSON data store (from initial prototype before MongoDB)
└── package.json                   # Project manifest, dependencies, and npm scripts
```

---

## 🛠 Tech Stack

- **Runtime**: [Node.js](https://nodejs.org/) (ES Modules, `"type": "module"`)
- **Web Framework**: [Express.js 5](https://expressjs.com/) (`^5.1.0`)
- **Database Driver**: [MongoDB Node.js Driver](https://www.mongodb.com/docs/drivers/node/current/) (`^6.19.0`)
- **Configuration**: [dotenv](https://github.com/motdotla/dotenv) (`^17.2.2`)
- **Frontend**: Vanilla HTML5, modern CSS3 (Custom properties, grid layout, glassmorphism), and vanilla JavaScript.

---

## 🗄 Data Model & Storage

The application connects to MongoDB using the native driver without an ORM.

- **Database**: Configured via `mongodb_name` in `.env`.
- **Collection**: `shorterner`

### Document Schema

```json
{
  "_id": { "$oid": "66487e49f872b7..." },
  "url": "https://www.facebook.com/",
  "finalShort": "po"
}
```

| Field Name | Type | Description |
| :--- | :--- | :--- |
| `_id` | `ObjectId` | Auto-generated unique MongoDB identifier. |
| `url` | `String` | Original long destination URL. |
| `finalShort` | `String` | Custom shortcode/slug entered by the user. |

> **Note on Migration**: The project originally used `links.json` for file-based persistence during early prototyping. The codebase has since migrated to MongoDB in `models/savers.model.js`.

---

## 📡 API & Routing Reference

All routes are defined in [`routes/url.routes.js`](file:///d:/project-new%20Strart/UrlSHortener2-with-express-/routes/url.routes.js):

| HTTP Method | Route Endpoint | Controller Handler | Purpose / Description |
| :--- | :--- | :--- | :--- |
| **GET** | `/` | `gettinData` | Renders the home page with the input form and clears the `{{shoten-code}}` placeholder. |
| **POST** | `/` | `savingthedetail` | Receives form data (`url`, `shorten`), checks if slug already exists, saves to MongoDB, and redirects to `/parcel/:shorten`. Returns `400` if taken. |
| **GET** | `/parcel/:shorten` | `showuptheLINK` | Renders the home page displaying the newly created short link under the `ShortenURL` section. |
| **GET** | `/:shorten` | `redirect` | Looks up `shorten` slug in MongoDB. If found, issues an HTTP 302 redirect to original `url`. Returns `"not found"` if absent. |

---

## 🚀 Setup & Installation Guide

### Prerequisites

Ensure you have the following installed on your machine:
- **Node.js** (v18.0.0 or higher recommended)
- **npm** (bundled with Node.js)
- A **MongoDB** database instance (either local MongoDB server or a [MongoDB Atlas](https://www.mongodb.com/cloud/atlas) cloud cluster)

---

### Step-by-Step Installation

1. **Clone the repository / navigate to project root:**
   ```bash
   cd "d:/project-new Strart/UrlSHortener2-with-express-"
   ```

2. **Install project dependencies:**
   ```bash
   npm install
   ```

3. **Configure Environment Variables:**
   Copy the `.env.example` file to create your own `.env`:
   ```bash
   cp .env.example .env
   ```
   *(On Windows Command Prompt, use `copy .env.example .env`)*

4. **Update `.env` values:**
   Open `.env` and fill in your connection details:
   ```env
   mongo_url=mongodb+srv://<username>:<password>@cluster0.mongodb.net/
   mongodb_name=urlshortener
   PORT=3000
   ```

---

### Environment Variables

| Variable Name | Required | Default / Example | Purpose |
| :--- | :--- | :--- | :--- |
| `mongo_url` | **Yes** | `mongodb+srv://user:pass@cluster...` | MongoDB connection URI string. |
| `mongodb_name`| **Yes** | `urlshortener` | Target database name in MongoDB. |
| `PORT` | No | `3000` | Port on which Express server listens (falls back to `pro` or `3000`). |
| `pro` | No | `3000` | Alternative port environment variable accepted by `anu02.js`. |

---

### Running the Application

- **Start the server:**
  ```bash
  npm start
  ```
  *(Or `npm run dev`, which runs `node anu02.js`)*

- **Verify in console:**
  ```text
  running on 3000 port
  ```

- **Open the web application:**
  Visit `http://localhost:3000` in your web browser.

---

## 🖥 Client-Side Validation & UI

- **Form Validation**: If a user attempts to submit the form without providing a custom shortcode, an alert displays `"Enter a short code, dammit!"` and halts form submission via `e.preventDefault()`.
- **CSS Styling (`pub/style.css`)**:
  - Dark mode color palette using CSS variables (`--bg: #0f172a`, `--panel: #111827`, `--accent: #22d3ee`).
  - Radial gradient backgrounds and backdrop blur.
  - Responsive grid layout adjusting down to mobile devices (`max-width: 520px`).
  - Active button and input focus states with cyan glow ring effect.
