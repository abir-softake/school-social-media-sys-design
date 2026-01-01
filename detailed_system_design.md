# Detailed System Design

## 1) Diagram (logical overview)

Clients
- Web / Mobile UI: REST for CRUD/feeds, WebSocket for realtime chat/notifications.

High-level diagram (logical):

    Client (Web/Mobile)
    ├─ REST → API Gateway / Django DRF
    │         ├─ Auth (JWT)  
    │         ├─ Posts / Feeds / Admin APIs
    │         └─ Payments / CV generation / Ads endpoints
    └─ WebSocket → ASGI (Uvicorn/Daphne) + Channels Consumers
              ├─ Chat channels (course/inst/user)
              └─ Live notifications (user-scoped)

    Backend (Monolith initially)
    ├─ Django + DRF (HTTP APIs)
    ├─ Django Channels (WS consumers)
    ├─ Celery / RQ (background workers)
    ├─ Redis (channel layer, cache, presence)
    ├─ PostgreSQL (primary relational DB)
    └─ Object store (S3) for attachments/media

    Supporting or extractable services
    ├─ Auth service (JWT token management, session integration)
    ├─ Notifications service (email, push, SMS gateways)
    ├─ SMS / Email gateways (3rd party)
    ├─ Payments service (PCI flow, webhooks)
    ├─ CV generation service (on-demand worker)
    └─ Advertisement service (internal or 3rd-party)

Relationship view (logical model connections):
    User ↔ Profile (1:1)
    User —< Enrollment >— Course
    Institution —< Department, Course, Faculty, Club
    Course —< Curriculum, Exam, Attendance, Message
    Post (polymorphic owner: User/Institution/Department/Facility)
    Message (channel scoped by Course/Institution/User)
    Role / UserRole / Permission govern access to above resources

Notification & side-effect flows (complete list aggregated from project files):


**Middlewares required:**
- custom jwt auth middleware
- performance and error logger
- respnse formatting middleware
- content safety middleware
- global paginator

**Email cases:**
- Registration (verification email)
- Successful verification (welcome/activation)
- Forgot password / password reset (reset link/code)
- Exam date declarations
- Exam results publication / result-ready notifications
- Enrollment/admission updates (accepted/rejected/selected)
- Payment receipts and payment-failure alerts
- New curriculum / course announcements (optional digests)
- New institution notices/announcements (digest or targeted emails)
- CV generation completion (download link)
- Audit reports and compliance notifications (when required)

**SMS cases:**
- Registration verification codes
- Password reset codes
- Critical exam date alerts (time-sensitive)
- Payment confirmations (receipt/OTP for payment flows)
- Enrollment/admission status changes (urgent updates)
- Role-change alerts where immediate action is required

**WebSocket / In-app / Push cases (immediate/live):**
- Live chat messages and direct texts
- Live notifications for new posts/announcements in followed institutions/departments
- Presence/online status updates (when permitted)
- Live reactions / likes / comment updates (optimistic updates)
- Real-time enrollment/admission status updates (UI push)
- Instant noticeboard posts and course updates

**Background-job notifications (via notifications service when complete):**
- CV generation complete
- Media processing / attachment thumbnails ready
- Automatic result generation complete
- Heavy feed recomputation finished

**System / Ops / Audit alerts:**
- Cron/cleanup job failures or important results
- Audit events: role assignments, permission changes (notify compliance/ops)
- Advertisement billing/errors (notify system admins)

Delivery channels guidance:
- Immediate channels: WebSocket, push notifications, in-app notifications for live UX.
- Persistent/asynchronous channels: Email and SMS for critical flows and deliverability fallback.
- Background work: Celery/RQ executes heavy tasks and notifies via the notifications service when finished.

This list is assembled directly from `other services.txt`, `realtime_architecture.txt`, `social_media.txt`, `user.txt`, and `Institution.txt` to ensure every scenario mentioned in the project files is covered.

