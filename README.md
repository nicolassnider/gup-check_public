# GupCheck

**The consensus of black belts, the decision of the Sabum-Nim.** 🥋

![GupCheck Cover](public/gupcheck-cover.png)

GupCheck is a web application designed specifically for ITF Taekwondo schools that digitalizes and centralizes pre-evaluations for Gup (colored belt) students.

## The Problem

Every time exam dates approach, evaluating dozens of colored belts becomes a communication chaos. Black belts take notes on papers that get lost, or send scattered WhatsApp messages with corrections about a student's Tul or sparring. In the end, the Sabum-Nim (Master Instructor) has to make decisions about who can test based on fragmented comments or their own memory, overloading themselves with work in the weeks leading up to the exam.

## The Solution

GupCheck allows any black belt to select a student from their phone, right in the Dojang, and leave quick, technical, and structured feedback on their Tuls (forms), Matsogi (sparring), and Theory. All this information is consolidated in a dashboard exclusive to the Head Instructor.

## Benefits

- **Data-driven decisions**: The Head Instructor no longer decides alone; they see the real consensus and technical comments from the entire Dan (black belt) body on a single screen.
- **Time optimization**: Fewer meetings to "see who's ready" and zero lost papers.
- **Personalized tracking**: Allows detecting exactly what each student is missing (e.g., "Facundo needs to remember the meaning of his belt color") to adjust it in the final classes before the exam.
- **School identity**: Elevates the institution's professionalism in the eyes of students and parents.

## Future Vision

GupCheck is just the first step. Built on a modular architecture designed to become the school's digital ecosystem, it will allow future integration of attendance modules, membership management, and tournament tracking.

## Business Rules

For detailed information about the system's business rules, permissions, and workflows, please refer to [BUSINESS_RULES.md](./BUSINESS_RULES.md). This document covers:

- User roles and permissions (Instructors can create exams and students)
- Evaluation rules and data integrity
- Final decision requirements (80% threshold)
- Exam management rules
- UI/UX guidelines for different user types
- Technical architecture details

## Architecture

### Firebase Project

GupCheck uses a single Firebase project (**fenix-gup-check**) for all data storage and authentication:

- **Firebase Authentication**: Handles user authentication with email/password
- **Firestore**: Stores all application data including users, students, exams, evaluations, and audit logs
- **Custom Claims**: User roles (instructor, assistant_instructor, admin) are stored as Firebase Auth custom claims for efficient authorization

### Firebase Admin SDK Setup

The project includes scripts for creating users and students that use Firebase Admin SDK. To use these scripts, you need to set up service account credentials:

**Steps to obtain private keys:**

1. Go to Firebase Console: <https://console.firebase.google.com/>
2. Select the fenix-gup-check project
3. Navigate to Project Settings (gear icon)
4. Go to the "Service accounts" tab
5. Click "Generate new private key"
6. Download the JSON file
7. Copy the values to your `.env` file:
   - `FIREBASE_DATA_PROJECT_ID` (from `project_id`)
   - `FIREBASE_DATA_PRIVATE_KEY` (from `private_key` - keep the \n characters)
   - `FIREBASE_DATA_CLIENT_EMAIL` (from `client_email`)

**Important:** Never commit your `.env` file or private keys to version control. The `.env.example` file contains placeholder values for reference only.

### Firebase Security Rules

The project includes comprehensive security rules for Firestore:

- **firestore-data.rules** - Rules for the unified Firebase project (fenix-gup-check)

Security rules enforce:

- Role-based access control using Firebase Auth custom claims
- Immutability for evaluations, final decisions, and promotions
- Exam status validation (closed exams are immutable)
- Audit log restricted to server-side writes only

Deploy rules using PowerShell script (recommended):

```powershell
.\deploy-data-rules.ps1
```

Or manually:

```powershell
Copy-Item firestore-data.rules firestore.rules
firebase deploy -P fenix-gup-check --only firestore:rules
```

Note: Firebase CLI expects the rules file to be named `firestore.rules`, so the script copies the appropriate rule file before deployment.

### Database Architecture

#### fenix-gup-check (Unified Project)

**Collections:**

