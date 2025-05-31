**Owler Full-Stack Application**
*Integrated Frontend (Android) & Backend (Flask) Calendar/Task Management*

This document explains how to clone, configure, and run both the Android frontend and Python/Flask backend as a complete, end-to-end application. It assumes you have two separate GitHub repositories:

* **OwlerBackend**: Flask-based REST API (Python)  
  [https://github.com/Mike0Fouad/OwlerBackend](https://github.com/Mike0Fouad/OwlerBackend)
* **OwlerFrontend**: Android (Java) client using Retrofit/Hilt/Gson to consume the backend  
  [https://github.com/Mike0Fouad/OwlerFrontend](https://github.com/Mike0Fouad/OwlerFrontend)

Below you’ll find:

1. Architecture Overview
2. Prerequisites (Common + Per-Repo)
3. Backend Setup
4. Frontend Setup
5. Running the System End-to-End
6. Environment Variables / Configuration
7. Folder Structure References
8. Acknowledgments
9. Report
–––

## 1. Architecture Overview

```
┌──────────────────────────────┐
│        Android App           │
│   (OwlerFrontend - Java)     │
│ Retrofit/Hilt/Gson/API Calls │
└─────────────┬────────────────┘
              │   JSON over HTTP(S)
              ▼
   Internet (or local network + Ngrok)
              │
              ▼
┌──────────────────────────────┐
│       Flask Backend          │
│ (OwlerBackend - Python)      │
│ MongoDB (NoSQL) & MySQL (SQL)│
└──────────────────────────────┘
```

1. **Frontend (Android)**

   * Java (MVVM) app that displays a calendar, day views, tasks, and user profile/settings.
   * Uses **Retrofit** + **Gson** to call the Flask backend’s endpoints.
   * Uses **Hilt** for dependency injection and **EncryptedSharedPreferences** for token storage.
   * Offline-first: keeps local JSON cache; WorkManager syncs when online.
   * Auth: Google Sign-In & email/password (JWT tokens from backend).

2. **Backend (Flask)**

   * Python-based REST API (Flask ★ SQLAlchemy + MongoEngine).
   * Stores user/auth data in MySQL (SQLAlchemy).
   * Stores calendar/task data + Google Fit data in MongoDB.
   * Issues JWT tokens, validates them on each request.
   * Exposes endpoints for user management (login/register/edit/delete), calendar/day/schedule/task CRUD, and fitness data retrieval via Google Fit.

3. **Data Flow**

   * When the user opens the Android app (OwlerFrontend), the LauncherActivity checks for a valid JWT.

     * If no token (or expired), it opens **AuthActivity** (email/password or Google Sign-In).
     * On successful login/registration, backend returns a JWT stored securely on device.
   * All subsequent Retrofit calls attach `Authorization: Bearer <JWT>` in headers.
   * API calls for calendar/day/task:

     1. Frontend calls RetrofitService → e.g. `GET /api/calendar/{year}/{month}`
     2. Flask verifies JWT, loads user’s MongoDB document, fetches calendar data.
     3. Returns JSON to frontend, which deserializes into Java model classes (`Calendar`, `Day`, `Schedule`, `Task`, etc.).
   * Offline-first/caching:

     * Frontend writes JSON responses into local storage (in `/data/local/…`).
     * On-demand sync: WorkManager checks connectivity → pushes local modifications (e.g. new/edited tasks) to backend.
   * Google Fit integration (Backend-side):

     * Backend holds a long-lived refresh token in MySQL for each user (obtained at Google Sign-In).
     * Periodically (or on-demand), Backend’s service uses that refresh token to fetch steps/sleep/activity from Google Fit API, then stores them under the user’s MongoDB document (`google_fit_data` field).
     * Frontend merely requests “GET /api/user/fitness” → Backend returns aggregated fitness JSON from MongoDB.

–––

## 2. Prerequisites

### 2.1 Common (Both Frontend & Backend)

* **git** (to clone repos)
* **Java 11+** (for Android Studio & Gradle)
* **Android Studio Arctic Fox (or later)** (for compiling/running the Android app)
* **Android SDK** (minimum SDK level as defined in `build.gradle`)
* A device/emulator with Android 9.0+ (API 28+)
* **Python 3.8+** (for the Flask backend)
* **pip** (Python package installer)
* **MySQL Server** (8.x recommended)
* **MongoDB Server** (4.4+ recommended)
* **Ngrok** (optional, to expose local Flask to internet for testing)

### 2.2 Backend-Specific

* Navigate to `OwlerBackend/`
* **Virtual Environment** (venv) recommended:

  ```bash
  cd OwlerBackend
  python3 -m venv venv
  source venv/bin/activate    # (macOS/Linux)
  venv\Scripts\activate       # (Windows)
  ```
* Create a `.env` file (see “Env Variables” below).
* Install Python dependencies:

  ```bash
  pip install -r requirements.txt
  ```
* Ensure **MySQL** is running on your machine (or remote).
* Ensure **MongoDB** is running (local or remote).

### 2.3 Frontend-Specific

* Navigate to `OwlerFrontend/`
* Android Studio will automatically download/configure Gradle wrappers.
* In `app/build.gradle`, verify that you have at least:

  ```groovy
  compileSdkVersion 33
  defaultConfig {
      minSdkVersion 21
      targetSdkVersion 33
      ...
  }
  ```
* Ensure the `BASE_URL` in the Retrofit configuration (`RetrofitService.java` or `BuildConfig.java`) points to your Flask server’s URL (e.g. `http://localhost:5000/` or your Ngrok HTTPS URL).

–––

## 3. Backend Setup (OwlerBackend)

1. **Clone the repository**

   ```bash
   git clone https://github.com/Mike0Fouad/OwlerBackend.git
   cd OwlerBackend
   ```

2. **Set up a virtual environment**

   ```bash
   python3 -m venv venv
   source venv/bin/activate     # macOS/Linux
   # (or venv\Scripts\activate on Windows)
   ```

3. **Install dependencies**

   ```bash
   pip install -r requirements.txt
   ```

4. **Create/Configure `.env`**
   In the root of `OwlerBackend/`, create a file named `.env`. Copy the keys below, replacing placeholder values:

   ```dotenv
   FLASK_APP=run.py
   FLASK_ENV=development

   # MySQL (user auth & JWT storage)
   SQLALCHEMY_DATABASE_URI=mysql+pymysql://<MYSQL_USER>:<MYSQL_PASSWORD>@localhost:3306/<MYSQL_DB_NAME>

   # MongoDB (calendar & fitness data)
   MONGO_URI=mongodb://localhost:27017/<MONGODB_DB_NAME>

   # JWT secret key for signing tokens
   JWT_SECRET_KEY=your_jwt_secret_key_here

   # Optional: Google OAuth2 credentials for Google Sign-In / Google Fit
   GOOGLE_CLIENT_ID=your_google_client_id.apps.googleusercontent.com
   GOOGLE_CLIENT_SECRET=your_google_client_secret
   ```

5. **Initialize Databases**

   * **MySQL**:

     1. Log into MySQL: `mysql -u root -p`
     2. Create database: `CREATE DATABASE your_db_name CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;`
     3. (If needed) Create a user, grant privileges:

     ```sql
     CREATE USER 'owler_user'@'localhost' IDENTIFIED BY 'owler_password';
     GRANT ALL PRIVILEGES ON your_db_name.* TO 'owler_user'@'localhost';
     FLUSH PRIVILEGES;
     ```
   * **MongoDB**:

     1. Start MongoDB (e.g. `sudo systemctl start mongod`)
     2. By default, the Flask code will auto-create collections. You just need to ensure the `MONGO_URI` points to an existing database.
     3. To verify:

     ```bash
     mongo
     > show dbs
      # If <MONGODB_DB_NAME> is not listed, it will be created at first write.
     ```

6. **Run Database Migrations (if applicable)**

   > *Note: The provided backend does not use Flask-Migrate by default. If you already have `models.py` and want to create tables manually, run Python shell and `db.create_all()`. Otherwise, the tables will be auto-created when you attempt to use them.*

7. **Run the Flask App**

   ```bash
   # Activate environment if not yet active:
   source venv/bin/activate

   # Start Flask
   flask run
   ```

   By default, this serves on `http://127.0.0.1:5000/`.
   If you want to expose it to the internet (for device testing), in a separate terminal run:

   ```bash
   ngrok http 5000
   ```

   Then copy the HTTPS Ngrok URL (e.g. `https://abcd1234.ngrok.io`) and use that in the Android app as `BASE_URL`.

8. **Verify Endpoints**
   Open a browser or Postman to verify a simple ping endpoint (if one exists, e.g. `GET /healthcheck` or `GET /api/user/profile` with a valid token).

–––

## 4. Frontend Setup (OwlerFrontend)

1. **Clone the repository**

   ```bash
   git clone https://github.com/Mike0Fouad/OwlerFrontend.git
   cd OwlerFrontend
   ```

2. **Open in Android Studio**

   * Launch Android Studio → “Open an existing Android Studio project.”
   * Navigate to the `OwlerFrontend/` folder.
   * Wait for Gradle to sync.

3. **Configure Dependencies**

   * In `app/build.gradle`, verify the following dependencies are present (they should match the template below or similar versions):

     ```groovy
     implementation 'com.squareup.retrofit2:retrofit:2.9.0'
     implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
     implementation 'com.google.dagger:hilt-android:2.44'
     kapt 'com.google.dagger:hilt-compiler:2.44'
     implementation 'androidx.security:security-crypto:1.1.0-alpha03'  // EncryptedSharedPreferences
     implementation 'com.google.android.gms:play-services-auth:20.5.0' // Google Sign-In
     implementation 'androidx.work:work-runtime:2.8.1'  // WorkManager
     implementation 'com.jakewharton.timber:timber:5.0.1'
     // ... plus other AndroidX core/compat libraries, lifecycle components, etc.
     ```
   * Sync Gradle if not already done.

4. **Set `BASE_URL`**
   In `app/src/main/java/com/owlerdev/owler/network/RetrofitService.java` (or in `BuildConfig.java` if your project centralizes it), find something like:

   ```java
   public static final String BASE_URL = "http://YOUR_BACKEND_URL/";
   ```

   Replace `"http://YOUR_BACKEND_URL/"` with:

   * `http://10.0.2.2:5000/` (if running the Flask server locally and testing on the Android emulator), **OR**
   * Your Ngrok HTTPS link (e.g. `https://abcd1234.ngrok.io/`) if testing on a physical device.

5. **Permissions & Manifest Declarations**
   Ensure `AndroidManifest.xml` includes:

   ```xml
   <uses-permission android:name="android.permission.INTERNET" />
   <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
   <!-- If using location or fitness features later, declare them here -->
   ```

6. **Run the App**

   * Connect your Android device (USB debug) or start an emulator.
   * Click “Run” in Android Studio → select your target.
   * The LauncherActivity will check for a stored JWT.

     * If you haven’t logged in yet, it will show **AuthActivity** (Login/Register).
     * If you want to skip backend login for testing UI only, temporarily comment out the token check in `LauncherActivity.java` to always open `MainActivity`.

7. **Testing the Flow**

   1. **Register a New User** (on the Android app) → backend will create MySQL user record + empty calendar entry in MongoDB.
   2. **Log In** → backend returns JWT → Android securely stores it.
   3. **MainActivity/CalendarView** → Retrofit fetches calendar for current month.
   4. **DayView** → Retrofit fetches tasks for that day from `/api/day/{userId}/{date}`.
   5. **Add/Edit Task** → POST/PUT to `/api/task/` → backend updates MongoDB.
   6. **Sync** → WorkManager periodically checks local edits and pushes them to backend if online.

–––

## 5. Running the System End-to-End

1. **Launch Backend**

   ```bash
   cd OwlerBackend
   source venv/bin/activate
   flask run
   # (Optionally run `ngrok http 5000` and copy the forwarded URL.)
   ```

2. **Launch Frontend**

   * Open `OwlerFrontend` in Android Studio
   * Make sure `BASE_URL` in Retrofit points to your Flask server (localhost vs. ngrok).
   * Run on emulator or device.

3. **User Flow**

   * **AuthActivity** (Login/Register).
   * **CalendarActivity** (Monthly grid).

     * Click any day to open **DayFragment**, which shows tasks.
     * Click “Add Task” to open **TaskCreationFragment**.
   * **ProfileView / AccountSettingsView** to manage user details (calls `/api/user/edit`, `/api/user/delete`).
   * **Sync / Offline**:

     * If offline, user can still add tasks locally; they’re kept in JSON under `data/local/…`.
     * Once connectivity returns, WorkManager triggers `SyncWorker.java` to push local changes via Retrofit.

4. **Common Pitfalls**

   * If Retrofit calls return `401 Unauthorized`: verify the JWT is present in `Authorization` header.
   * If `BASE_URL` is incorrect (e.g. using `localhost` on a physical device), use the Ngrok domain or emulator’s special `10.0.2.2`.
   * Database not found errors: ensure MySQL/MongoDB are running and the URIs in `.env` are correct.

–––

## 6. Environment Variables / Configuration Details

### 6.1 Backend `.env` Keys

```dotenv
# Flask configuration
FLASK_APP=run.py
FLASK_ENV=development

# SQLAlchemy (MySQL)
SQLALCHEMY_DATABASE_URI=mysql+pymysql://<DB_USER>:<DB_PASSWORD>@<HOST>:3306/<DB_NAME>

# MongoDB (NoSQL)
MONGO_URI=mongodb://<HOST>:27017/<MONGO_DB_NAME>

# JWT secret for signing tokens
JWT_SECRET_KEY=<YOUR_SECRET_KEY>

# (Optional) Google API credentials for Google Sign-In & Google Fit
GOOGLE_CLIENT_ID=<your-google-client-id>
GOOGLE_CLIENT_SECRET=<your-google-client-secret>
```

* **SQLALCHEMY\_DATABASE\_URI** should point to a running MySQL instance.
* **MONGO\_URI** should point to a running MongoDB instance.
* **JWT\_SECRET\_KEY** must be a strong, random string (e.g. use `openssl rand -base64 32`).
* If you plan to enable Google Sign-In and Google Fit: add valid OAuth2 credentials as `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET`.

> **Tip**: If you modify `.env`, restart the Flask server to pick up changes.

### 6.2 Frontend `BASE_URL`

* In `RetrofitService.java` (or wherever `BASE_URL` is defined), set:

  ```java
  public static final String BASE_URL = "http://10.0.2.2:5000/"; 
  // or 
  // public static final String BASE_URL = "https://<your-ngrok-id>.ngrok.io/";
  ```

* If you later move your Flask app to a production server (`https://api.myowlerapp.com/`), update this constant and rebuild the APK.

–––

## 7. Folder Structure References

Below is a high-level snapshot of each repository’s folder structure. This can help you confirm you’ve cloned the correct repo and are in the right directories.

### 7.1 Backend (`OwlerBackend/`)

```
OwlerBackend/
├── app/
│   ├── __init__.py        # Flask app factory, extensions (SQLAlchemy, PyMongo, JWT)
│   ├── models.py          # SQLAlchemy user & token models, MongoEngine calendar models
│   ├── routes/
│   │   ├── auth.py        # /api/auth (login/register) endpoints
│   │   ├── user.py        # /api/user (profile/edit/delete)
│   │   ├── calendar.py    # /api/calendar (month view) endpoint
│   │   ├── day.py         # /api/day (daily tasks) endpoint
│   │   ├── task.py        # /api/task (CRUD tasks) endpoint
│   │   ├── fitness.py     # /api/fitness (Google Fit data) endpoint
│   ├── services/          # Business logic, Google Fit service, ML predictor (if any)
│   └── utils.py           # Helper functions (token utils, date parsing)
├── migrations/            # (if using Flask-Migrate; might be empty)
├── requirements.txt       # Flask, Flask-PyMongo, PyJWT, SQLAlchemy, PyMySQL, etc.
├── run.py                 # Entry point to start Flask (calls app.factory)
├── .env.example           # Template environment variables
└── README.md              # Backend-only readme (you’ll replace/integrate this into the combined doc)
```

### 7.2 Frontend (`OwlerFrontend/`)

```
OwlerFrontend/
└── app/
    ├── src/
    │   ├── main/
    │   │   ├── java/com/owlerdev/owler/
    │   │   │   ├── adapter/             # RecyclerView adapters (TaskAdapter, CalendarAdapter, etc.)
    │   │   │   ├── data/
    │   │   │   │   ├── local/            # JSON storage & EncryptedSharedPreferences utils
    │   │   │   │   ├── remote/           # Retrofit client, services, API interfaces
    │   │   │   │   └── repository/       # Repository classes that expose data source abstractions
    │   │   │   ├── di/                  # Hilt modules (NetworkModule, StorageModule, etc.)
    │   │   │   ├── model/               # Java model classes matching backend (User, Calendar, Day, Task, GoogleFitData, etc.)
    │   │   │   ├── network/             # Interceptors (AuthInterceptor), RetrofitService builder, connectivity utils
    │   │   │   ├── service/             # Business logic & helper classes (SyncWorker, TokenManager)
    │   │   │   ├── ui/
    │   │   │   │   ├── activity/        # LauncherActivity, AuthActivity, CalendarActivity, MainActivity
    │   │   │   │   ├── fragment/        # CalendarFragment, DayFragment, TaskFragment, ProfileFragment, AccountSettingsFragment
    │   │   │   ├── utils/               # DateUtils, NotificationUtils, NetworkUtils, Constants
    │   │   │   ├── viewmodel/           # ViewModel classes per feature (AuthViewModel, CalendarViewModel, etc.)
    │   │   │   └── storage/             # EncryptedSharedPreferences wrapper (TokenManager)
    │   │   ├── res/
    │   │   │   ├── layout/              # XML layouts (activity_auth.xml, fragment_calendar.xml, etc.)
    │   │   │   ├── drawable/  
    │   │   │   └── values/              # Colors, strings, styles
    │   │   └── AndroidManifest.xml
    │   └── test/  (unit tests, optional)
    ├── build.gradle
    └── settings.gradle
```

–––

## 8. Acknowledgments

* **Flask & Python**: For an easy-to-use web framework and ecosystem.
* **MySQL** & **MongoDB**: Robust, well-documented datastores for user/auth security and flexible calendar/task data.
* **Retrofit** & **Gson**: Simplifies HTTP/JSON interactions on Android.
* **Hilt** (Dagger) & **EncryptedSharedPreferences** & **WorkManager**: Best practices for DI, secure data storage, and background syncing.
* **Timber**: Simplified logging on Android.
* **Ngrok**: Expose local Flask server to the internet for mobile testing.

Thank you to all the open-source communities whose libraries and tools made this full-stack app possible!

---

## 9. Report
This report provides a comprehensive overview of the Owler full-stack application, detailing the architecture, setup, and usage of both the Android frontend and Flask backend. It serves as a guide for developers to understand how to clone, configure, and run the application end-to-end.

[Owler Report PDF](https://drive.google.com/file/d/1eRGHvrJ2XVWehWsnHKHFeMANCeRXO85m/view?usp=sharing)

---

**End of README**