## 2) System architecture summary
- Monolith Django app recommended initially (DRF + Channels + Celery). PostgreSQL is source-of-truth, Redis for channel-layer & cache, S3 for attachments/media.
- Auth: JWT (short-lived access, refresh), encode minimal role-scopes in tokens.
- Permissions: RBAC with scoped roles, optional ResourceACL and ownership fallback. Cache permission checks in Redis for performance.
- Realtime: Channels consumers authenticate via JWT, check `realtime_send`/`read` perms, use Redis pub/sub and persist messages to DB (or via background task).
- Background tasks: email/SMS/notifications, heavy media processing, feed recompute.
- Observability: centralized logs, metrics, backups for DB/storage.

## 3) Model relationships

Core user & identity:
- `User` 1:1 `Profile` (profile may store contacts, social links, following lists).
- `Profile.friends` via `Friendship` (from_user -> to_user; status: pending/accepted/blocked).
- `SchoolListing` links `Institution` -> `User` (user-school membership/history).

Institution & hierarchy:
- `Institution` has owner -> `User` (owner FK).
- `Institution` 1:N `Department` (department.institute FK).
- `Institution` 1:N `Course` (course.institute FK).
- `Institution` 1:N `Faculty` (faculty.institution FK).
- `Institution` 1:N `Club` (club inherits BaseModel fields and links to institution via association).
- `Institution` 1:N `Facility` (institutions can list many facilities; facilities reused across institutions).
- `Institution` has many `Session` (session.institute FK) and many `Admission` entries.

Department & faculty composition:
- `Department.parent_department` -> self (hierarchical departments).
- `Department.department_head` -> `User` (head FK).
- `Department.staffs` M:N `User` (administrative staff)
- `Department.teachers` M:N `User` (teaching staff)
- `Department.students` M:N `User` (student membership)
- `Department.courses` M:N/1:N `Course` (courses attached to departments).

Course & academic objects:
- `Course.course_teacher` -> `User` (primary instructor FK).
- `Course.session` -> `Session` (active session/term FK).
- `Course.schedule` -> `Schedule` (time pattern FK).
- `Course` 1:N `Curriculum` (course content items/homework/assignments).
- `Course` 1:N `Exam`.
- `Course` 1:N `Attendance` records.
- `Course` 1:N `Enrollment` entries (students enrolled in course).
- `Course.attachments` M:N `Attachment` (course materials/media).

Enrollment, Admission & Schedule:
- `Enrollment.course` -> `Course`; `Enrollment.student` -> `User`.
- `Enrollment.lecturer` -> `User` (assigned lecturer/TA)
- `Enrollment.session` -> `Session`; `Enrollment.institution` -> `Institution`.
- `Enrollment.schedule` -> `Schedule` to align class timing.
- `Admission.session` -> `Session`; `Admission.user` -> `User` (applicant); `Admission.faculty` -> `Faculty`.
- `Admission.applied_users` and `Admission.selected_users` are M:N with `User` (applicant pools and selections).

Exams & results:
- `Exam.course` -> `Course` and `Exam.institution` -> `Institution`.
- `Exam.examinees` M:N `User` (registered students for an exam).
- Exam result publishing triggers notifications and may create `Enrollment.result` or separate `Result` records.

Attendance & timetabling:
- `Attendance.user` -> `User`, `Attendance.course` -> `Course`, `Attendance.institute` -> `Institution`, `Attendance.schedule` -> `Schedule`.
- `Schedule` reused by `Course` and `Enrollment` to represent class/time blocks.

Content, social & noticeboard:
- `Post` (polymorphic owner): `owner_type` + `owner_uid` references `User`, `Institution`, `Department`, or `Facility`.
- `Reply` -> FK `post` (one-to-many replies per post); `Reply.parent_reply` supports nested replies.
- `NoticeBoard.notice_to_type` + `notice_to_uid` is polymorphic (department, course, institution, user).
- `Post` and `NoticeBoard` may reference `Attachment` for media.

Attachments & achievements (polymorphic):
- `Attachment.a_from` + `a_from_type` references the source owner (User/Institution/Department/Club/Facility).
- `Achievement.achievement_from` + `a_from_type` + `achievement_to` link achievements between entities (user-to-institution, club awards, etc.).

