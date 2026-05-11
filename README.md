📋 Table of Contents

Overview
Tech Stack
Features

Authentication
Multi-Step Registration
Face ID Biometric Verification
Math CAPTCHA Protection
Password Recovery (TOTP)
Profile Management
Admin Dashboard
Suspicion Scoring System
Real-Time Messaging
OAuth Social Login


Project Structure
Database Schema
Security Configuration
Installation & Setup
Environment Variables
API Endpoints
Author


🌟 Overview
The User Management Module of Harmony is a full-featured, security-focused system that handles the complete user lifecycle — from registration to daily authentication — for a student wellness platform. It combines classical form-based auth with modern biometric verification, OAuth social login, and AI-powered fraud detection.

Branch: user-management
Framework: Symfony 7.x
Language: PHP 8.2+


🛠 Tech Stack
LayerTechnologyBackend FrameworkSymfony 7.xORMDoctrine ORMDatabaseMySQL 8.0TemplatingTwigReal-timeMercure (Server-Sent Events)Face Recognitionface-api.js (TinyFaceDetector)OAuthKnpUOAuth2ClientBundlePDF ExportKnpSnappyBundle + wkhtmltopdfChartsChart.js 4.xTOTPCustom RFC 6238 implementationPassword Hashingbcrypt (cost: 12)

✨ Features
🔐 Authentication
A multi-layered authentication system ensuring both security and usability.
Form-Based Login:

Email + password authentication with bcrypt hashing (cost: 12)
Remember Me functionality — session maintained for 3 hours
CSRF token protection on every form submission
Flash messages for clear user feedback (errors, successes)
Automatic redirection: admins → admin dashboard, students → homepage

Social Login (OAuth 2.0):

Google OAuth — one-click login using Google account
Facebook OAuth — one-click login using Facebook account
Automatic account linking if email already exists in the system
New accounts created automatically on first OAuth login
OAuth avatar fetched and stored as profile picture

Role-Based Access Control:

ROLE_USER — standard student access
ROLE_ADMIN — full admin access (inherits ROLE_USER)
Route-level protection via security.yaml access control rules
Attribute-level protection via #[IsGranted] annotations


📝 Multi-Step Registration
A guided 3-step registration flow that progressively collects user information while maintaining data integrity across steps via PHP sessions.
Step 1 → Step 2 → Step 3
Email/Password  Physical/Academic  Face Capture
Step 1 — Identity:

First name, last name, email, password, date of birth
Password validation: minimum 8 chars, uppercase, digit, symbol
Password confirmation with mismatch detection
Real-time password strength indicator (Weak / Medium / Strong)
Email uniqueness check via UniqueEntity constraint

Step 2 — Physical & Academic Profile:

Gender (HOMME / FEMME / AUTRE)
Weight (40–200 kg) and Height (100–210 cm) with range validation
Physical activity level (SEDENTAIRE / LEGER / MODERE / INTENSE / TRES_INTENSE)
School level (PRIMAIRE → DOCTORAT) and institution name
Profile picture upload (JPG, PNG, WebP, GIF — max 2MB)

Step 3 — Biometric Enrollment:

Optional face capture via webcam
Base64 PNG stored server-side in /public/face_data/
Face ID enabled by default if photo is taken
"Skip" option available — account created without biometric data


🪪 Face ID Biometric Verification
A client-side biometric layer powered by face-api.js that adds a second authentication factor after successful password login.
How it works:

At login, if Face ID is enabled, the user is redirected to /face-verify
The server returns the reference image (stored at registration) as Base64
The webcam captures a live frame
face-api.js computes descriptors for both images using:

TinyFaceDetector (optimized for real-time)
faceLandmark68TinyNet (68 facial landmark points)
faceRecognitionNet (128-dimension face descriptor)


Euclidean distance is computed — match threshold: < 0.55
On success: session is marked face_verified = true, redirected to homepage
On failure: 3 attempts allowed, then automatic logout

AI Models used (loaded from /public/models/):

tiny_face_detector_model
face_landmark_68_tiny_model
face_recognition_model

Settings Management:

Toggle Face ID on/off from profile settings
Recapture face image at any time
Cannot enable Face ID without a stored reference image

Session Flow:
Login Success
     ↓
FaceAuthListener::onLogin()
     ↓  sets pending_face_user_id in session
FaceAuthListener::onRequest()
     ↓  intercepts every request until verified
/face-verify  →  check  →  success  →  homepage

🧮 Math CAPTCHA Protection
A custom server-side CAPTCHA system that protects the login endpoint from automated attacks without relying on any third-party service.
Mechanism:

Random addition operation generated on each page load (e.g., 7 + 4 = ?)
Answer stored in PHP session (captcha_answer)
CaptchaListener intercepts POST /login at priority 10 (before Symfony's firewall at priority 8)
If answer is wrong: flash error added, new CAPTCHA generated, redirect to login
If answer is correct: request continues to the authentication firewall
CAPTCHA regenerated after every attempt (correct or incorrect)

Implementation details:

Numbers range: 1–15 for each operand (result: 2–30)
Validation is trimmed string comparison (prevents whitespace issues)
Fully stateless on the client side — no JavaScript required


🔑 Password Recovery (TOTP)
A 3-step, email-free password recovery system based on RFC 6238 TOTP (Time-based One-Time Password), fully compatible with Google Authenticator and Authy.
Step 1 — Email Verification:

User enters their registered email address
System checks if the account exists

Step 2 — Authenticator Code:

If first time: a QR code is generated and displayed (via qrcodejs)
User scans QR code with Google Authenticator / Authy
TOTP secret stored in user.totp_secret (Base32 encoded, 80-bit entropy)
User enters the 6-digit code shown in their authenticator app
Live countdown timer shows seconds remaining before code expiry
Clock tolerance: ±1 period (±30 seconds) to handle clock drift

Step 3 — New Password:

Password strength indicator with 4 rules (length, uppercase, digit, symbol)
Confirmation field with real-time mismatch detection
bcrypt re-hashing on save, session invalidated after change

Custom TOTP Implementation (TotpService.php):

RFC 4226 HOTP base algorithm
RFC 6238 TOTP (30-second window)
Base32 encode/decode (RFC 4648)
HMAC-SHA1 with dynamic truncation
No external dependency — pure PHP


✏️ Profile Management
Users can manage all aspects of their profile from a unified settings page with tab-based navigation.
Tab 1 — Edit Profile:

Update first name, last name, date of birth, gender
Update physical data: weight, height, activity level
Update academic data: school level, institution
Avatar upload with live preview before saving
Form validation mirrors registration constraints

Tab 2 — Security:

Change email address → session invalidated, redirect to login
Change password → requires current password verification
Password strength indicator with visual feedback
Session invalidation after any credential change (security best practice)

Tab 3 — Face ID:

Toggle Face ID authentication on/off
Live webcam recapture to update reference image
Previous face image deleted on recapture
Cannot activate without a stored face image

Avatar Upload Flow:

File stored in /public/user_images/
Filename: {slugged-lastname}-{uniqid}.{ext}
Supported: JPG, PNG, WebP, GIF (max 2MB)
Directory auto-created if missing


📊 Admin Dashboard
A rich analytics dashboard giving administrators a real-time overview of the student population.
KPI Cards:

Total students registered
Active accounts count
Suspended accounts count

Interactive Charts (Chart.js 4):
ChartTypeDescriptionGender DistributionDoughnutHOMME / FEMME / AUTRE / Non renseignéSuspicion ScoresDoughnutNormal / Modéré / Suspect / Très suspectPhysical ActivityHorizontal Bar5 activity levelsRegistrations TimelineLineLast 12 monthsSchool LevelBarPRIMAIRE → DOCTORAT
Recent Registrations Table:

Latest 5 registered students
Quick links to full profile view

PDF Export:

Browser print dialog triggered via JavaScript
Print-specific CSS hides navigation and shows report header
Timestamp included in report header


🛡️ Suspicion Scoring System
An automated fraud detection system that scores every student account from 0 to 100 based on behavioral and data heuristics.
Scoring Criteria (total capped at 100):
CriterionPointsDetection LogicSuspicious words in name/surname30Matches against 25+ known fake words (test, admin, toto, foo, lorem…)Repetitive characters25Regex: 3+ consecutive identical charactersName equals surname20Exact string match (case-insensitive)Suspicious email domain25Matches 10+ known disposable domains (yopmail, mailinator…)Suspicious word in email15Same word list applied to email addressName/surname too short151 character or fewerNumeric sequences in name103+ consecutive digits
Risk Labels:
ScoreLabelColor0–14Normal🟢 Green15–34Modéré🟣 Purple35–59Suspect🟠 Orange60–100Très suspect🔴 Red
Admin Features:

Sort all users by suspicion score (highest first)
Live search with real-time card updates (no page reload)
Modal popup with full breakdown per user (criterion + points + detail)
Color-coded card borders matching risk level
Dedicated suspended accounts view


💬 Real-Time Messaging
A floating chat bubble accessible from every page of the application, powered by Mercure for real-time delivery.
Features:

Floating bubble (bottom-right) with unread count badge
Conversation list with last message preview and timestamp
User search to start new conversations (minimum 2 characters)
Message bubbles with sent time, differentiated by sender
Auto-scroll to latest message
Mark as read on conversation open

Real-Time (Mercure):

JWT token generated server-side per user
Subscribe to personal topic: user/{userId}/messages
Fallback polling every 5 seconds if Mercure unavailable
Incoming messages appended without page reload

API Endpoints:
MethodEndpointDescriptionGET/api/messaging/conversationsList user conversationsGET/api/messaging/conversations/{id}/messagesLoad messages + mark as readPOST/api/messaging/sendSend a messageGET/api/messaging/search-usersSearch users to messageGET/api/messaging/unread-countGet unread message countGET/api/messaging/mercure-tokenGet JWT for Mercure subscription

📁 Project Structure
src/
├── Controller/
│   ├── AdminDashboardController.php   # Admin KPIs & charts
│   ├── AdminUserController.php        # User CRUD + suspicion API
│   ├── FaceAuthController.php         # Face ID verify/toggle/update
│   ├── HomepageController.php         # Entry point + role redirect
│   ├── MessagingController.php        # Real-time chat API
│   ├── OAuthController.php            # Google & Facebook OAuth
│   ├── ProfileController.php          # Settings tabs
│   ├── RegistrationController.php     # 3-step registration
│   └── SecurityController.php        # Login + CAPTCHA
│
├── Entity/
│   ├── User.php                       # Main user entity
│   ├── Conversation.php               # Messaging conversation
│   └── Message.php                    # Individual message
│
├── EventListener/
│   ├── CaptchaListener.php            # CAPTCHA pre-firewall check
│   ├── FaceAuthListener.php           # Face ID session enforcement
│   ├── TrustedHostListener.php        # Host header normalization
│   └── DebugRequestListener.php       # Dev debug logging
│
├── Form/
│   ├── RegistrationFormType.php       # Step 1 form
│   ├── RegistrationStep2FormType.php  # Step 2 form
│   ├── ProfileFormType.php            # Profile edit form
│   ├── SecuritySettingsFormType.php   # Security tab form
│   └── AdminEditUserFormType.php      # Admin user edit form
│
├── Repository/
│   ├── UserRepository.php             # User queries
│   ├── ConversationRepository.php     # findOrCreate logic
│   └── MessageRepository.php         # markAsRead, unread count
│
├── Security/
│   ├── GoogleAuthenticator.php        # Google OAuth authenticator
│   └── FacebookAuthenticator.php     # Facebook OAuth authenticator
│
└── Service/
    ├── CaptchaService.php             # CAPTCHA generation & validation
    ├── SuspicionScoreService.php      # Fraud scoring engine
    └── TotpService.php                # RFC 6238 TOTP implementation

templates/
├── security/
│   ├── login.html.twig                # Login page
│   └── face_verify.html.twig         # Face ID verification page
├── registration/
│   ├── register_step1.html.twig
│   ├── register_step2.html.twig
│   └── register_step3.html.twig
├── profile/
│   └── settings.html.twig            # Profile settings (3 tabs)
├── admin/
│   ├── base.html.twig                 # Admin layout + sidebar
│   ├── dashboard.html.twig           # Admin analytics
│   └── users/
│       ├── index.html.twig           # User cards grid
│       ├── show.html.twig            # User detail
│       ├── edit.html.twig            # Edit form
│       └── suspended.html.twig      # Suspended list
├── messaging/
│   └── bubble.html.twig              # Floating chat widget
└── base.html.twig                    # Main layout

🗃️ Database Schema
Table: user
ColumnTypeDescriptionuser_idINT (PK, AUTO)Primary keyuser_nomVARCHAR(50)Last nameuser_prenomVARCHAR(50)First nameuser_emailVARCHAR(100) UNIQUEEmail (user identifier)user_passwordVARCHAR(255)bcrypt hashed passworduser_date_de_naissanceVARCHAR(10)Date of birth (YYYY-MM-DD)user_sexeENUM('HOMME','FEMME','AUTRE')Genderuser_poidsDECIMAL(5,2)Weight in kguser_tailleINTHeight in cmuser_niveau_activite_physiqueENUMPhysical activity leveluser_niveau_scolaireENUMSchool leveluser_etablissement_scolaireVARCHAR(255)Institution namedate_inscriptionVARCHAR(10)Registration datetype_utilisateurENUM('ETUDIANT','ADMIN')Roleis_activeBOOLEANAccount statususer_image_pathVARCHAR(255)Profile picture pathface_image_pathVARCHAR(500)Face ID reference imageface_id_enabledBOOLEANFace ID togglegoogle_idVARCHAR(255) UNIQUEGoogle OAuth IDfacebook_idVARCHAR(255) UNIQUEFacebook OAuth IDoauth_avatar_urlVARCHAR(500)OAuth profile picture URLtotp_secretVARCHAR(255)TOTP secret for password recovery
Table: conversation
ColumnTypeDescriptionidINT (PK)Primary keyuser1_idINT (FK)First participantuser2_idINT (FK)Second participantupdated_atDATETIMELast activity timestamp
Table: message
ColumnTypeDescriptionidINT (PK)Primary keyconversation_idINT (FK)Parent conversationsender_idINT (FK)Message sendercontentTEXTMessage contentsent_atDATETIMESend timestampis_readBOOLEANRead status

🔒 Security Configuration
Firewall (security.yaml):
yamlsecurity:
  password_hashers:
    App\Entity\User:
      algorithm: bcrypt
      cost: 12

  firewalls:
    main:
      form_login:
        login_path: app_login
        check_path: app_login
      remember_me:
        lifetime: 10800   # 3 hours
Access Control:
PathRequired Role/login, /register, /face-verifyPUBLIC/forgot-passwordPUBLIC/connect/google, /connect/facebookPUBLIC/admin/**ROLE_ADMIN/profile/**ROLE_USER/**ROLE_USER
Event Listener Priority Stack:
ListenerPriorityActionTrustedHostListener255Normalize HOST headerCaptchaListener10Validate CAPTCHA before firewallSymfony Firewall8Standard authenticationFaceAuthListener7Enforce Face ID session check

⚙️ Installation & Setup
Prerequisites

PHP 8.2+
Composer
MySQL 8.0+
Node.js (for asset management)
wkhtmltopdf (for PDF export)
Caddy server (for Mercure)

Steps
bash# 1. Clone the repository
git clone https://github.com/your-org/harmony.git
cd harmony
git checkout user-management

# 2. Install PHP dependencies
composer install

# 3. Configure environment
cp .env .env.local
# Edit .env.local with your values (see Environment Variables section)

# 4. Create database & run migrations
php bin/console doctrine:database:create
php bin/console doctrine:migrations:migrate

# 5. Install assets
php bin/console assets:install

# 6. Download face-api.js models
# Place in /public/models/:
# - tiny_face_detector_model-weights_manifest.json
# - tiny_face_detector_model-shard1
# - face_landmark_68_tiny_model-weights_manifest.json
# - face_landmark_68_tiny_model-shard1
# - face_recognition_model-weights_manifest.json
# - face_recognition_model-shard1

# 7. Start Mercure (real-time messaging)
caddy run --config dev.Caddyfile

# 8. Start Symfony dev server
symfony server:start

🌍 Environment Variables
env# App
APP_ENV=dev
APP_SECRET=your_secret_key_here

# Database
DATABASE_URL="mysql://user:password@127.0.0.1:3306/harmony?serverVersion=8.0.32"

# Mercure (real-time messaging)
MERCURE_URL=http://localhost:3000/.well-known/mercure
MERCURE_PUBLIC_URL=http://localhost:3000/.well-known/mercure
MERCURE_JWT_SECRET=your_mercure_jwt_secret

# Google OAuth
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret

# Facebook OAuth
FACEBOOK_CLIENT_ID=your_facebook_app_id
FACEBOOK_CLIENT_SECRET=your_facebook_app_secret

# PDF Export
WKHTMLTOPDF_PATH=C:/wkhtmltopdf/bin/wkhtmltopdf.exe

📡 API Endpoints
Authentication
MethodRouteDescriptionGET/POST/loginLogin page + form submitGET/logoutLogoutGET/registerRegistration Step 1GET/POST/register/step2Registration Step 2GET/register/step3Registration Step 3 (face)POST/register/save-faceSave face image & create accountPOST/register/skip-faceCreate account without face
Face ID
MethodRouteDescriptionGET/face-verifyFace verification pagePOST/face-verify/checkGet reference imagePOST/face-verify/successConfirm successful matchPOST/profile/toggle-faceidEnable/disable Face IDPOST/profile/update-faceUpdate face reference image
Password Recovery
MethodRouteDescriptionGET/POST/forgot-passwordStep 1 — Enter emailPOST/forgot-password/verifyStep 2 — Verify TOTP codePOST/forgot-password/resetStep 3 — Set new password
Admin
MethodRouteDescriptionGET/adminAdmin dashboardGET/admin/usersUser listGET/admin/users/searchLive search (JSON)GET/admin/users/suspendedSuspended accountsGET/admin/users/{id}User detailGET/POST/admin/users/{id}/editEdit userPOST/admin/users/{id}/toggleSuspend/reactivateGET/admin/users/{id}/suspicionSuspicion breakdown (JSON)
Messaging
MethodRouteDescriptionGET/api/messaging/conversationsList conversationsGET/api/messaging/conversations/{id}/messagesLoad messagesPOST/api/messaging/sendSend messageGET/api/messaging/search-usersSearch usersGET/api/messaging/unread-countUnread countGET/api/messaging/mercure-tokenMercure JWT

👨‍💻 Author
Harmony Project — User Management Module
Developed as part of a multi-team student platform integration project.

Built with ❤️ using Symfony 7, Doctrine ORM, face-api.js, and Mercure