- `users` - User profiles and metadata
  - `id` (string) - User ID
  - `name` (string) - Full name
  - `email` (string) - Email address
  - `rank` (string) - Dan rank (e.g., "IV Dan")
  - `role` (string) - Role: "instructor", "assistant_instructor", or "admin"
  - `createdAt` (timestamp) - Account creation date
  - `updatedAt` (timestamp) - Last update date

- `exams` - Exam events and scheduling
  - `id` (string) - Exam ID
  - `title` (string) - Exam title
  - `examDate` (timestamp) - Official exam date
  - `simulationDate` (timestamp) - Simulation date
  - `status` (string) - Status (e.g., "scheduled", "completed")
  - `evaluators` (array) - List of evaluator IDs
  - `students` (array) - List of registered students
  - `createdAt` (timestamp) - Creation date
  - `updatedAt` (timestamp) - Last update date

- `evaluations` - Evaluation feedback from evaluators
  - `id` (string) - Evaluation ID
  - `examId` (string) - Reference to exam
  - `evaluatorId` (string) - Reference to evaluator
  - `evaluatorName` (string) - Evaluator name
  - `studentId` (string) - Reference to student
  - `studentName` (string) - Student name
  - `currentBelt` (string) - Current belt rank
  - `targetBelt` (string) - Target belt rank
  - `date` (timestamp) - Evaluation date
  - `isReady` (boolean) - Whether student is ready for exam
  - `categories` (object) - Evaluation by category
    - `tul` (object) - Forms evaluation
      - `score` (number) - Score 1-10
      - `comments` (string) - Feedback
      - `specificTul` (string) - Specific pattern name
    - `matsogi` (object) - Sparring evaluation
      - `score` (number) - Score 1-10
      - `comments` (string) - Feedback
      - `specificTechnique` (string) - Specific technique
    - `theory` (object) - Theory evaluation
      - `score` (number) - Score 1-10
      - `comments` (string) - Feedback
      - `topics` (array) - Topics covered
  - `overallNotes` (string) - General feedback
  - `recommendations` (array) - Specific recommendations
  - `submittedAt` (timestamp) - Submission timestamp

- `students` - Student information
  - `id` (string) - Student ID
  - `name` (string) - Full name
  - `currentBelt` (string) - Current belt rank (from static belt ranks)
  - `targetBelt` (string) - Target belt rank for next exam (from static belt ranks)
  - `dateOfBirth` (timestamp) - Date of birth
  - `joinDate` (timestamp) - Date joined the school
  - `notes` (string) - Additional notes
  - `createdAt` (timestamp) - Record creation date
  - `updatedAt` (timestamp) - Last update date

**Static Data:**

- `belt-ranks` - Belt rank definitions (static, not stored in database)
  - Defined in `src/data/belt-ranks.json`
  - Includes progression from blanco to rojo punta negra
  - Supports both solid colors and color-with-tip variations
  - Used for validation and UI display

## Workflow

### Exam Simulation Process

1. **Schedule Exam**: The Sabum-Nim schedules an official exam date (e.g., June 13th)
2. **Schedule Simulation**: Approximately 2 weeks before the exam, a simulation of the exam table is conducted (e.g., May 30th)
3. **Evaluation**: During the simulation, each black belt evaluator:
   - Selects a student from their mobile device
   - Observes the student's performance in Tuls (forms), Matsogi (sparring), and Theory
   - Records structured feedback and notes
   - Indicates whether the student is ready to test (apt/no apt)
4. **Consolidation**: All evaluations are automatically consolidated in the Head Instructor's dashboard
5. **Decision**: The Sabum-Nim reviews the consensus and technical feedback from all evaluators to make informed decisions about which students are ready for the official exam

## Use Cases

### 1. Authentication and Login

- Evaluators select their user to access the system
- Roles: Instructor (Sabum-Nim), Assistant Instructor (Busabum), Admin

### 2. Main Dashboard

- View general statistics
- List of students with evaluation summary
- Navigation to other sections

### 3. Student Management

- Create new student
- Edit student information (name, current/target belt, join date, notes)
- Delete student (logical deletion - student is marked as deleted but can be restored)
- Restore deleted student (only admins and instructors can restore)
- View student list with categories (Child, Youth, Adult)

### 4. Exam Table Management

