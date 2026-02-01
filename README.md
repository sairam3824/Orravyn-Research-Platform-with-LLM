## Research Platform — Detailed Architecture and Flows

View at [orravyn.cloud](https://orravyn.cloud) or at [orravyn.saiii.in](https://orravyn.saiii.in)

A Django-based research collaboration platform: accounts, papers, groups, search, real‑time chat, REST APIs with JWT, background processing, and a simple ML layer for recommendations and summaries.

### High-level Architecture
- Project config: `research_platform/research_platform/` (`settings.py`, `urls.py`, `asgi.py`, `wsgi.py`)
- Apps: `apps/accounts`, `apps/papers`, `apps/groups`, `apps/chat`, `apps/search`, `apps/ml_engine`, `apps/api`
- Templates: `templates/` (server-rendered HTML)
- Media storage: `media/` (avatars, PDFs)
- Optional infra: Redis/Celery/Channels (production), in‑memory Channels layer (dev)

### Environment & Run
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
```
- JWT auth available under `/api/auth/token/` and `/api/auth/token/refresh/`.
- WebSocket (dev): in‑memory layer via `ASGI_APPLICATION`.

---

## Request Lifecycles and Module Flows

### 1) Settings, URLs, ASGI/WSGI
- `settings.py`
  - Registers third‑party: DRF, SimpleJWT, CORS, Channels, django‑filters.
  - Auth: `AUTH_USER_MODEL='accounts.User'` with JWT+Session auth and `IsAuthenticated` default.
  - Channels: in‑memory layer for dev; switch to `channels_redis` in production.
  - DB: SQLite default; MySQL scaffold commented.
  - Media/Static: local paths under project base.
  - Logging: file `logs/django.log` + console.
- `urls.py`
  - Home view renders `templates/home.html` with `recent_papers` and `popular_categories`.
  - Mounts: `/api/`, `/accounts/`, `/papers/`, `/groups/`, `/chat/`, `/search/`.
- `asgi.py`
  - `ProtocolTypeRouter` for HTTP + WebSocket.
  - WebSocket routes imported from `apps.chat.routing.websocket_urlpatterns`.
- `wsgi.py`
  - Standard WSGI app for sync servers.

### 2) Accounts
Models: `apps/accounts/models.py`
- `User(AbstractUser)`
  - Email is unique and used as `USERNAME_FIELD` (login with email).
  - `user_type`: `admin|moderator|publisher|reader` controls permissions.
- `UserProfile`
  - One‑to‑one with `User`, stores profile fields and avatar.
- `SearchHistory`
  - Per‑user saved queries with timestamp ordering.

Forms/Serializers: `forms.py`, `serializers.py`
- `LoginForm(email, password)` — used by `LoginView` for email‑based auth.
- `UserRegistrationForm` limits `user_type` choices to `publisher|reader` at registration.
- `UserRegistrationSerializer` validates and creates both `User` and `UserProfile` for API sign‑ups.

Views: `views.py`
- `LoginView(FormView)`
  - On valid form: `authenticate(username=email, password=...)` → `login()`; success redirect to dashboard.
- `RegisterView(FormView)`
  - On valid form: `form.save()` → create `User`; then `UserProfile` with provided fields; success redirect to login.
- `LogoutView` — logs out and redirects to `home`.
- `ProfileView(LoginRequired)`
  - Ensures `UserProfile` exists; aggregates counts: uploaded papers, bookmarks, ratings, groups.
- `ProfileEditView(LoginRequired, UpdateView)`
  - Edits `UserProfile`; only current user's profile is editable.
- `DashboardView(LoginRequired)`
  - Shows recent activity (uploads/bookmarks/ratings) and optional `UserRecommendation` items.
- `AdminDashboardView(LoginRequired)`
  - Guards `user_type=='admin'`; lists pending papers and site stats.

Flow summary
1) User registers → `User` + `UserProfile` created.
2) User logs in with email → session established.
3) Dashboard aggregates activity; admin dashboard shows moderation stats.

### 3) Papers
Models: `apps/papers/models.py`
- `Category`
- `Paper`
  - Tracks metadata, uploader, categories, `is_approved`, `view_count`, `download_count`, optional `summary`.
  - `average_rating` property aggregates related `Rating` rows.
  - `citation_count` property counts incoming `Citation` relations.
- `PaperCategory` through table, unique `(paper, category)`.
- `Bookmark`, `Rating` with unique `(user, paper)` constraints.
- `ReadingProgress`, `PaperView` for per‑user progress and de‑duped views.

Forms: `apps/papers/forms.py`
- `PaperUploadForm`/`PaperEditForm` handle core fields and required categories.
- `RatingForm` for rating (1–5) and optional review text.

Views: `apps/papers/views.py`
- `PaperListView`
  - Filters approved papers; supports search, category filter, and sorting by recency, popularity, rating, citations.
- `PaperDetailView`
  - Queryset filtered by user role (moderator/admin see all; publisher sees own + approved; anonymous sees approved).
  - On first view per authenticated user: creates `PaperView`, increments `view_count` atomically.
  - Injects `ratings`, `citations` (incoming and outgoing), current user's `Bookmark` and `Rating` if logged in.
- `PaperUploadView(LoginRequired, CreateView)`
  - Guards `user_type` ∈ {publisher, moderator, admin}.
  - Sets `uploaded_by`; auto‑approves if uploader is moderator/admin; saves M2M categories after instance save.
- `PaperEditView(LoginRequired, UpdateView)`
  - Publishers edit their own; moderators/admins can edit any.
- `PaperDeleteView(LoginRequired, DeleteView)`
  - Publishers delete their own; moderators/admins can delete any.
- `MyPapersView(LoginRequired)` — lists uploads by current user.
- `BookmarkListView(LoginRequired)` — lists user's bookmarks.
- `CategoryListView`, `CategoryDetailView` — browse categories and approved papers within.
- `PendingApprovalView(LoginRequired)` — moderators/admins view unapproved submissions.
- `bookmark_paper(LoginRequired)` — toggles bookmark; redirects back to detail.
- `rate_paper(LoginRequired)` — creates/updates rating from `RatingForm`; redirects back to detail.
- `download_paper(LoginRequired)`
  - Ensures `is_approved`; increments `download_count`; streams PDF if present, else 404/error message.
- `approve_paper` / `reject_paper` — moderators/admins approve or delete submissions.
- `AdminPaperListView(LoginRequired, UserPassesTestMixin)` — admin paper management with search.
- `PaperSummaryView(LoginRequired, DetailView)` — view paper plus `summary` text if available.

Signals: `apps/papers/signals.py`
- `post_save(Paper)` → if created and `pdf_path` exists, submit `process_summary(paper_id, pdf_file)` to thread executor.
  - `process_summary` calls `ml_models.bart_summarizer_lambda.summarize_text_from_pdf(pdf)` and updates `Paper.summary`.
- Executor: `apps/papers/background.py` (`ThreadPoolExecutor(max_workers=4)`).

Simple API stubs inside `papers/views.py` (DRF generics)
- `PaperListCreateView` — lists approved papers; creation returns 501 (not implemented).
- `BookmarkListCreateView`, `RatingListCreateView` — list for current user; create returns 501.

Flow summary
1) Upload → set uploader and approval based on role → signal kicks off async summary if PDF provided.
2) Detail view increments unique views per user and exposes related ratings/citations and user state.
3) Users can bookmark/rate and download approved PDFs; moderators/admins moderate.

### 4) Groups
Models: `apps/groups/models.py`
- `Group` (creator, privacy flag), `GroupMember` (role: admin/moderator/member, unique per group+user), `GroupPaper` (papers attached to group).

Views: `apps/groups/views.py`
- `GroupListView` — lists public groups.
- `GroupDetailView` — shows members and papers; flags whether current user is a member; lists user's approved uploads for add‑to‑group.
- `GroupCreateView` — creator becomes `GroupMember` with role `admin`.
- `GroupEditView` — creator can edit their group.
- `MyGroupsView` — lists user's memberships.
- `join_group` — blocks private groups; otherwise creates membership.
- `leave_group` — removes membership if present.
- `add_paper_to_group` — only members can add approved papers.
- Member management:
  - `invite_member` — only group `admin|moderator`; add user by username/email with role.
  - `remove_member` — only `admin|moderator`; blocks removing creator unless self (admin creator).
  - `update_member_role` — only group `admin` may change roles.

Flow summary
1) Create group → creator added as admin.
2) Members join/leave; admins/moderators manage members and attach papers.

### 5) Chat (WebSockets)
Models: `apps/chat/models.py`
- `ChatRoom` — scoped to a `Paper` or `Group`, created by a user.
- `ChatMessage` — messages per room; can be user or bot; ordered by timestamp.

Routing: `apps/chat/routing.py`
- `ws/chat/<room_id>/` routed to consumer.

Consumer: `apps/chat/consumers.py (AsyncWebsocketConsumer)`
- `connect` — joins channel layer group `chat_<room_id>` and accepts.
- `disconnect` — leaves group.
- `receive` — parses JSON; persists message via `save_message`; broadcasts to group; if message starts with `@bot`, generates bot response and broadcasts.
- `chat_message` — sends event JSON to client.
- `save_message` — sync DB call via `database_sync_to_async` to create `ChatMessage` for `ChatRoom`.
- `generate_bot_response` — simple echo‑style bot; can be replaced by `ml_engine` chatbot.

Flow summary
1) Client connects to `ws/chat/<room_id>/` → joins room group.
2) Messages are saved and broadcast; optional bot replies when prefixed with `@bot`.

### 6) Search
Views: `apps/search/views.py`
- `SearchView(ListView)` — filters approved `Paper` by `q` (title/abstract/authors), category, author, year range; saves `SearchHistory` for authenticated users; paginated and ordered by recency.
- `AdvancedSearchView` — renders advanced form with categories.
- `SearchHistoryView(LoginRequired)` — shows user's prior queries.
- `search_suggestions` — returns up to 10 matching paper titles as JSON.
- `PaperSearchView` — API form of `SearchView` logic.

Flow summary
1) Search query persists to history; results support multiple filters and are paginated.

### 7) REST API
Routing: `apps/api/urls.py`
- JWT: `auth/token/`, `auth/token/refresh/`.
- Auth: `auth/register/`, `auth/profile/`, `auth/login/` (when implemented in `apps/api/views.py`).
- Papers: `papers/`, `papers/<pk>/`, `papers/<pk>/approve/` (using views from `apps/papers/views.py`).
- Lists: `bookmarks/`, `ratings/`.
- Search: `search/`, `search/suggestions/`.
- Recommendations: `recommendations/`.

Notes
- Some API views are placeholders or thin wrappers over server views; defaults require authentication.
- Use header: `Authorization: Bearer <JWT>`.

### 8) ML Engine
Models: `apps/ml_engine/models.py`
- `UserRecommendation` — stores scored recommendations per user (unique per `user+paper`).
- `RecommendationModel` — registry of available models.
- `PaperEmbedding` — one‑to‑one per paper storing an embedding vector (JSON).

Recommendation engine: `apps/ml_engine/recommendation_engine.py`
- `generate_paper_embeddings(paper_id)` — simple numeric embedding; persists to `PaperEmbedding`.
- `collaborative_filtering(user_id)` — finds similar users by high ratings and recommends their high‑rated papers.
- `content_based_filtering(user_id)` — recommends approved papers matching categories of user's liked/bookmarked items; falls back to popular.
- `hybrid_recommendations(user_id)` — weighted combination (0.6 CF, 0.4 CB).
- `save_recommendations(user_id, recommendations)` — persists to `UserRecommendation`.

Tasks layer: `apps/ml_engine/tasks.py`
- `process_paper_upload(paper_id)` — generates embeddings and logs result.
- `generate_recommendations(user_id)` — computes hybrid recommendations and saves them.

Text processing: `apps/ml_engine/text_processing.py`
- `TextProcessor.extract_paper_text(paper_id)` — concatenates title+abstract, cleans punctuation/whitespace.
- `extract_keywords(text)` — simple keyword frequency extraction with stop‑word filtering.

Chatbot: `apps/ml_engine/chatbot.py`
- `ResearchChatBot.generate_response(message, paper)` — answers basic questions about paper metadata.

### 9) Background Execution
- `apps/papers/background.py` sets `ThreadPoolExecutor(max_workers=4)`.
- `apps/papers/signals.py` submits PDF summary generation to executor on new `Paper` with PDF.
- For production with heavier workloads, swap to Celery (`research_platform/celery.py`) + Redis broker.

### 10) Home Page Context
- `research_platform/urls.py:home_view` queries:
  - `recent_papers`: latest approved papers.
  - `popular_categories`: annotates categories by paper count and orders desc.

---

## Security & Deployment Notes
- Replace `SECRET_KEY` and tighten `ALLOWED_HOSTS` for production.
- Serve static/media via a web server or storage service; configure `CHANNEL_LAYERS` with Redis.
- Run with an ASGI server (Daphne/Uvicorn) for WebSockets; WSGI for legacy only.
- Consider Celery workers for summaries/embeddings and scheduled recommendation refresh.

## Known Placeholders
- Some DRF create endpoints return 501 (intentionally not implemented yet).
- API views in `apps/api/views.py` may be placeholders until fully implemented.