Faculties, clubs & facilities:
- `Faculty.members` M:N `User` (faculty membership list); `Faculty.dean` -> `User`.
- `Club.leader` and `Club.student_coordinator` -> `User`; `Club.members` M:N `User`.
- `Facility` objects referenced by `Institution.facilities` (M:N) and by posts/owners as `owner_type`.

Messaging & realtime:
- `Message.sender` -> `User`; `Message.channel` is a namespaced string (e.g., `course:{id}`, `inst:{id}`, `user:{id}`) mapping to course/institution/user channels.
- Channels consumers authorize joins based on `UserRole` scopes and per-channel permission (`realtime_send`/`read`).
- Messages persisted in `Message` table; presence (online users) maintained in Redis.

Friendship, following & school history:
- `Friendship.from_user` -> `User`, `Friendship.to_user` -> `User` (state machine: pending/accepted/blocked).
- `Profile.following_user` M:N `User` (follow relationships); `Profile.following_institutes` stored as list/JSON or M:N depending on implementation.
- `SchoolListing.from_school` -> `Institution`, `SchoolListing.to_user` -> `User` records historic enrollments/affiliations.

Auth, roles & permissions:
- `Role` M:N `Permission` through `RolePermission`.
- `UserRole.user` -> `User`, `UserRole.role` -> `Role`, with `UserRole.scope` indicating scope (e.g., `inst:123`, `dept:45`) and optional `expires_at`.
- `ResourceACL` provides per-object overrides (content_type + object_id → role allow/deny entries).
- Role hierarchy: `super_admin` > `institute_admin` > `campus_admin` > `teacher/staff` > `student/basic_user` (enforced in permission checks).

Services & side-effect mappings (models that trigger notifications/tasks):
- `User` events: registration, verification, password reset → triggers Email/SMS/Notifications.
- `Exam`, `Enrollment`, `Admission`, `Payment` model changes trigger Notifications & background tasks (result generation, receipts).
- `Post`, `NoticeBoard`, `Curriculum` creates/updates trigger feed updates, cache invalidation, and WS notifications.
- `Attachment` uploads trigger media processing workers and update referencing models when processed.

Notes on polymorphism & implementation choices:
- Polymorphic relations use lightweight `owner_type` + `owner_uid` or Django `ContentType` for flexibility; consider `GenericForeignKey` where you need ORM convenience.
- Many-to-many collections in files sometimes overlap (e.g., `Institution.courses` and `Course.institute` both present); choose a single authoritative direction for FK vs M2M to avoid duplication.
- Cache hot paths (feed counts, like counts, presence) are stored in Redis and should be derived from authoritative DB records asynchronously.

This detailed relationships list is constructed from the project's text files (`Institution.txt`, `common.txt`, `social_media.txt`, `realtime_architecture.txt`, `other services.txt`, `user.txt`) and expands every explicit/implicit link mentioned.

## 4) Realtime/chat specific requirements (from files)
- Chat channels must support: authentication via token, join/leave, send/receive, presence, live likes/comments.
- Authorization per-channel: require `realtime_send` or `read` depending on channel type; scope channels by course/institution/department or user private rooms.
- Redis channel layer used for pub/sub and presence; message persistence to DB is required (messages table referenced in architecture).
- Rate-limiting by role to prevent abuse.
- Presence details restricted (detailed presence only to admins/teachers).

## 5) Django models (skeleton classes derived from files)
Note: these are schema-level definitions following the files. Use `django.db.models` types; adapt types to requirements (JSONField for lists/metadata).