- Create new exam table
- Assign students to the table (system defaults to student's targetBelt, or use exam target belt as filter)
- Assign evaluators to the table
- Define target belt for the exam (acts as filter, e.g., "Exam table for green belts")

### 5. Exam Table List

- View all created exam tables
- Filter by status (scheduled, active, completed, cancelled, closed)
- View number of assigned candidates

### 6. Student Evaluation

- Evaluators evaluate assigned students in a table
- Grade specific techniques (forms, sparring, etc.)
- Indicate if student is ready to pass
- Only assigned evaluators can evaluate

### 7. View Student Evaluations

- Sabum-Nim can view all evaluations of a student
- View score averages
- View approval recommendation based on evaluators

### 8. Final Evaluation

- Sabum-Nim performs final student evaluation
- Requires minimum 80% of completed evaluations
- Final approval/rejection decision (saved in final_decisions collection)
- Only available when evaluation requirement is met
- Exam status changes to completed as Sabum-Nim evaluates students one by one
- Final decision does NOT automatically close the exam

### 9. Role and Permission Management

- Instructors can create/edit students and exam tables
- Assistant Instructors can evaluate students but cannot create exam tables (button is hidden)
- Admin has all permissions
- Access control by role in each functionality

### 10. Close Exam Table

- Manual action to close an exam table
- Only available when all students have final decisions
- Changes exam status to closed
- Enables write lock on API (prevents further modifications)
- Separated from final evaluation to maintain explicit workflow

### 11. Official Exam Graduation (Post-Simulation)

- Triggered after an exam table is closed and the official exam date is reached
- Sabum-Nim records official exam results only for students marked as "Apto" during simulation
- Allows marking students as "Aprobó" (moves to target belt), "Sobresaliente" (double promotion to a selected belt), or "Desaprobó/Ausente" (keeps current belt)
- Bulk action automatically updates student profiles (`currentBelt` and future `targetBelt`)
- Creates an immutable historical record of the promotion and archives the exam

### 12. Audit Logs

- Instructors and Admins can view a comprehensive audit trail of all system actions
- Tracks evaluations, final decisions, belt changes, student modifications, and more
- Filterable by event type and date range
- Provides transparency and accountability for critical operations

## 🚀 Project Structure

Inside of your Astro project, you'll see the following folders and files:

```text
/
├── public/
├── src/
│   └── pages/
│       └── index.astro
└── package.json
```

Astro looks for `.astro` or `.md` files in the `src/pages/` directory. Each page is exposed as a route based on its file name.

There's nothing special about `src/components/`, but that's where we like to put any Astro/React/Vue/Svelte/Preact components.

Any static assets, like images, can be placed in the `public/` directory.

## 🧞 Commands

All commands are run from the root of the project, from a terminal:

| Command                   | Action                                           |
| :------------------------ | :----------------------------------------------- |
| `npm install`             | Installs dependencies                            |
| `npm run dev`             | Starts local dev server at `localhost:4321`      |
| `npm run build`           | Build your production site to `./dist/`          |
| `npm run preview`         | Preview your build locally, before deploying     |
| `npm run astro ...`       | Run CLI commands like `astro add`, `astro check` |
| `npm run astro -- --help` | Get help using the Astro CLI                     |
| `npm test`                | Run unit tests with Vitest                        |
| `npm test:ui`             | Run tests with Vitest UI                         |
| `npm run create-user`     | Create a new user via script                    |
| `npm run create-students` | Create test students via script                  |

## 🚀 Deployment

### Netlify Deployment

The project is configured for deployment to Netlify. See [DEPLOYMENT.md](./DEPLOYMENT.md) for detailed instructions.

**Quick Start:**

1. Configure environment variables in Netlify dashboard (see `.env.example`)
2. Connect your Git repository to Netlify
3. Deploy automatically on push to main branch

**Environment Variables Required:**

- Firebase Admin SDK (Server-side): `FIREBASE_DATA_PROJECT_ID`, `FIREBASE_DATA_PRIVATE_KEY`, `FIREBASE_DATA_CLIENT_EMAIL`
- Public Firebase Config (Client-side): `PUBLIC_FIREBASE_DATA_*`

**Cost:** Netlify free tier (100GB bandwidth/month, 300 minutes build/month) + Firebase free tier (50k reads/day, 20k writes/day per project) = **$0/month**