```python
from django.db import models
from django.contrib.postgres.fields import JSONField

# --- Choice enums (TextChoices) ---
class UserStatus(models.TextChoices):
    NOT_VERIFIED = 'NOT_VERIFIED', 'Not Verified'
    ACTIVE = 'ACTIVE', 'Active'
    DEACTIVED = 'DEACTIVED', 'Deactivated'
    PASS_RESET_REQUIRED = 'PASS_RESET_REQUIRED', 'Password Reset Required'
    BANNED = 'BANNED', 'Banned'
    INACTIVE = 'INACTIVE', 'Inactive'


class UserType(models.TextChoices):
    SUPER_ADMIN = 'super_admin', 'Super Admin'
    INSTITUTE_ADMIN = 'institute_admin', 'Institute Admin'
    CAMPUS_ADMIN = 'campus_admin', 'Campus Admin'
    STUDENT = 'student', 'Student'
    TEACHER = 'teacher', 'Teacher'
    BASIC_USER = 'basic_user', 'Basic User'
    PARENT = 'parent', 'Parent'
    STAFF = 'staff', 'Staff'


class InstitutionPlatform(models.TextChoices):
    ONLINE = 'online', 'Online'
    OFFLINE = 'offline', 'Offline'
    HYBRID = 'hybrid', 'Hybrid'


class DepartmentType(models.TextChoices):
    EDUCATIONAL = 'educational', 'Educational'
    MANAGING_BODY = 'managing_body', 'Managing Body'
    GENERAL = 'general', 'General'
    OTHERS = 'others', 'Others'


class CourseLevel(models.TextChoices):
    UG = 'UG', 'Undergraduate'
    PG = 'PG', 'Postgraduate'
    SSC = 'SSC', 'Secondary'
    DIPLOMA = 'DIPLOMA', 'Diploma'
    CERTIFICATE = 'CERTIFICATE', 'Certificate'


class PeriodPattern(models.TextChoices):
    MONTHLY = 'monthly', 'Monthly'
    ANNUAL = 'annual', 'Annual'
    SEMESTER = 'semester', 'Semester'
    MANUAL = 'manual', 'Manual'


class EnrollmentStatus(models.TextChoices):
    PENDING = 'pending', 'Pending'
    ACCEPTED = 'accepted', 'Accepted'
    EXPIRED = 'expired', 'Expired'
    OVER = 'over', 'Over'


class PaymentStatus(models.TextChoices):
    DONE = 'done', 'Done'
    PENDING = 'pending', 'Pending'


class ExamPlatform(models.TextChoices):
    ONLINE = 'online', 'Online'
    OFFLINE = 'offline', 'Offline'


class PostType(models.TextChoices):
    POST = 'post', 'Post'
    ANNOUNCEMENT = 'announcement', 'Announcement'


class ReplyType(models.TextChoices):
    TEXT = 'text', 'Text'
    REACTION = 'reaction', 'Reaction'


class NoticeTargetType(models.TextChoices):
    DEPARTMENT = 'department', 'Department'
    COURSE = 'course', 'Course'
    INSTITUTION = 'institution', 'Institution'
    USER = 'user', 'User'


### Base model
class BaseModel(models.Model):
    uid = models.UUIDField(primary_key=True, editable=False)
    display_name = models.CharField(max_length=255)
    status = models.CharField(
        max_length=32,
        choices=[('active', 'active'), ('inactive', 'inactive')],
    )
    logo = models.URLField(blank=True, null=True)
    cover_image = models.URLField(blank=True, null=True)
    achievements = JSONField(blank=True, null=True) 
    code = models.CharField(max_length=128, blank=True, null=True)
    established_date = models.DateField(blank=True, null=True)
    description = models.TextField(blank=True, null=True)

    class Meta:
        abstract = True


### User & Profile
class User(models.Model):
    uuid = models.UUIDField(primary_key=True, editable=False)
    profile = models.OneToOneField('Profile', on_delete=models.CASCADE, null=True, blank=True)
    email = models.EmailField(unique=True)
    password = models.CharField(max_length=128)
    phone_number = models.CharField(max_length=32, blank=True, null=True)
    status = models.CharField(max_length=32, choices=UserStatus.choices, default=UserStatus.NOT_VERIFIED)
    type = models.CharField(max_length=64, choices=UserType.choices, default=UserType.BASIC_USER)
    username = models.CharField(max_length=150, unique=True)
    firstname = models.CharField(max_length=100, blank=True, null=True)
    lastname = models.CharField(max_length=100, blank=True, null=True)
    avatar = models.URLField(blank=True, null=True)


class Profile(models.Model):
    uuid = models.UUIDField(primary_key=True, editable=False)
    user = models.OneToOneField('User', on_delete=models.CASCADE)
    bio = models.TextField(blank=True, null=True)
    address = models.TextField(blank=True, null=True)
    other_emails = JSONField(blank=True, null=True)
    other_phone_numbers = JSONField(blank=True, null=True)
    profile_images = JSONField(blank=True, null=True)
    birthday = models.DateField(blank=True, null=True)
    gender = models.CharField(max_length=32, blank=True, null=True)
    language = models.CharField(max_length=32, blank=True, null=True)
    timezone = models.CharField(max_length=64, blank=True, null=True)
    websites = JSONField(blank=True, null=True)
    social_links = JSONField(blank=True, null=True)
    achievements = JSONField(blank=True, null=True)
    friends = models.ManyToManyField(
        'self', through='Friendship', symmetrical=False, related_name='friend_of'
    )
    school_history = JSONField(blank=True, null=True)  # or through SchoolListing
    cover_image = models.URLField(blank=True, null=True)
    resume = models.URLField(blank=True, null=True)
    following_user = models.ManyToManyField('User', related_name='followers', blank=True)
    following_institutes = JSONField(blank=True, null=True)


class Friendship(models.Model):
    from_user = models.ForeignKey('User', related_name='friendship_from', on_delete=models.CASCADE)
    to_user = models.ForeignKey('User', related_name='friendship_to', on_delete=models.CASCADE)
    status = models.CharField(max_length=32)  # pending, accepted, blocked
    accepted_at = models.DateTimeField(null=True, blank=True)


class SchoolListing(models.Model):
    from_school = models.ForeignKey('Institution', on_delete=models.CASCADE)
    to_user = models.ForeignKey('User', on_delete=models.CASCADE)
    status = models.CharField(max_length=32)  # pending, accepted, blocked, rejected, retired


### Achievements & Attachments
class Achievement(models.Model):
    achievement_from = models.UUIDField()  # FK to User/Institution/Department/Club (use content-type or polymorphic)
    a_from_type = models.CharField(max_length=64)
    achievement_to = models.UUIDField()
    date_achieved = models.DateField(null=True, blank=True)
    expiration_date = models.DateField(null=True, blank=True)
    title = models.CharField(max_length=255)
    description = models.TextField(blank=True, null=True)
    attachment = models.ForeignKey('Attachment', null=True, blank=True, on_delete=models.SET_NULL)


class Attachment(models.Model):
    a_from = models.UUIDField()
    a_from_type = models.CharField(max_length=64)
    title = models.CharField(max_length=255)
    description = models.TextField(blank=True, null=True)
    file_url = models.URLField()
    expiration_date = models.DateField(null=True, blank=True)


### Institution-related
class Institution(BaseModel):
    owner = models.ForeignKey('User', on_delete=models.SET_NULL, null=True)
    title = models.CharField(max_length=255)
    platform = models.CharField(
        max_length=32, choices=InstitutionPlatform.choices, default=InstitutionPlatform.ONLINE
    )
    location_metadata = JSONField(blank=True, null=True)
    accredation_metadata = JSONField(blank=True, null=True)
    departments = models.ManyToManyField('Department', blank=True)
    courses = models.ManyToManyField('Course', blank=True)
    language = models.CharField(max_length=64, blank=True, null=True)
    emails = JSONField(blank=True, null=True)
    websites = JSONField(blank=True, null=True)
    registration_metadata = JSONField(blank=True, null=True)
    facilities = models.ManyToManyField('Facility', blank=True)
    inspection_metadata = JSONField(blank=True, null=True)
    ranking_metadata = JSONField(blank=True, null=True)
    faculties = models.ManyToManyField('Faculty', blank=True)
    user_rating = models.FloatField(null=True, blank=True)
    reviews = JSONField(blank=True, null=True)
    education_levels = JSONField(blank=True, null=True)
    clubs = models.ManyToManyField('Club', blank=True)
    admissions = models.ManyToManyField('Admission', blank=True)
    type = models.CharField(max_length=64, blank=True, null=True)


class Session(models.Model):
    institute = models.ForeignKey('Institution', on_delete=models.CASCADE)
    start_date = models.DateField()
    end_date = models.DateField()
    term = models.CharField(max_length=64)
    year = models.IntegerField()
    title = models.CharField(max_length=255)
    status = models.CharField(max_length=64)


class Department(BaseModel):
    institute = models.ForeignKey('Institution', on_delete=models.CASCADE)
    title = models.CharField(max_length=255)
    session = models.ForeignKey('Session', null=True, blank=True, on_delete=models.SET_NULL)
    type = models.CharField(max_length=64, choices=DepartmentType.choices, blank=True, null=True)
    department_head = models.ForeignKey('User', null=True, blank=True, on_delete=models.SET_NULL)
    parent_department = models.ForeignKey('self', null=True, blank=True, on_delete=models.SET_NULL)
    emails = JSONField(blank=True, null=True)
    phone_numbers = JSONField(blank=True, null=True)
    space_location_metadata = JSONField(blank=True, null=True)
    staffs = models.ManyToManyField('User', related_name='department_staffs', blank=True)
    teachers = models.ManyToManyField('User', related_name='department_teachers', blank=True)
    courses = models.ManyToManyField('Course', blank=True)
    language = models.CharField(max_length=64, blank=True, null=True)
    students = models.ManyToManyField('User', related_name='department_students', blank=True)
    attachments = models.ManyToManyField('Attachment', blank=True)
    learning_objectives = JSONField(blank=True, null=True)


class Course(BaseModel):
    institute = models.ForeignKey('Institution', on_delete=models.CASCADE)
    course_instructor_uid = models.ForeignKey('User', null=True, blank=True, on_delete=models.SET_NULL)
    session = models.ForeignKey('Session', null=True, blank=True, on_delete=models.SET_NULL)
    department = models.ForeignKey('Department', null=True, blank=True, on_delete=models.SET_NULL)
    course_teacher = models.ForeignKey(
        'User', null=True, blank=True, on_delete=models.SET_NULL, related_name='teaching_courses'
    )
    schedule = models.ForeignKey('Schedule', null=True, blank=True, on_delete=models.SET_NULL)
    shift = models.CharField(max_length=64, blank=True, null=True)
    course_level = models.CharField(max_length=64, choices=CourseLevel.choices, blank=True, null=True)
    total_credit = models.IntegerField(null=True, blank=True)
    duration_days = models.IntegerField(null=True, blank=True)
    period_pattern = models.CharField(max_length=64, choices=PeriodPattern.choices, blank=True, null=True)
    language = models.CharField(max_length=64, blank=True, null=True)
    course_fees = models.DecimalField(max_digits=10, decimal_places=2, null=True, blank=True)
    attachments = models.ManyToManyField('Attachment', blank=True)
    learning_objectives = JSONField(blank=True, null=True)


class Curriculum(models.Model):
    course = models.ForeignKey('Course', on_delete=models.CASCADE)
    title = models.CharField(max_length=255)
    type = models.CharField(max_length=64)  # homework, attachment, assignment
    start_date = models.DateField(null=True, blank=True)
    end_date = models.DateField(null=True, blank=True)
    attachment = models.ForeignKey('Attachment', null=True, blank=True, on_delete=models.SET_NULL)
    submission = JSONField(blank=True, null=True)
    remarks = models.TextField(blank=True, null=True)
    points = models.IntegerField(null=True, blank=True)


class Faculty(BaseModel):
    institution = models.ForeignKey('Institution', on_delete=models.CASCADE)
    education_levels = JSONField(blank=True, null=True)
    dean = models.ForeignKey('User', null=True, blank=True, on_delete=models.SET_NULL)
    attachment = models.ForeignKey('Attachment', null=True, blank=True, on_delete=models.SET_NULL)
    members = models.ManyToManyField('User', blank=True)


class Club(BaseModel):
    leader = models.ForeignKey('User', null=True, blank=True, related_name='club_leader', on_delete=models.SET_NULL)
    student_coordinator = models.ForeignKey(
        'User', null=True, blank=True, related_name='club_coordinator', on_delete=models.SET_NULL
    )
    attachments = models.ManyToManyField('Attachment', blank=True)
    type = models.CharField(max_length=64, blank=True, null=True)
    members = models.ManyToManyField('User', blank=True)


class Facility(models.Model):
    description = models.TextField(blank=True, null=True)
    type = models.CharField(max_length=128)


class Schedule(models.Model):
    title = models.CharField(max_length=255)
    saturday_in = models.TimeField(null=True, blank=True)
    saturday_out = models.TimeField(null=True, blank=True)
    sunday_in = models.TimeField(null=True, blank=True)
    sunday_out = models.TimeField(null=True, blank=True)
    monday_in = models.TimeField(null=True, blank=True)
    monday_out = models.TimeField(null=True, blank=True)
    tuesday_in = models.TimeField(null=True, blank=True)
    tuesday_out = models.TimeField(null=True, blank=True)
    wednesday_in = models.TimeField(null=True, blank=True)
    wednesday_out = models.TimeField(null=True, blank=True)
    thursday_in = models.TimeField(null=True, blank=True)
    thursday_out = models.TimeField(null=True, blank=True)
    friday_in = models.TimeField(null=True, blank=True)
    friday_out = models.TimeField(null=True, blank=True)
    time_flexibility = models.BooleanField(default=False)


class Enrollment(models.Model):
    uid = models.UUIDField(primary_key=True, editable=False)
    course = models.ForeignKey('Course', on_delete=models.CASCADE)
    student = models.ForeignKey('User', related_name='enrollments', on_delete=models.CASCADE)
    session = models.ForeignKey('Session', null=True, blank=True, on_delete=models.SET_NULL)
    lecturer = models.ForeignKey('User', null=True, blank=True, related_name='lectures', on_delete=models.SET_NULL)
    institution = models.ForeignKey('Institution', on_delete=models.CASCADE)
    schedule = models.ForeignKey('Schedule', null=True, blank=True, on_delete=models.SET_NULL)
    academic_session = models.CharField(max_length=255, blank=True, null=True)
    enrollment_date = models.DateTimeField(null=True, blank=True)
    status = models.CharField(max_length=64, choices=EnrollmentStatus.choices, default=EnrollmentStatus.PENDING)
    result = JSONField(blank=True, null=True)
    expire_date = models.DateField(null=True, blank=True)
    completion_date = models.DateField(null=True, blank=True)
    attendance_percentage = models.FloatField(null=True, blank=True)
    total_attendance = models.IntegerField(null=True, blank=True)
    scholarship_applied = models.BooleanField(default=False)
    payment_status = models.CharField(max_length=32, choices=PaymentStatus.choices, blank=True, null=True)
    remarks = models.TextField(blank=True, null=True)


class Admission(BaseModel):
    session = models.ForeignKey('Session', on_delete=models.CASCADE)
    user = models.ForeignKey('User', on_delete=models.CASCADE)
    faculty = models.ForeignKey('Faculty', null=True, blank=True, on_delete=models.SET_NULL)
    exam_date = models.DateField(null=True, blank=True)
    applied_users = models.ManyToManyField('User', related_name='applied_admissions', blank=True)
    admission_datetime = models.DateTimeField(null=True, blank=True)
    selected_users = models.ManyToManyField('User', related_name='selected_admissions', blank=True)


class Exam(models.Model):
    course = models.ForeignKey('Course', on_delete=models.CASCADE)
    examinees = models.ManyToManyField('User', blank=True)
    exam_date = models.DateField(null=True, blank=True)
    start_time = models.TimeField(null=True, blank=True)
    end_time = models.TimeField(null=True, blank=True)
    institution = models.ForeignKey('Institution', on_delete=models.CASCADE)
    title = models.CharField(max_length=255)
    platform = models.CharField(max_length=32, choices=ExamPlatform.choices, blank=True, null=True)
    total_marks = models.IntegerField(null=True, blank=True)
    earned_marks = models.IntegerField(null=True, blank=True)
    remarks = models.TextField(blank=True, null=True)
    result_publish_date = models.DateField(null=True, blank=True)
    earned_score = models.FloatField(null=True, blank=True)
    score_scheme = models.CharField(max_length=64, blank=True, null=True)


class ComplaintBox(models.Model):
    type = models.CharField(max_length=64)
    raised_to = models.UUIDField() 
    raised_by = models.ForeignKey('User', on_delete=models.CASCADE)
    title = models.CharField(max_length=255)
    description = models.TextField(blank=True, null=True)
    attachment = models.ForeignKey('Attachment', null=True, blank=True, on_delete=models.SET_NULL)
    priority_score = models.IntegerField(null=True, blank=True)
    total_votes = models.IntegerField(default=0)
    voted_by = models.ManyToManyField('User', related_name='complaint_votes', blank=True)


class Attendance(models.Model):
    clock_in = JSONField(blank=True, null=True)
    clock_out = JSONField(blank=True, null=True)
    total_duration = models.DurationField(null=True, blank=True)
    total_break = models.DurationField(null=True, blank=True)
    date = models.DateField()
    user = models.ForeignKey('User', on_delete=models.CASCADE)
    course = models.ForeignKey('Course', null=True, blank=True, on_delete=models.SET_NULL)
    institute = models.ForeignKey('Institution', null=True, blank=True, on_delete=models.SET_NULL)
    schedule = models.ForeignKey('Schedule', null=True, blank=True, on_delete=models.SET_NULL)
    late_arrival_timer = models.IntegerField(null=True, blank=True)
    on_schedule = models.BooleanField(default=False)


class NoticeBoard(models.Model):
    title = models.CharField(max_length=64, blank=True)
    body = models.TextField()
    notice_to_type = models.CharField(max_length=64, choices=NoticeTargetType.choices)
    notice_to_uid = models.UUIDField()
    attachments = models.ManyToManyField('Attachment', blank=True)


### Social/Post
class Post(models.Model):
    uid = models.UUIDField(primary_key=True, editable=False)
    body = models.TextField()
    slugs = models.CharField(max_length=255, blank=True, null=True)
    images = JSONField(blank=True, null=True)
    owner_type = models.CharField(max_length=64)
    owner_uid = models.UUIDField()
    owner_avatar = models.URLField(blank=True, null=True)
    owner_display_name = models.CharField(max_length=255, blank=True, null=True)
    comments = models.ManyToManyField('Reply', blank=True)
    viewer_count = models.IntegerField(default=0)
    allow_comments = models.BooleanField(default=True)
    total_comments = models.IntegerField(default=0)
    total_reactions = models.IntegerField(default=0)
    countdown = models.DateTimeField(null=True, blank=True)
    share_count = models.IntegerField(default=0)
    tags = JSONField(blank=True, null=True)
    type = models.CharField(max_length=64, choices=PostType.choices, blank=True, null=True)
    location = JSONField(blank=True, null=True)


class Reply(models.Model):
    uid = models.UUIDField(primary_key=True, editable=False)
    post = models.ForeignKey('Post', on_delete=models.CASCADE)
    parent_reply = models.ForeignKey('self', null=True, blank=True, on_delete=models.CASCADE)
    type = models.CharField(max_length=64, choices=ReplyType.choices, blank=True, null=True)
    body = models.TextField(blank=True, null=True)
    tags = JSONField(blank=True, null=True)


### Chat / Message (minimal, as files lacked full detail)
class Message(models.Model):
    uid = models.UUIDField(primary_key=True, editable=False)
    sender = models.ForeignKey('User', on_delete=models.CASCADE)
    content = models.TextField()
    sent_at = models.DateTimeField(auto_now_add=True)
    channel = models.CharField(max_length=255)
    metadata = JSONField(blank=True, null=True)


```

## 6) To do
- yearly/semeter/manual session control for courses
- optimized feed system
- advertisement service
- payment service
---
